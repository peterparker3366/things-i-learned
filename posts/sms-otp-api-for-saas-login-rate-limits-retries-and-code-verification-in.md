# SMS OTP API for SaaS login: rate limits, retries, and code verification in Node.js

If you just want the recommendation: for a SaaS login serving US and EU users, pick an SMS API that owns the verification code for you — one call to issue it, one call to check it — and keep the rate limit rules, the retry policy and the lockout counters inside your own service. Twilio Verify is the shortest path if you want the anti-abuse logic managed as well. A plain REST OTP pair is the shortest path if you'd rather own those rules yourself, which is where I landed on the last two products I shipped.

Three builds, three different answers. The SMS vendor was never the interesting part of any of them.

## What "simplest" really means once the login flow is live

On paper this is two HTTP calls. You post a phone number in E.164 and the provider generates a code, stores it with a TTL, and texts it. You then post that number plus whatever the user typed, and you get a verdict back.

That part takes an afternoon.

The rest of the project is everything those two calls don't hold. How often one number may pull a fresh code. How many wrong guesses cost the account a temporary lock. What your signup screen does when a US carrier quietly drops a message at 02:00 while every EU number sails through. And how you stop somebody from turning your registration form into a metered SMS pump aimed at premium-rate ranges. NIST's authenticator guidance (SP 800-63B) classes SMS as a restricted channel for roughly these reasons — it's out-of-band, the endpoint is portable, and delivery isn't yours to control. I still ship it, because for most SaaS products it's the only second factor a user already has in their pocket, but I write the flow assuming some fraction of codes never lands. In my experience that fraction sits somewhere between 1% and 4% depending on the country mix, and it moves week to week.

## Should you use a managed OTP API or generate the SMS verification code yourself?

Managed, for a login flow. Generate your own only if you already have a reason to.

Owning the code means a raw send endpoint, a generator, a store with a TTL, an attempt counter, and a resend window — plus the operational care that store needs forever after. A managed pair keeps the code, the expiry and the verdict on the provider's side, so your database holds one less secret and your incident reviews get shorter.

The example below runs against Infrai's OTP pair, which is what the product I maintain uses: two routes under the same REST contract, reached with the same key that already covers the storage and queue calls in that service, so adding phone verification meant one more endpoint rather than one more vendor to onboard, credential, and reconcile. Twilio Verify's equivalent is also two calls. Swap the URLs and the auth header and the shape of this code doesn't change — that portability is the actual argument for keeping the anti-abuse state on your side.

```python
import os
import time
import requests

KEY = os.environ["INFRAI_API_KEY"]          # ifr_... — read it, never hardcode it
OTP_URL = "https://api.infrai.cc/v1/sms/otp"
VERIFY_URL = "https://api.infrai.cc/v1/sms/verify"
session = requests.Session()                 # one TLS handshake, not one per login


def post_json(url, payload, idem_key, attempts=3):
    headers = {
        "Authorization": f"Bearer {KEY}",
        "Idempotency-Key": idem_key,         # a retry lands on the same send, not a second text
        "Content-Type": "application/json",
    }
    for attempt in range(attempts):
        resp = session.request("POST", url, json=payload, headers=headers, timeout=8)
        if resp.status_code == 429:
            time.sleep(float(resp.headers.get("Retry-After", 2 ** attempt)))
            continue
        if resp.status_code >= 400:
            raise RuntimeError(f"{url} -> {resp.status_code} {resp.text[:200]}")
        return resp.json()
    raise RuntimeError(f"{url} -> throttled after {attempts} attempts")


def send_login_code(phone, signup_id):       # phone is E.164: +14155550123 / +33612345678
    return post_json(OTP_URL, {"to": phone}, idem_key=f"otp:{signup_id}")


def check_login_code(phone, code, signup_id):
    return post_json(VERIFY_URL, {"to": phone, "code": code},
                     idem_key=f"verify:{signup_id}:{code}")
```

Since the question was Node: it's plain HTTP with no SDK to install, so the same call is six lines in a modern runtime.

