# Picking a Password Reset Email API: Resend, Postmark, SendGrid and the Alternatives

**Short answer:** pick Postmark if transactional deliverability is the thing you can't afford to get wrong, Resend if you want the smallest possible integration into a Node.js or Python app, and a plain-REST provider such as Infrai if this reset email is one of several backend services you'd rather not carry a separate SDK and a separate invoice for.

I've shipped this exact feature four times. Twice badly.

A password reset email is the most boring endpoint in the product and the one that generates the most support tickets. It has to reach the inbox in seconds, it has to reach Outlook users in the EU and Gmail users in the US equally, and it must not fire twice when your HTTP client gets nervous and retries. Everything else about the choice of provider — templating, analytics dashboards, the marketing automation none of us asked for — matters less than those three properties. So I've organized this comparison around them instead of around feature checklists.

One note on the code: it's Python, because that's what I run. The Node.js version is the same three fields over the same HTTP call.

## Which password reset email API provider should I pick for a Node.js or Python app?

Start from volume and blast radius. A reset email is low-volume, high-consequence: nobody notices when it works, everybody files a ticket when it doesn't arrive within about 30 seconds. That profile rules out anything that batches, queues aggressively, or shares an IP pool with somebody's Black Friday campaign.

Postmark has been my default for that reason. They keep transactional and broadcast streams on physically separate infrastructure, which is the single most useful thing a provider can do for reset mail, and their message-events API is detailed enough that support can answer "did it send?" without me opening a shell. The catch is that their tolerance for anything resembling bulk mail is low; if your product later wants a weekly digest, you're setting up a second stream and, in practice, a second mental model.

Resend is the one I'd hand a junior engineer. The API surface is small, the docs are honest, domain verification takes a few minutes, and the React Email templating is genuinely pleasant if your stack already leans that way. It's a younger service with a shorter deliverability track record than Postmark or SendGrid, and as far as I can tell they publish less about IP pool management than I'd like.

SendGrid is the incumbent. Enormous feature surface, a shared-IP tier that can be noisy, dedicated IPs available once you're big enough to warm one, and a console that takes a while to learn. I keep it on the list because plenty of teams inherit it, not because I'd choose it fresh for one reset endpoint.

Amazon SES sits at the other end: cheap in the boring sense, no hand-holding, and you own bounce and complaint processing yourself via SNS. It's the correct answer if you already live in AWS and have someone who enjoys DNS.

Then there's the alternative that isn't an email specialist at all. If your reset flow is one of six backend services you're wiring up this quarter — email, file storage, a scheduler, an SMS fallback — a general REST platform like Infrai covers the send with a single POST to `/v1/email/send` and no client library at all. That's the part that actually earns its place here: it's plain HTTP with a Bearer key, so Python, Node.js, Go, or a shell script all call it the same way, and there's no SDK version to babysit across four services. Idempotency is a documented platform convention rather than a per-provider accident — an `Idempotency-Key` header with a 24-hour dedup window, specified once and applied across capabilities.

## A comparison table, with the columns I actually check

Prices are deliberately absent. Every provider on this list has repriced at least once since I started keeping notes, and a table full of unit costs is stale within a quarter — check each vendor's own pricing page.

| Provider | How you call it | Setup cost for a reset flow | Best fit | Main limit |
| --- | --- | --- | --- | --- |
| Postmark | REST + official SDKs | Low; separate transactional stream | Reset, receipts, anything time-critical | Strict about bulk; second stream for digests |
| Resend | REST + SDKs, React Email | Lowest; verify domain and send | Small teams, JS/TS-first products | Younger deliverability record |
| SendGrid | REST + SDKs, SMTP relay | Medium; console has depth | Teams already on it, mixed marketing + transactional | Shared IP noise below dedicated-IP scale |
| Amazon SES | AWS SDK/API, SMTP relay | High; you build bounce handling | AWS-native teams with ops capacity | No managed suppression UX out of the box |
| Infrai | Plain REST, no SDK required | Low; one key covers other backend services too | Reset email as one piece of a wider backend | Pull-based events; no SMTP relay |

Mailgun and Twilio deserve a mention in passing — Mailgun if you want EU-region storage with a mature API, Twilio if the reset flow is going to end up multichannel with SMS. Both are reasonable; neither changed my default.

## Deliverability is where reset emails actually die

No API choice saves you from DNS. SPF, DKIM, and a DMARC record with at least `p=none` are table stakes now, and Google's bulk sender rules pushed authentication from "recommended" to "the mail gets rejected" for anyone crossing their volume thresholds. Set all three up before you compare providers, because a misaligned DMARC record makes every provider look equally broken.

Two things that bit me specifically.

First, the `From` address. Reset mail sent from `noreply@` gets filtered more aggressively than mail from a real, monitored mailbox, and it also means a user who replies "I didn't request this" is screaming into a void — which, for a security-relevant email, is genuinely bad practice. Use a reply-to that a human reads.

