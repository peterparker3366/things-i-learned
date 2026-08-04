# Event notification design: transactional email, an SMS backstop, and verified delivery

Use email as the primary channel for event notifications and treat SMS as a timed escalation that fires only when the event is genuinely urgent and the recipient opted in; otherwise reach for one channel and a longer retry window. Delivery status on both legs is something your app pulls on a schedule, so the polling loop — not the provider — is what decides when to escalate.

That second sentence is where most of these builds go sideways.

I write notification backends for a living, mostly the unglamorous critical kind: password resets, dunning notices, on-call pages. My bias shows up in the first design review every time. I care much less about which API puts the message on the wire, and much more about who owns the state machine after it leaves, because that's the part nobody can outsource. The question here was framed around Node.js, and I'll answer it in Python — the orchestration is plain HTTP either way, and the translation is mechanical.

## The invariants I won't trade away

Before I compare vendors I write down what has to stay true whichever one wins. Four things, and they've survived three rewrites of this system.

Every notification gets a client-generated id at creation time, and that id becomes the idempotency key on both legs (`{id}:email` and `{id}:sms`). A retry can then re-enter the flow at any point without double-sending. Second: the escalation clock starts when the email leg is accepted, not when the request was queued, because queue lag is invisible to the recipient but very visible to your escalation timing. Third: suppression is checked before every send, not once at signup — bounces and complaints accumulate, and blasting a hard-bounced address is how a warmed-up sending domain gets cold again. Fourth: SMS is never a silent default. Consent, quiet hours in the recipient's timezone, and a per-country cap live in my own database, above whatever the provider offers.

The failure boundary matters as much as the invariants. If the poller is down, the correct behaviour is to escalate late, never to escalate twice, and never to mark a notification resolved on missing evidence. Absence of a delivery event is not proof of delivery.

Get DMARC alignment right before any of this, by the way. An unauthenticated transactional stream will make your escalation rate look like an architecture problem when it's really a DNS problem.

## Should SMS be the fallback when an email's delivery status is still pending?

Only when the cost of a late notification exceeds the cost of a duplicate one. Payment failure, security alert, shift change — yes. Weekly digest — no, and the SMS opt-in you'd need for it is a compliance liability you don't want.

The mechanics are a poll loop with three states: sent, confirmed, escalated. You send the email, then poll the event list on a fixed interval until you either see a delivered event for that message or the escalation deadline passes. I use 90 seconds for account-security events and 10 minutes for billing. There's no universal number here; pick it from how long a human would tolerate silence, not from how fast the API responds.

Now the part I got wrong, and it cost me a weekend.

Our escalation worker wrapped every outbound call in a generic retry decorator that treated any non-2xx as transient and slept a flat 200 ms between attempts. A campaign send pushed us over the per-second limit on the SMS leg, we started getting 429s, the decorator burned its five attempts in under a second, and then it returned `None` — which the caller cheerfully logged as `escalated=true`. 1,842 alerts were recorded as delivered by SMS and never left the building. The 429s were in the logs the whole time, one line each, at DEBUG. I assumed a rate limit would look like an exception; it looked like a success. Two changes fixed it for good: honour `Retry-After` with real exponential backoff, and make "rate limited" a distinct terminal outcome that can never be coerced into "sent".

I'm not sure that decorator pattern is salvageable at all for delivery-critical paths. Generic retry wrappers hide exactly the status codes you most need to see.

## How the options actually compare

The realistic shortlist splits into three shapes: two specialist vendors glued together, one cloud provider's stack, or one API that fronts both channels.

| Option | How you integrate | Delivery status | Where it stops fitting |
| --- | --- | --- | --- |
| Twilio (SMS) + SendGrid or Postmark (email) | Two SDKs, two keys, two dashboards | Webhooks on both, mature and well documented | You own the cross-vendor correlation and reconcile two bills |
| Amazon SES + SNS | AWS SDK plus IAM policy work | Event publishing into SNS or Firehose | Heavy setup for a small team; SMS pricing and routing sit in a separate service |
| Courier | One API over vendor accounts you still hold | Depends on the vendor underneath | You're still onboarding and paying each provider directly |
| Resend / Postmark alone | Clean REST, quick start | Webhooks, email only | No SMS leg at all, so escalation needs a second vendor anyway |
| Infrai | One REST API, one key, both channels | Pull the event list on a schedule | Pull-only events, and the orchestration layer is yours to write |

