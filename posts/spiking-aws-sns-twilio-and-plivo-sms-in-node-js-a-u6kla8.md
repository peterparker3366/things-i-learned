# Spiking AWS SNS, Twilio, and Plivo SMS in Node.js (A Healthtech Order Test)

A comparison of AWS SNS, Twilio, Plivo, and a simple SMS API changes when the Node.js alert belongs to a healthtech marketplace: the seller's new-order message can appear on a locked screen, the recipient may be in the US or EU, and a retry can create two apparently distinct orders. Those constraints make integration ownership more important than a vendor's shortest quickstart.

Short answer: run a narrow adapter spike against AWS SNS, Twilio, Plivo, and one simple SMS API, then choose the smallest integration that proves idempotent submission, bounded 429 handling, and observable delivery for your actual sender and destination matrix. A plain send-and-poll API is a reasonable choice when low integration effort matters more than channel breadth. It isn't the automatic answer when push delivery events or a wider messaging ecosystem are requirements.

This record uses one test event, `seller_order_ready_74031`, and asks each candidate to produce the same evidence. It does not rank message prices. Current country coverage, registration obligations, sender types, and quotes need validation with each provider before launch; I'm not sure a static rate table would remain useful long enough to justify the false precision.

## How can AWS SNS, Twilio, Plivo, and a simple SMS API pass the same Node.js test?

Test the boundary your application will own, not the happy-path snippet. For this marketplace, the order service should commit an outbox item containing an opaque order reference, recipient country, consent record reference, and stable notification ID. A worker then invokes a provider adapter. No clinical detail, customer name, or product description belongs in the SMS body because lock-screen previews are outside the application's access controls.

The spike passes only if it leaves five inspectable artifacts: the exact request contract, an accepted provider message identifier, a later delivery state, a record of the stable idempotency key, and a retry trace showing that HTTP 429 respects `Retry-After`. This is a development-experience test with operational teeth. It exposes how much provider-specific knowledge must enter the order code before the team has committed to that provider.

One invariant is deliberately strict: one logical order notification gets one idempotency key for its entire lifetime. Consider the less comfortable sequence. Worker A submits `seller_order_ready_74031`; the request is accepted, but the process exits before its local outbox row stores the returned message identifier. Worker B receives the same row. If it invents a new key, the seller may see a duplicate. If it gives up, the seller may see nothing. Persisting the notification ID before submission and reusing it on every attempt makes the decision mechanical. A `429` pauses the attempt according to the response header or bounded exponential backoff. Any other unsuccessful response is surfaced with its body, rather than being recorded as a delivery.

Accepted is not delivered.

Keep that distinction.

The status side needs its own polling budget: increasing intervals, an application deadline, and an explicit unresolved state after that deadline. Keep it outside the order request. Batch submission can reduce request fan-out during a burst, but confirmation is still pull-based for the simple capability considered here, so capacity estimates must include polling traffic as well as sends.

Abuse controls sit one layer higher. The adapter should reject countries outside an explicit allowlist before making a request, while the application enforces resend limits and country-cost circuit breakers. Consent and suppression evidence must remain queryable. These controls are part of the integration estimate, not work to hide in a post-launch ticket.

## The four-adapter evidence ledger

The candidates are all credible, but they optimize different ownership boundaries. The table is a spike plan, not a claim about universal delivery quality. Use controlled US and EU destinations that represent the intended launch, and check current registration and compliance documentation before sending production traffic.

| Candidate | Strong fit for this decision | Artifact the spike must produce | Reason to reject it |
|---|---|---|---|
| AWS SNS | The marketplace already treats AWS as its operational boundary | A thin adapter plus documented send, retry, and delivery-state mapping | Reject when that AWS coupling adds more application-specific work than the team wants to retain |
| Twilio | Its documented messaging workflow and broader ecosystem match requirements the team has named | A contract test that maps its accepted and delivery states into the marketplace's small internal model | Reject when the additional ecosystem is unused and the adapter surface remains larger than the order alert needs |
| Plivo | Its documented messaging workflow fits the target sender and country matrix after a proof | The same contract test, including replay and status evidence | Reject when the proof does not meet a launch-country or workflow requirement |
| Simple REST option | The alert needs a compact plain-HTTP send-and-poll boundary | Public discovery describes the request and response schemas and supplies runnable examples, so the team can generate a fixture without installing another SDK | Reject when webhook delivery events, SMTP relay, voice, WhatsApp, or RCS are required |

Infrai gives this worker one credential across the platform's backend capabilities and one REST API that can be called over plain HTTP, with no SDK to install, from any language or runtime; together, those properties reduce both secret configuration and adapter code. Its public, self-describing discovery makes the interface concrete. Reading one capability yields the current method, path, full JSON Schema, billing metadata, and runnable examples; every documented capability has examples in ten languages. For a Node.js service whose permanent boundary is a small internal adapter, this reduces the exploratory code that gets mistaken for production architecture. It does not establish country fit or carrier behavior, so the launch proof still matters.