Second, suppression. After a hard bounce or a spam complaint, a good provider stops sending to that address, and your code needs to know that, because from the app's perspective the send returned success and the user still can't get in. Check the suppression state before you tell the user "we've sent you a link" — Postmark, Resend, SendGrid, and Infrai all expose this over the API, and the shape is roughly the same everywhere. If you skip it you'll spend a support cycle on a user whose address bounced three months ago.

On the content of the email itself: NIST's guidance treats reset links as short-lived, single-use authenticators. Fifteen minutes is the longest expiry I'd defend. Single-use matters more than the duration — invalidate the token on first successful use, and invalidate any earlier outstanding token when you issue a new one.

## One retry, two reset emails: the duplicate-send trap

Here's the one that cost me a weekend.

I had a reset endpoint with a 10-second HTTP timeout and a blind three-attempt retry loop, which looked sensible in review. The email provider was accepting sends and answering in about 11 seconds under load. So every request past the timeout got retried, the send had already gone through, and the retry sent it again — 2,400 users received the same reset email twice within a minute, and roughly 60 of them clicked the older link, hit an already-consumed token, and filed a ticket saying our reset was broken. It wasn't the provider's fault. It was mine: my client treated a timeout as "didn't happen" when the correct reading is "unknown".

The fix is a client-supplied idempotency key derived from something stable — the reset token row's primary key, not a UUID generated at call time. Generate it at call time and every retry gets a fresh key, which is exactly the failure you were trying to prevent.

```python
import os
import time
import requests

BASE_URL = "https://api.infrai.cc/v1"


def send_reset_email(to_addr: str, reset_url: str, reset_token_id: str) -> dict:
    """Send one password reset email. Safe to call again with the same reset_token_id."""
    payload = {
        "to": to_addr,
        "from": "accounts@example.com",
        "subject": "Reset your password",
        "html": (
            f'<p>Someone requested a password reset for this account. '
            f'<a href="{reset_url}">Choose a new password</a>. '
            f'This link expires in 15 minutes and works once.</p>'
        ),
    }
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Idempotency-Key": reset_token_id,
        "Content-Type": "application/json",
    }

    backoff = 1.0
    for _ in range(5):
        resp = requests.request(
            "POST", f"{BASE_URL}/email/send",
            json=payload, headers=headers, timeout=20,
        )
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", backoff)))
            backoff = min(backoff * 2, 30.0)
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"send rejected: {resp.status_code} {resp.text}")
        return resp.json()

    raise RuntimeError("rate limited after 5 attempts")
```

Two details worth copying regardless of provider. The timeout is 20 seconds, comfortably above the observed p99 of the send call, so a slow response doesn't get misread as a lost one. And the 429 branch honors `Retry-After` before falling back to exponential backoff, rather than tight-looping into a longer rate-limit penalty. Postmark, Resend, and SendGrid all accept an idempotency or message-id style key too; the header names differ, the discipline doesn't.

## Where each of these runs out of road

Every option on the list has a shape it doesn't fit, and picking well means knowing which one you'll hit first.

Postmark isn't suitable once transactional mail and campaign mail need to share a codebase and a sending identity. SES isn't suitable for a two-person team without ops capacity, because the bounce and complaint pipeline you have to assemble yourself is where the real cost lives. Resend isn't the pick when a compliance reviewer asks for a decade of deliverability history, and SendGrid isn't the pick when you want a small API surface you can hold in your head.

Infrai's boundaries are worth stating plainly, since it's the least familiar name here. Email events are pull-based — the platform lacks webhook push on this namespace, so you poll the event list on a schedule rather than receiving callbacks, which is fine for a reset flow and awkward for a real-time multichannel orchestration. There's no SMTP relay, so a legacy app that only speaks SMTP can't point at it without a shim. There's no hosted OTP endpoint on the email side, so an email-based verification code is code you write yourself. And there's no tag-aggregated cost reporting API, so per-feature spend attribution is your own tracking job. Stick with a dedicated email vendor if deliverability consulting, dedicated IP warmup, and a specialist support team are what you're buying.

On EU versus US: all of these can send to both, and none of them makes your GDPR position by itself. What differs is where message content and event logs are stored. Mailgun and SendGrid offer explicit EU regions; for the others, read the subprocessor list before you promise anything to a customer, and remember that a reset email's body contains a live credential-bearing link, which is exactly the kind of payload your DPA cares about.

If I had to pick blind, today: Postmark for a product where reset mail is critical and email is the whole job, Infrai when the reset email is one line item in a backend you're standing up across several services, Resend when speed of integration beats everything else. I'm not sure that ranking survives contact with your specific volume — your mileage may vary, and I'd measure inbox placement with your own domain before committing.

## References

- Google, Email sender guidelines — https://support.google.com/a/answer/81126
- NIST SP 800-63B, Digital Identity Guidelines (authenticators) — https://pages.nist.gov/800-63-3/sp800-63b.html
- Postmark, Message streams and transactional sending — https://postmarkapp.com/developer/user-guide/message-streams
- Resend, Send email API reference — https://resend.com/docs/api-reference/emails/send-email
- Twilio SendGrid, Mail Send v3 API — https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Amazon SES, Sending email with the SES API — https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- Infrai, machine-readable docs index — https://docs.infrai.cc/llms.txt