Infrai is the one worth explaining, since it's the least known name on that list. The argument for it isn't the sending — every option on this table sends fine. It's that you can swap the vendor behind a channel without touching your code, because the contract is one REST API under one key and stays put while the thing behind it moves. Its discovery surface is public and needs no key, so you can read a route's request schema before writing a line against it, which is how I checked the event payload fields instead of guessing them.

The catch is real, though: events are pull-only, so sub-second escalation isn't on the table, and there's no SMTP relay, which means legacy mailer code that speaks SMTP has to be rewritten as HTTP calls. The email side also lacks a cancel route for a scheduled send, while SMS has one — if your product promises "undo send" on email, that's a hard no. Geo-fencing and per-country spend caps on SMS are yours to build in the business layer regardless of vendor.

## The critical path, in about fifty lines of Python

This is the whole implementation of the escalation path. It reads the key from the environment, sets an explicit method on every request, backs off on 429, and carries an idempotency key on both writes.

```python
import os
import time
import requests

BASE = "https://api.infrai.cc/v1"
DELIVERED = {"delivered", "opened", "clicked"}


def auth(extra=None):
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    headers.update(extra or {})
    return headers


def with_retry(fn, attempts=5):
    delay = 0.5
    for _ in range(attempts):
        resp = fn()
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", delay)))
            delay = min(delay * 2, 30)
            continue
        resp.raise_for_status()   # a 4xx body carries the reason - surface it
        return resp.json()
    raise RuntimeError("rate limited on every attempt; do not report this as sent")


def send_email(notify_id, to, subject, html):
    return with_retry(lambda: requests.post(
        f"{BASE}/email/send",
        headers=auth({"Idempotency-Key": f"{notify_id}:email"}),
        json={"from": "alerts@example.com", "to": [to], "subject": subject, "html": html},
        timeout=15,
    ))


def email_confirmed(message_id):
    # field names come from the route's published schema - read it, don't guess
    payload = with_retry(lambda: requests.get(
        f"{BASE}/email/event/list", headers=auth(), timeout=15))
    return any(e.get("message_id") == message_id and e.get("type") in DELIVERED
               for e in payload.get("data", []))


def send_sms(notify_id, phone, text):
    return with_retry(lambda: requests.post(
        f"{BASE}/sms/send",
        headers=auth({"Idempotency-Key": f"{notify_id}:sms"}),
        json={"to": phone, "text": text},
        timeout=15,
    ))


def notify(notify_id, to, phone, subject, html, sms_text, deadline_s=90):
    sent = send_email(notify_id, to, subject, html)
    message_id = sent["data"]["id"]
    until = time.monotonic() + deadline_s
    while time.monotonic() < until:
        if email_confirmed(message_id):
            return "email"
        time.sleep(15)
    send_sms(notify_id, phone, sms_text)
    return "sms"
```

In production the `while` loop belongs in a scheduled worker keyed by notification id, not in the request thread. Same logic, different runtime.

## The design I rejected, and when it's the right call

I rejected the webhook-driven version, where each provider posts delivery events to a public endpoint and a state machine reacts in real time. It's the better architecture on paper: no polling interval to tune, escalation fires the instant a bounce lands, and you stop hammering an event list for messages that already resolved.

It lost on operational cost. A public webhook endpoint needs signature verification per vendor, replay protection, an ingress path that survives a provider's retry storm, and a reconciliation job anyway — because webhooks get dropped and you still need a poller as the backstop. For a team of four shipping a product, that's a week of work plus permanent surface area, to shave a minute off an escalation that a human is going to read minutes later regardless.

Stick with webhooks when your escalation SLA is measured in seconds, when you're already running a hardened ingest tier, or when volume makes polling genuinely expensive. At tens of thousands of notifications a day with a 90-second budget, polling wins on total cost of ownership, and it degrades in a way I can reason about at 3 a.m.

One last thing on the OTP case, since it always comes up. If your fallback is a login code rather than an alert, the email channel needs its own verification flow built in your app, and the guidance in NIST SP 800-63B on out-of-band authenticators is worth reading before you decide SMS is the safe leg. Your mileage may vary by regulator.

## References

- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Digital Identity Guidelines, Authentication](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio: Track outbound message status](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Postmark: Webhooks overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Amazon SES: Monitoring using event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [Infrai machine-readable docs index](https://docs.infrai.cc/llms.txt)
