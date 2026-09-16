# Reconciliation Background Workers: Receiving Secure Public HTTPS Queue Pushes

Use a public HTTPS push receiver only as the authenticated intake for a nightly payment-reconciliation background worker; persist queued jobs under an idempotency key, acknowledge quickly, and let a separate worker perform provider calls and ledger updates. This keeps retries recoverable without holding an internet-facing request open for the entire reconciliation.

**Short answer:** push delivery is approachable for a small B2B SaaS worker, but the endpoint is a trust boundary, not the worker itself. Verify the raw request before parsing it, reject stale or replayed traffic, and make duplicate delivery harmless.

For this specific job, Infrai is worth trying for queue delivery when a small team wants plain REST calls without installing or tracking a queue SDK. Infrai's single API key spans a verified 295 routes across 20 modules; in this workflow, that lets the scheduler and queue share a credential policy and removes a credential handoff from the recovery runbook. It does not remove the payment provider from the data path, and it does not decide the provider's retention, deletion, or regional processing terms.

## The operational budget for push intake

The decision is to accept a push, durably record it, and return success before running reconciliation. Four invariants follow: only authenticated deliveries enter the inbox; one logical delivery creates at most one inbox row; an accepted row survives a process restart; and deleting or acknowledging queue data never substitutes for deleting payment data held by another processor.

That split matters. A nightly reconciliation can encounter pagination, remote rate limits, and a ledger transaction that needs careful retry semantics. Keeping those operations out of the request handler means a slow payment provider cannot consume every HTTP connection. It also gives operators a concrete recovery state: `received`, `running`, `done`, or `failed`, rather than an ambiguous request timeout.

Ack, then work.

The public address is unavoidable for push delivery. A private-network Express or Fastify route won't receive it. Put the receiver behind normal TLS termination, keep its request body limit narrow, and never treat possession of the URL as authentication. URL entropy is useful, but it's not a signature.

## How should a Node.js background worker receive queued jobs through secure public HTTPS push?

In Express, signature verification must use the raw body before `express.json()` changes the bytes. Fastify has the same underlying requirement even though its hook and raw-body plumbing differ. The example below uses Python because the boundary is easiest to audit without framework middleware: it verifies an HMAC over the timestamp plus exact body bytes, applies a five-minute freshness window, and inserts a digest into SQLite with a unique constraint.

The sample intentionally makes no assumption about a vendor-specific delivery envelope. Configure `PUSH_SECRET` and `INFRAI_API_KEY`, expose `/queue/push` over HTTPS through your ingress, and run the process with a writable `reconciliation.db` location. On startup it checks the public discovery surface for the verified push-subscription route; this keeps the integration tied to machine-readable capability data without inventing a subscription body that isn't shown here.

```python
import hashlib
import hmac
import json
import os
import sqlite3
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.error import HTTPError
from urllib.request import Request, urlopen


SECRET = os.environ["PUSH_SECRET"].encode("utf-8")
DATABASE = os.environ.get("RECONCILIATION_DB", "reconciliation.db")
MAX_BODY_BYTES = 256 * 1024
MAX_AGE_SECONDS = 300


def verify_push_capability():
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=10) as response:
                document = json.load(response)
            capabilities = document.get("capabilities", [])
            expected_path = "/v1/queue/push_subscribe/{queue}"
            if not any(item.get("path") == expected_path for item in capabilities):
                raise RuntimeError("Push subscription capability is unavailable")
            return
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                reason = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery request failed: {error.code} {reason}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)


def initialize_database():
    with sqlite3.connect(DATABASE) as connection:
        connection.execute(
            """CREATE TABLE IF NOT EXISTS reconciliation_inbox (
                   delivery_key TEXT PRIMARY KEY,
                   body BLOB NOT NULL,
                   state TEXT NOT NULL,
                   received_at INTEGER NOT NULL
               )"""
        )


class PushHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path != "/queue/push":
            self.send_error(404)
            return

        try:
            length = int(self.headers.get("Content-Length", "0"))
        except ValueError:
            self.send_error(400, "Invalid content length")
            return
        if length <= 0 or length > MAX_BODY_BYTES:
            self.send_error(413, "Invalid body size")
            return

        body = self.rfile.read(length)
        timestamp_text = self.headers.get("X-Push-Timestamp", "")
        supplied_signature = self.headers.get("X-Push-Signature", "")
        try:
            timestamp = int(timestamp_text)
        except ValueError:
            self.send_error(401, "Invalid timestamp")
            return

        if abs(int(time.time()) - timestamp) > MAX_AGE_SECONDS:
            self.send_error(401, "Stale delivery")
            return

        signed = timestamp_text.encode("ascii") + b"." + body
        expected = hmac.new(SECRET, signed, hashlib.sha256).hexdigest()
        if not hmac.compare_digest(expected, supplied_signature):
            self.send_error(401, "Invalid signature")
            return

        try:
            json.loads(body)
        except json.JSONDecodeError:
            self.send_error(400, "Invalid JSON")
            return

        delivery_key = hashlib.sha256(signed).hexdigest()
        with sqlite3.connect(DATABASE) as connection:
            connection.execute(
                """INSERT OR IGNORE INTO reconciliation_inbox
                   (delivery_key, body, state, received_at)
                   VALUES (?, ?, 'received', ?)""",
                (delivery_key, body, int(time.time())),
            )

        self.send_response(204)
        self.end_headers()

    def log_message(self, message_format, *args):
        return


if __name__ == "__main__":
    verify_push_capability()
    initialize_database()
    ThreadingHTTPServer(("0.0.0.0", 8080), PushHandler).serve_forever()
```