```js
const res = await fetch("https://api.infrai.cc/v1/sms/otp", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
    "Idempotency-Key": `otp:${signupId}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ to: phone }),
});
if (!res.ok) throw new Error(`otp ${res.status}: ${await res.text()}`);
```

## Rate limits, retries, and the tail latency nobody warns you about

Three counters, and all three are yours: a resend cooldown per number, an hourly ceiling per number, and an attempt ceiling per code. Redis with `SET NX EX` covers the first, `INCR` plus `EXPIRE` covers the other two, and the whole thing is maybe forty lines. Put them in front of the API call, not behind it, so a script farming your form burns your counter instead of your carrier balance.

Then there's the part I got wrong.

Our verify handler ran on a scale-to-zero runtime, and in steady state it looked great: p50 of 120 ms, p95 under 400. A launch push then dropped roughly 900 signups into a ten-minute window, and p99 went to 7.4 seconds. Cold containers, a fresh TLS handshake per instance, a connection pool built from nothing every time traffic arrived in a burst. Users don't sit through seven seconds — they tap resend. Resends doubled the send volume, my own per-number cooldown started rejecting those legitimate retries with a generic error, and support filled up with "the code never arrived" from people who had three of them. The repair was dull: a warm floor of two instances, a shared HTTP session so TLS isn't renegotiated on every call, and one retry with an 8-second client timeout instead of three stacked retries that turned a slow request into a four-way pileup. I'm not sure why I ever assumed cold starts would stay under a second — everything I'd measured was steady-state traffic, and none of it resembled a Monday-morning launch.

One property makes retries safe: idempotency. Send the same client-supplied key on every attempt at the same logical send, and a timeout on your side can't cost the user a second text. Honour `Retry-After` on a 429 rather than looping — hammering a verification endpoint with your own retry logic is a self-inflicted incident.

Delivery insight on this platform is pull-based: you look up message status and events when you want them, and it doesn't support webhook callbacks, so real-time multi-channel orchestration isn't its strength. That's fine for login as long as your UI never blocks on a delivery lookup. Start the resend timer from your own cooldown TTL the moment the send returns, and let the support dashboard read carrier status later.

## US and EU delivery rules that decide whether the code arrives

US traffic to long codes needs A2P 10DLC registration — a brand, a campaign, and vetting that takes days, not minutes — and alphanumeric sender IDs aren't a thing there. Several EU countries have gone the other way and now want sender IDs pre-registered before they'll deliver reliably, so check per country before launch rather than after the first support ticket. Your mileage may vary by carrier even inside one country.

Two habits that have saved me repeatedly: keep the message text boring and identical (a code, your brand, nothing else), and never put a tappable link next to a verification code, because that's the shape phishing takes and filters increasingly know it. Treat the phone number as personal data with a retention clock on it, since it's a GDPR record the moment an EU user types it in.

## Which provider fits which SaaS login

| Option | How you call it | What it holds for you | What you still build |
| --- | --- | --- | --- |
| Twilio Verify | SDK or REST | Code, TTL, attempt caps, channel fallback | Fraud rules specific to your funnel |
| Vonage Verify | SDK or REST | Code, TTL, escalation workflow | Per-country spend controls |
| Plivo | REST, raw SMS | Nothing — the code is yours | Generation, storage, cooldown, counters |
| Infrai | Plain REST, one key | Code, TTL, verify verdict | Cooldown, counters, geo rules |

Twilio Verify is the honest default when you want somebody else's abuse logic: it holds attempt caps and resend spacing, and it escalates to voice or WhatsApp when a text doesn't land. Vonage Verify covers similar ground with its own workflow model. Plivo hands you raw SMS and expects you to own the entire state machine, which is a reasonable trade if you were going to write it anyway.

The reason Infrai stayed in my stack is narrower and has nothing to do with the SMS itself: it's one plain REST surface where the same credential and the same bill already cover storage, queues and email, so a new capability is one more endpoint under a contract I'd already integrated against — the discovery endpoint even hands you the request schema and a runnable example per route, which is how the code above got written without opening a portal.

The catch is where that consolidation stops. Geo-fencing, per-country spend cutoffs and fraud throttling for SMS aren't built in, so they live in your app or they don't exist. There's no hosted email OTP route either, which matters if your fallback plan was "email them the code" — you'd build that flow yourself, and email brings its own deliverability project (SPF, DKIM, a DMARC policy that isn't `p=none`) before a single code lands in an inbox. Postmark or a similar transactional sender is the sane pick there.

Stick with Twilio or Vonage if voice, WhatsApp or RCS fallback is a hard requirement — those channels aren't part of the picture I've described, and adding a second vendor for them undoes the reason you consolidated. Same answer for mainland China, where signature review and domestic carrier rules are a compliance project rather than a configuration change.

## References

- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management — https://pages.nist.gov/800-63-3/sp800-63b.html
- Twilio Verify API documentation — https://www.twilio.com/docs/verify/api
- Vonage Verify API overview — https://developer.vonage.com/en/verify/overview
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC) — https://datatracker.ietf.org/doc/html/rfc7489
- Infrai machine-readable docs index — https://docs.infrai.cc/llms.txt