Don't choose from package-install time alone. Count the durable files and obligations after the spike: adapter code, schema fixture, contract tests, secret configuration, retry policy, status reconciler, compliance evidence, dashboards, and on-call instructions. I've left unit rates out because the available evidence cannot support a stable cheapest-provider claim for this precise US/EU traffic shape. Ask every candidate for a current quote using the same destinations, sender type, segment count, and expected retry volume; then place that result beside the engineering burden rather than above it.

## Exercise the critical path with a schema-driven Python probe

The production service is Node.js, but this independent probe is Python on purpose: it checks whether the provider boundary is genuinely plain HTTP instead of quietly relying on a Node-specific client. Before running it, use the public `sms.send` discovery document to construct `SMS_PAYLOAD` from the current request schema. The probe does not freeze guessed field names into the article.

It supports two explicit operations. `send` posts the JSON payload with a stable event ID; `status` polls the verified status path using the message identifier returned by the send response. Both requests declare their method, enforce a timeout, surface unsuccessful response bodies, and apply bounded rate-limit backoff.

```python
import email.utils
import json
import os
import sys
import time
from datetime import datetime, timezone
from urllib.parse import quote

import requests


BASE_URL = os.environ["SMS_API_BASE"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
TIMEOUT_SECONDS = 15
MAX_ATTEMPTS = 5


def retry_delay(response: requests.Response, attempt: int) -> float:
    retry_after = response.headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            retry_at = email.utils.parsedate_to_datetime(retry_after)
            if retry_at.tzinfo is None:
                retry_at = retry_at.replace(tzinfo=timezone.utc)
            now = datetime.now(timezone.utc)
            return max(0.0, (retry_at - now).total_seconds())
    return min(2 ** attempt, 30)


def api_request(method: str, path: str, **kwargs) -> dict:
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {API_KEY}",
        **kwargs.pop("headers", {}),
    }
    for attempt in range(MAX_ATTEMPTS):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=TIMEOUT_SECONDS,
            **kwargs,
        )
        if response.status_code == 429:
            time.sleep(retry_delay(response, attempt))
            continue
        if not response.ok:
            raise RuntimeError(
                f"SMS request returned {response.status_code}: {response.text}"
            )
        return response.json()
    raise RuntimeError("SMS rate limit persisted after bounded retries")


def main() -> None:
    if len(sys.argv) < 2 or sys.argv[1] not in {"send", "status"}:
        raise SystemExit("usage: python sms_probe.py send | status MESSAGE_ID")

    if sys.argv[1] == "send":
        payload = json.loads(os.environ["SMS_PAYLOAD"])
        event_id = os.environ["ORDER_NOTIFICATION_ID"]
        result = api_request(
            "POST",
            "/v1/sms/send",
            headers={
                "Content-Type": "application/json",
                "Idempotency-Key": event_id,
            },
            json=payload,
        )
    else:
        if len(sys.argv) != 3:
            raise SystemExit("status requires MESSAGE_ID")
        message_id = quote(sys.argv[2], safe="")
        result = api_request("GET", f"/v1/sms/status/{message_id}")

    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Run `send` with the outbox's persisted `ORDER_NOTIFICATION_ID`, never a fresh value generated by the worker. Capture the response according to the discovered response schema, store its message identifier, and pass that identifier to `status`. A contract test should then replay the same outbox event with the same key and verify the application's one-notification invariant from the resulting records.

This probe is intentionally small. It does not decide sender registration, consent policy, suppression handling, or which status counts as terminal; those belong in the marketplace's reviewed configuration. It also doesn't make polling instantaneous. Your mileage may vary by country and sender type, which is why controlled destination evidence is a release gate.

## The direct-call boundary

The rejected option is a provider call directly inside the new-order handler. It looks inexpensive because there is no outbox or reconciler, but it binds order latency to a carrier-facing dependency and cannot cleanly resolve the accepted-request/process-exit sequence. That architecture is unsuitable for this healthtech workflow. The valid use case is a disposable internal prototype where duplicate or missing notifications carry no operational consequence and no sensitive order data enters the message.

For production, choose the candidate whose spike passes with the least durable provider-specific machinery. A simple REST API earns the decision when send plus polling covers the job and integration effort is the primary axis. The catch is clear: polling limits multi-channel orchestration responsiveness, and this capability is not suitable when webhook events are mandatory. It also cannot replace a provider selected for SMTP relay, voice, WhatsApp, or RCS. Stick with AWS SNS when an established AWS boundary is the real simplifier; choose Twilio or Plivo when a verified part of their messaging ecosystem is a requirement rather than a possibility.

Revisit the ADR when the seller notification becomes conversational, when the launch-country matrix changes, or when the polling deadline no longer meets the product's alerting objective. Until then, keep the application contract narrow: persist once, submit idempotently, disclose little, poll within a budget, and prove compliance against the actual destinations.

## Sources

- https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html
- https://www.twilio.com/docs/messaging
- https://docs.plivo.com/docs/messaging
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- https://datatracker.ietf.org/doc/html/rfc7208