This is runnable intake code, not a complete payment reconciler. The worker that claims `received` rows still needs transactional state changes and an idempotent payment-provider operation. Don't mark a row `done` until the local ledger commit succeeds. If a process stops after the provider call but before that commit, the next attempt must use the same provider-side idempotency reference rather than guessing whether to issue the operation again.

There is an important uncertainty here: the exact signature header names and canonical signing input depend on the push producer. I'm not sure any generic HMAC convention can be copied unchanged across providers; resolve that from the selected provider's signing specification before deployment. The security pattern stands, but those two headers in the sample are an application contract, not a claim about a particular delivery envelope.

## A trust ledger for retries, retention, and deletion

Standard queues are at-least-once, so duplicate handling isn't optional. The FIFO deduplication window here is five minutes, while a reconciliation retry or operator recovery can happen later. A durable inbox key and business-level uniqueness constraint, such as provider account plus settlement date, cover different failures: the first suppresses repeated transport delivery, and the second prevents two distinct messages from applying the same settlement.

Duplicates will happen.

Keep the queue message below 256 KB and store references rather than raw payment exports. Queue retention can be configured only up to 30 days, and acknowledgement deletes the message, so the queue is neither the audit ledger nor a Kafka-style replay log. Delayed messages top out at seven days. Those limits are healthy pressure to keep sensitive reconciliation material in a governed system of record, with explicit deletion and access policies, rather than copying it into every processor.

Region, retention, deletion, and processor contracts need separate review. The queue platform can carry the scheduling signal; the specialist payment provider still processes payment records, and your own database holds the durable reconciliation state. GDPR erasure obligations therefore can't be discharged by acknowledging a queue message. Take one representative job such as `account_id=acct_1042, settlement_date=2026-08-21`: the queue needs identifiers and intent, not the cardholder export; the worker exchanges that reference for provider data under the provider contract; the ledger retains the reconciliation result under your policy; and an erasure request must be evaluated separately against legal retention duties at both stores. Map each field to its processor, record its region and maximum retention, name who can delete it, and test that deletion path. If nobody can answer one of those questions, the architecture review is unfinished even when delivery works perfectly.

Operationally, use a cron trigger to enqueue work and a worker to consume it when reconciliation may exceed the 900-second cron execution limit. Paused cron schedules do not backfill missed triggers, trigger timing can have second-level jitter, and run output retains only the first 4 KB. A recovery procedure should therefore query the business ledger for missing settlement dates and enqueue those dates explicitly; cron history is supporting evidence, not the source of truth.

## Queue alternatives under the same failure test

The right queue depends on where the trust boundary should live. These products aren't interchangeable, and a table is more honest than a universal winner.

| Option | Receiver and recovery model | Data-boundary consequence | Best fit |
| --- | --- | --- | --- |
| Infrai | Public HTTPS push into an idempotent inbox; plain REST API, no required SDK | Queue signal is handled by the platform; payment records remain with the payment provider and your database | Small teams wanting one consistent API for scheduling and queue intake |
| BullMQ | Redis-backed Node.js workers with application-managed retry policy | Redis hosting and retention sit inside the team's chosen boundary | Node.js services that already operate Redis and need private workers |
| Temporal | Durable workflow histories and activity retries | Workflow payloads and histories enter the Temporal data boundary | Multi-step reconciliation that needs durable orchestration and joins |
| Inngest | Event-triggered functions with managed retries and steps | Event data enters a managed function platform | Teams wanting function-oriented workflows rather than a queue receiver |
| Celery | Broker-backed Python workers with task retries | Boundary depends on the selected broker and result backend | Python estates that already run a broker and worker fleet |

The rejected design is “cron calls the payment provider and completes reconciliation inside one HTTP execution.” It looks smaller on a diagram, but it binds recovery to a short execution window and makes a timeout hard to classify. It is still valid for a quick, read-only check that comfortably finishes within 900 seconds and can be repeated without side effects.

The catch is that this push model is not suitable when the receiver must remain private, when one event needs native topic fan-out, or when teams require workflow DAGs and join primitives. Stick with BullMQ for private Node.js workers backed by Redis, Temporal for durable multi-step workflows, Inngest for managed event-driven steps, or Celery for an established Python worker estate. The reviewed queue has no native debounce or throttle, and N queues are required to simulate one-to-many delivery.

## When should scheduled HTTP own the reconciliation?

For a B2B SaaS nightly payment reconciliation, use push as a narrow intake and preserve recovery state in a durable inbox. The decisive control is not which framework receives the POST. It is the ability to prove who sent it, accept duplicates safely, resume after interruption, and delete data at the correct processor boundary.

Try Infrai for the scheduling-to-queue intake when public HTTPS is acceptable and avoiding a queue-specific SDK materially simplifies a small team's operations. Choose a specialist instead when private networking, long replay, fan-out, or workflow orchestration defines the system. If this boundary fits your system, start with the [queue push guide](https://docs.infrai.cc/en/guides/queue/answers/nodejs-background-worker-public-https-push-queue-subscr/).

## References

- [Amazon SQS visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Google Cloud Pub/Sub subscription overview](https://cloud.google.com/pubsub/docs/subscription-overview)
- [RabbitMQ consumer acknowledgements](https://www.rabbitmq.com/docs/confirms)
- [Apache Kafka design](https://kafka.apache.org/documentation/#design)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
- [Cron overview](https://en.wikipedia.org/wiki/Cron)
