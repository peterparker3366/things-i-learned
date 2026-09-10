# Recovery-Safe Patient Access with OAuth (Keeping Data Consent Explicit)

Short answer: let OAuth and phone codes prove who is signing in, but keep clinical-data consent and account recovery in separate, auditable state machines owned by the patient portal.

For a telemedicine platform sold to clinics, the deciding question is not which login button produces the fewest screens. It is what happens after a patient loses a phone, an email address changes, or a clinic relationship ends. A recovered identity must not silently inherit yesterday's permission to expose records. The architecture decision is therefore to link several authentication methods to one internal patient account while requiring the portal's own consent decision before every data-bearing session.

This is deliberately stricter than treating a successful identity-provider callback as permission. It also creates more work: recovery operations, consent evidence, and session issuance need distinct records and tests. That cost is justified where the application handles sensitive patient data; for a low-risk scheduling page with no clinical content, the same machinery may be excessive.

## Which invariants and failure boundaries matter?

The first invariant is plain: authentication is not consent. OAuth can establish that an external account completed an authorization flow, and a one-time code can establish control of a delivery channel under the portal's policy. Neither event says which clinical-data purposes the patient accepted, which clinic requested access, or whether that acceptance is still current. Those answers belong to a consent record with its own subject, purpose, policy version, decision, and timestamp.

The second invariant is that account linking is explicit. Matching an email address or phone number is useful evidence, but it isn't enough to merge two patient identities automatically. A recycled phone number is the edge case that matters here — deliverability success proves that a handset received a message, not that its current owner is the person represented by an old chart. Linking should require an authenticated session on the existing account or a reviewed recovery process.

Keep the failure boundaries narrow:

- An OAuth callback may create or resume an authentication transaction; it cannot create consent.
- A valid phone code may satisfy one authentication factor; it cannot approve an account merge.
- Recovery may restore access to the internal account; it cannot revive withdrawn or expired consent.
- Consent may permit a named data use; it cannot prove the identity standing behind the session.

One more rule saves trouble later: return generic authentication and recovery responses to the browser. The OWASP Authentication Cheat Sheet recommends generic error messages because different responses can disclose whether an account exists. Internally, operators still need precise reason codes and correlation identifiers. The patient sees “We couldn't complete that request”; the audit stream can distinguish `code_expired`, `identity_unlinked`, and `consent_required` without leaking that distinction across the public boundary.

Small boundary. Large payoff.

## How should a patient portal combine OAuth login convenience with explicit data consent?

Use a brokered internal account model. The portal owns a stable patient-account identifier, then attaches verified login identities to it: an OAuth subject from an issuer, a phone destination represented by a protected internal reference, or another method approved by policy. A login transaction resolves one of those identities to the internal account. Only after that resolution does a separate authorization step evaluate the requested clinical purpose against the current consent record.

The sequence should be visible in traces even if it feels like one flow in the interface. Start an authentication transaction with an opaque nonce and an intended return location. Complete OAuth or verify the phone code. Resolve the external identity to exactly one internal account. Evaluate whether the session is normal or a recovery session. Then load the consent decision for the clinic, purpose, and policy version. If consent is absent, withdrawn, or no longer applicable, show the consent screen; if it is current, issue a narrowly scoped application session. Don't bury the consent mutation inside the callback handler. Retries and duplicate callbacks are normal control-flow concerns, while recording consent is a legally and operationally meaningful write that deserves an explicit user action.

Recovery options expose the real trade-off:

| Option | Recovery strength | Consent behavior | Operational cost | Best fit |
| --- | --- | --- | --- | --- |
| OAuth identity plus a separately verified phone | Two independent channels can support recovery policy | Re-evaluate consent after recovery | More linking and support logic | Portals exposing clinical records |
| Phone one-time code as the only login identity | Simple sign-in path, but recovery depends heavily on phone-change handling | Re-evaluate consent after any ownership change | SMS delivery and number-reassignment review | Narrow portals with a strong staffed recovery desk |
| OAuth identity with staff-reviewed recovery | Recovery can use documented clinic procedures instead of a second consumer channel | Keep consent blocked until review completes | Slow and labor-intensive | Small populations where manual review is acceptable |

There is no universally correct row. Phone-only access can be reasonable when the portal has little data and a clinic can verify patients through an established offline process. A high-volume platform with clinical documents usually benefits from independent identities because a single lost channel otherwise becomes both the login failure and the recovery failure. Your mileage may vary with clinic staffing, patient population, and the data exposed; those facts should be written into the decision record rather than hidden behind a generic “passwordless” label.

