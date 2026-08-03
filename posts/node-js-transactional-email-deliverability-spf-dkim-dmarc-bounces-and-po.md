# Node.js Transactional Email Deliverability: SPF, DKIM, DMARC, Bounces, and Polling

**For a Node.js transactional email deliverability setup, verify the sending domain before release, require aligned authentication, and suppress permanent bounces before another send can leave the queue.**

The Node.js call that submits a message is the least interesting part. I care about the boundaries around it: DNS can lag, verification can remain pending, a queue can retry, and a recipient can become ineligible between enqueue and send. Receipts and OTPs make those gaps visible fast.

My architecture decision is to keep message intent, transport submission, and delivery feedback as separate states. A deployment gate owns domain verification. A sender owns idempotent submission. An event consumer owns bounce classification and suppression. No component gets to infer that an accepted request reached an inbox.

## How should a Node.js transactional email setup verify a domain with SPF, DKIM, and DMARC?

Use a dedicated transactional subdomain, publish the exact authentication records selected for that domain, and block production traffic until public DNS and the sending system both report the expected state. The visible `From` identity should align with an authenticated domain. DMARC is the policy layer that evaluates identifier alignment and tells receivers what the domain owner requests when authentication doesn't align; RFC 7489 also defines aggregate and failure reporting. I begin with observation, inspect every legitimate sender, then tighten policy through a reviewed change. A copied enforcement record can strand password resets from a forgotten system.

SPF, DKIM, and DMARC have different jobs. SPF associates authorized sending infrastructure with the envelope identity. DKIM attaches a cryptographic signature tied to a signing domain. DMARC evaluates alignment against the author-visible domain and publishes policy. Treating any one of them as a magic inbox switch misses the operational problem. Authentication makes a message accountable; recipient quality, complaint handling, content, and sending behavior still shape delivery outcomes.

I record four invariants in the ADR. The transactional stream has an owner. Every production `From` domain has a reviewed authentication configuration. A release cannot replace an active signing path until the replacement is observed and accepted. Suppression is checked at the final send boundary, not merely when a job enters the queue. These are deliberately provider-independent because DNS ownership and recipient policy outlive a transport contract.

Polling needs two witnesses — public DNS and the verification control plane — because either view alone is incomplete. A DNS match proves that a resolver can see the intended value; it doesn't prove that the sender has accepted it. A verified control-plane state without a matching public answer is equally unsuitable for a fresh release. DNS caches make timing variable, so use a deadline and backoff instead of promising an exact propagation time. I'm not sure why fixed, endless polling loops remain common in deployment scripts. They turn a bounded readiness check into an invisible release hang.

## The invariants and failure boundaries

The critical path starts before submission. Normalize the recipient, consult the local suppression ledger, reserve a stable message intent, render the approved template, and submit once. Store the transport's external message identifier as evidence, not as your primary key. On the feedback path, authenticate the incoming event according to the selected transport's documented scheme, deduplicate it, preserve the minimum audit material your policy requires, and apply the state transition in one transaction.

Retries bite.

I hit a duplicate-write bug when a naive retry ran the same suppression operation twice after a worker lost its acknowledgment. The address ended in the right state, but the retry created 2 audit rows and emitted two downstream notifications. That made the first database inspection misleading: the recipient looked safely blocked, while the audit consumer had already treated both rows as distinct work. I traced the event from the queue receipt through the database commit and found that deduplication happened after the side effects, where it could no longer protect them. I moved a unique event key, the suppression transition, and an outbox write into one transaction, then replayed the captured event to verify that the second delivery became a no-op. I've used that shape ever since because “already suppressed” doesn't mean “no duplicate side effects.”

Order matters.

The state model should keep submission and delivery claims apart. `queued`, `submitted`, `delivered`, `deferred`, `permanent_bounce`, `complaint`, and `suppressed` are useful internal concepts even if a transport uses different labels. Map external events at the adapter edge. A permanent bounce or complaint closes the send path for the applicable stream; a temporary deferral follows a bounded policy. Don't put a lost API response straight back on the wire unless your intent record and transport semantics make another submission safe.

Privacy belongs in the same design review. Logs need a correlation ID, template revision, message class, domain, state transition, and external message ID. They don't need an OTP, full body, or an unredacted recipient address. Open tracking is also a poor delivery invariant: Apple's Mail Privacy Protection can prevent senders from learning whether a recipient opened a message and can download remote content privately. Use authenticated events and recipient-visible outcomes for operational decisions, with open data treated as ambiguous telemetry.

## Comparing the architecture options

The useful comparison isn't a dashboard beauty contest. It is where correctness lives, who operates it, and what happens during a retry or provider change.

| Option | Where policy lives | Good fit | Main limitation |
|---|---|---|---|
| Transport-managed controls | Mostly in one sending account | Small teams with a single stream and limited operational staffing | Application context and cross-transport history may be thin |
| Application-owned delivery ledger | In local message, event, and suppression tables | OTP, account security, or several transactional streams | The team must operate event ingestion, retention, and reconciliation |
| Shared internal delivery service | Behind a stable organization-wide interface | Several applications that need common policy and audit rules | Adds a service boundary and an on-call surface |

I usually choose the application-owned ledger for authentication and account-recovery mail because eligibility can change after enqueue, and the application already understands tenant, purpose, consent, and risk. The transport's suppression controls remain a second guard. This is an engineering preference, not a universal rule; your mileage may vary with team size, mailbox mix, and compliance obligations.

The catch is operational ownership. A local ledger is not suitable when a small team cannot authenticate event callbacks, monitor a consumer, reconcile missing transitions, and handle data-retention requests. In that case, stick with transport-managed suppression and a narrow adapter, accepting the reduced policy detail. A shared service is valid when several applications genuinely share requirements, but it is a poor first move for one low-volume notifier. Centralization can convert a local mail incident into an organization-wide dependency.

Cost belongs in the comparison, but not as the leading criterion. Measure engineering time for DNS changes, key rotation, webhook review, reconciliation, incident response, and migration alongside message charges. I also test failure behavior in staging: duplicate the same signed event, reorder a delivery event and a bounce event, delay feedback, and enqueue a message immediately before suppression. The winning design is the one whose state transitions remain explainable under those cases.

## Polling verification and enforcing suppression in the critical path

The release probe below is intentionally transport-neutral and written in Python so the state machine is obvious. The Node.js application can expose the same contract through its deployment tooling. Inject one checker for public DNS and another for the documented verification API; don't invent a route or scrape a dashboard. The probe requires both observations in the same polling cycle, applies bounded exponential backoff, and fails closed at its deadline.

```python
import time
from collections.abc import Callable


Check = Callable[[], bool]


def wait_until_domain_ready(
    dns_matches: Check,
    verification_is_accepted: Check,
    timeout_seconds: int = 600,
) -> None:
    deadline = time.monotonic() + timeout_seconds
    delay_seconds = 5

    while time.monotonic() < deadline:
        if dns_matches() and verification_is_accepted():
            return
        time.sleep(delay_seconds)
        delay_seconds = min(delay_seconds * 2, 60)

    raise TimeoutError("domain verification did not complete before release deadline")


def may_submit(recipient_key: str, is_suppressed: Check) -> bool:
    # Bind the real checker to recipient_key in the application adapter.
    del recipient_key
    return not is_suppressed()
```

Keep the checker inputs explicit: record name, record type, expected value, signing selector, and domain identity. Normalize DNS presentation carefully, but compare against values issued for the actual domain. The verification API call must come from the chosen transport's current documentation, including its authentication method and terminal states. This generic seam is deliberate — API paths and response fields are contracts, not details to guess in a durable ADR.

Deployment records should capture the expected fingerprint, observation time, and release outcome. Don't log private signing material. During a planned signing-key change, retain the active path while the new one moves through verification, then switch through the same gate. Rollback should restore a previously verified configuration rather than create a third, unobserved state.

The rejected option is “send first and watch the bounce dashboard.” It has a valid use case for disposable development mail isolated from real recipients. It is wrong for production transactional traffic because it moves domain readiness, duplicate submission, and suppression checks beyond the application's enforceable boundary. For a durable setup, polling is a release concern and bounces are state transitions, not cleanup work after the campaign.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