Consent presentation needs the same precision. Show the clinic or data recipient, the purpose, the categories of data involved, and the consequence of declining. Store the decision separately from the authentication transaction so a login retry cannot replay it. If policy text changes materially, require a new decision tied to the new version. If consent is withdrawn, invalidate the relevant application authorization while leaving the patient's login identities intact. The user may still need to sign in to inspect settings, obtain support, or grant a narrower permission.

## What does the critical path look like in code?

The useful abstraction is a policy boundary, not an SDK wrapper. The following Python sketch keeps authentication evidence, recovery state, and consent evaluation separate. The names are intentionally domain-level; storage, cryptography, delivery, and OAuth validation sit behind interfaces whose implementations must follow the chosen standards and deployment policy.

```python
from dataclasses import dataclass
from enum import Enum


class SessionMode(str, Enum):
    NORMAL = "normal"
    RECOVERED = "recovered"


@dataclass(frozen=True)
class LoginEvidence:
    issuer: str
    subject: str
    transaction_id: str


@dataclass(frozen=True)
class DataRequest:
    clinic_id: str
    purpose: str
    policy_version: str


def finish_patient_access(evidence: LoginEvidence, request: DataRequest):
    identity = identity_store.find_verified(evidence.issuer, evidence.subject)
    if identity is None:
        return public_result("authentication_not_completed")

    account = account_store.get(identity.account_id)
    mode = recovery_policy.session_mode(account, evidence.transaction_id)

    decision = consent_store.current_decision(
        patient_id=account.patient_id,
        clinic_id=request.clinic_id,
        purpose=request.purpose,
        policy_version=request.policy_version,
    )

    audit.record_access_evaluation(
        patient_id=account.patient_id,
        transaction_id=evidence.transaction_id,
        session_mode=mode.value,
        consent_status=decision.status if decision else "missing",
    )

    if mode is SessionMode.RECOVERED or decision is None or not decision.allows_access:
        return consent_flow.begin(account.patient_id, request)

    return session_issuer.issue(
        account_id=account.id,
        clinic_id=request.clinic_id,
        purpose=request.purpose,
    )
```

The recovered-session branch is intentionally conservative. It doesn't claim that recovery cancels every prior decision; instead, it makes the portal re-evaluate and, under this policy, obtain a fresh decision before clinical access. A different organization may permit an existing decision after low-risk recovery. I'm not sure that choice can be standardized without knowing the recovery evidence and the sensitivity of the exposed data, so make it a named policy with reviewable inputs rather than an incidental `if` buried in a controller.

Test this path as a matrix, not as one happy-path browser test. Include an unlinked OAuth subject, an expired phone-code transaction, a duplicate callback, a recovered session with current consent, withdrawn consent, a changed policy version, and two identities that appear to share a contact value. Assert two outputs for every case: the deliberately vague public response and the exact internal audit event. Also verify that logs omit raw codes, tokens, and unnecessary patient data. Deliverability metrics should use coarse operational dimensions; authentication logs are a poor place to accumulate message bodies or personal details.

Deployment deserves a pause point. Roll out identity linking before making it the only route into existing accounts, observe unresolved-link and recovery-review volumes, and keep support procedures aligned with the state machine. Rate limits should apply to code requests, code verification, account lookup, and recovery initiation as separate controls. A single global counter can punish legitimate retries during carrier delay, while no per-account control invites targeted nuisance messages. The exact thresholds depend on traffic and abuse evidence, so they belong in monitored policy rather than in an article's magic numbers.

## Why reject consent stored inside the login session?

The rejected design puts a `consented=true` claim in the OAuth or phone-login session and treats successful login as enough to continue. It looks compact. It fails the recovery test: once a new channel restores the session, the application has no independent record explaining who accepted what, for which clinic and purpose, under which policy version. Session rotation, logout, and identity linking also become tangled with the lifecycle of the patient's decision.

The catch is that a separate consent ledger adds schema, retention policy, audit review, and another authorization lookup. It is not suitable when the “consent” is merely acceptance of a low-risk interface preference and no sensitive data access depends on it. In that narrower case, stick with ordinary application settings and avoid labeling every checkbox as consent.

For a patient portal that releases clinical information, keep the separation. The conclusion is operational, not vendor-specific: choose login methods by the recovery evidence they preserve, and authorize data from an explicit decision the portal can explain later.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
