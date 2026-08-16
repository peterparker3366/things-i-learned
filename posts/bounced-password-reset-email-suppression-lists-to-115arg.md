# Bounced Password Reset Email: Suppression Lists, Token Expiry, and Rate Limits

Use the suppression list before you mint a password reset token, not after the support ticket lands. On a freight dispatch portal the emailed link is the entire recovery path, and if that address hard-bounced three weeks ago then the hashed token, the expiry, the single-use consume and the rate limit sitting behind it are all theater. The account is unrecoverable and nothing in the security design says so.

Deliverability is the first control in this flow.

That ordering isn't a matter of taste. NIST SP 800-63B is blunt about it: methods that don't prove possession of a specific device — email among them — can't be used for out-of-band authentication. A reset link is a recovery mechanism balanced on an address you collected at onboarding and probably haven't touched since. Treating the mailbox as a verified fact is where most of these builds go wrong.

## Start from the mailbox, not from the token

Address quality in logistics decays faster than in consumer products. Dispatcher accounts get typed into a terminal at a warehouse counter (`@gmial.com` shows up more than you'd like). Shipper HR sends CSV imports with role mailboxes like `dispatch@` behind aggressive filtering. Drivers leave mid-season and the carrier deletes their mailbox the same week, which turns a working address into a permanent failure without anyone telling your application. None of that is visible from the accounts table. It's only visible in the delivery feed.

RFC 3463 gives you the vocabulary. Enhanced status codes are `class.subject.detail`, where a 5.x.x class is a permanent failure and a 4.x.x class is persistent transient — the sending side should keep retrying for a while. A `5.1.1` means the destination mailbox doesn't exist. A `4.2.2` means the mailbox is full today and may not be tomorrow. Collapsing those two into one "bounced" flag is the single most expensive shortcut in this area, because you either suppress people who were briefly over quota or keep hammering addresses that will never accept mail again.

| Delivery signal | What it usually means | What the reset flow should do |
| --- | --- | --- |
| `5.1.x` permanent | mailbox gone or never existed | suppress the address, stop issuing reset mail, hand the account to a verified support path |
| `4.2.2` and other transient | full mailbox, temporary policy | keep the send queued, suppress only after repeated strikes |
| `4.7.x` greylisting | first attempt deliberately delayed | leave it alone and make sure the token outlives the retry window |
| complaint feedback | recipient marked mail as spam | suppress, and check whether transactional and marketing mail share a domain |

Store that verdict somewhere your application logic can read synchronously: a suppression table keyed by normalized address, with reason, first seen, last seen and a strike count. A dashboard nobody queries at request time is not a suppression list.

## How should a Node.js password reset flow handle a bounced email link?

The public behavior doesn't change at all. The request endpoint answers "if that address is on file, a reset link is on its way" for a known account, an unknown account, and a suppressed account alike — OWASP's guidance on this is old and still right, because a differentiated response hands an attacker a free account-enumeration oracle. What changes is everything behind that response.

For a suppressed address, don't mint a token. Record the attempt against the account so support tooling can see it, skip the send, and leave the rate-limit budget untouched. Issuing a token you know can't be delivered costs you three things: reputation with the receiving domain, a support ticket that arrives without context, and a user who tries four more times because the interface implied success.

In the fleet portals I've worked on, the Express layer stays thin — validate, call the policy service, return the constant response — while the messaging service owns suppression, token state and the send queue. Python here, since that's where the delivery events already land.

```python
import hashlib
import secrets
from datetime import datetime, timedelta, timezone

RESET_TTL = timedelta(hours=1)          # sized from delivery p99, not from habit


def normalize(address: str) -> str:
    local, _, domain = address.strip().partition("@")
    return f"{local}@{domain.lower()}"


def request_reset(db, raw_address: str, now: datetime) -> None:
    """Returns nothing. The HTTP layer answers identically every time."""
    address = normalize(raw_address)

    if db.suppression_state(address) == "hard":
        db.record_blocked_attempt(address, now)      # support tooling reads this
        return

    account = db.account_by_address(address)
    if account is None:
        return

    token = secrets.token_urlsafe(32)                # 256 bits from the OS CSPRNG
    digest = hashlib.sha256(token.encode()).hexdigest()
    db.invalidate_open_tokens(account.id, now)       # one live link per account
    db.insert_token(account.id, digest, expires_at=now + RESET_TTL)
    db.enqueue_reset_mail(address, token, dedupe_key=digest)


def consume(db, submitted: str, password_hash: str, now: datetime) -> bool:
    digest = hashlib.sha256(submitted.encode()).hexdigest()
    with db.transaction():
        rows = db.execute(
            "UPDATE reset_token SET consumed_at = %s "
            "WHERE digest = %s AND consumed_at IS NULL AND expires_at > %s "
            "RETURNING account_id",
            (now, digest, now),
        )
        if not rows:
            return False                             # expired, spent, or never issued
        db.set_password(rows[0].account_id, password_hash)
        db.revoke_active_logins(rows[0].account_id)
    return True
```

Only the digest is stored, so a database dump doesn't hand anyone a working link. The single-use property comes from the conditional `UPDATE`, not from reading a row and updating it later — two dispatchers double-clicking the same link half a second apart must produce exactly one password change.

## Expiry is a delivery budget, not a security dial

Everyone reaches for 15 minutes.

It reads as careful, and against a greylisting receiver it produces a dead link that the user will immediately try to replace.

Greylisting, described in RFC 6647, works by temporarily rejecting mail from an unfamiliar sender and accepting it on a later attempt. The delay is whatever the receiver and your queue negotiate, which for a first-time sending domain can run well past a quarter of an hour. Corporate scanners add their own hold. So the failure isn't subtle: the driver taps the link, gets an error page, requests another one, and your send volume to a domain that already distrusts you triples. Expired-token rate and reset-request rate climb together, which is exactly the signature a receiving domain reads as abuse.

Size the lifetime from your own measurement — accepted-at to first-click, at p99, per destination domain — and then decide how much of that you're willing to lose. An hour with strict single-use, immediate invalidation of older tokens, and revocation of active logins on success is a defensible position for a portal like this. Fifteen minutes is defensible too, once you can show the mail lands in two. Your mileage may vary, and the honest answer is that nobody can pick this number for you from the outside.

The catch is support load. Longer expiry means a leaked link stays useful longer, so it's only reasonable if the consume step is genuinely atomic and you're revoking prior tokens on every new request. If your threat model includes shared terminals in a warehouse, shorten it and pay the support cost knowingly.

## Limits that count sends, bounces, and destination domains

Three counters, not one. Per account, so a single dispatcher can't be mail-bombed by an attacker who knows their address. Per source IP, which catches the trivial scripted sweep. Per destination domain, which is the one teams skip and the one that protects your sending reputation when a shipper's whole mail estate starts rejecting.

That third counter is a circuit breaker. When permanent failures against one domain cross a threshold in a rolling window, stop sending there, alert operations, and let the accounts fall through to the support path. Google's sender guidelines put the spam complaint ceiling at 0.3% and expect SPF, DKIM and DMARC alignment on the sending domain; RFC 7208 and RFC 7489 are the specs behind those two. Transactional mail should sit on its own subdomain, separate from anything a marketing team touches, so a campaign can't cost a driver their password reset.

None of these counters may leak state. Same response, same latency profile, whether the request was rate-limited, suppressed, or perfectly ordinary.

```python
PERMANENT = {"5.1.1", "5.1.2", "5.1.3", "5.1.10"}   # RFC 3463 bad-destination codes


def ingest_delivery_event(db, event: dict, now: datetime) -> None:
    address = normalize(event["recipient"])
    status = event.get("status", "")                # enhanced status code, e.g. "5.1.1"

    if event["type"] == "complaint":
        db.suppress(address, reason="complaint", at=now)
    elif status in PERMANENT:
        db.suppress(address, reason="hard", at=now)
    elif status.startswith("4."):
        strikes = db.record_soft_failure(address, since=now - timedelta(days=7))
        if strikes >= 5:
            db.suppress(address, reason="soft-exhausted", at=now)
```

## Rolling suppression into a live portal, and the migration order that avoids lockouts

Turn it on in shadow mode first. Log what would have been suppressed for a week, then read the list by hand: role mailboxes and typo'd domains will dominate, and you'll find a handful of accounts that are genuinely stranded. Backfill from 90 days of delivery logs if you have them. Only then let `request_reset` actually short-circuit.

You need an escape hatch before you need it. Suppression without an operator override is an outage waiting for a Monday morning, so build the audited unsuppress action — who, when, why — in the same change that adds the table. In a logistics setting the identity check behind it is usually easy: a dispatcher confirms the driver at the terminal, the address gets corrected, the suppression entry is cleared with a reason attached.

Watch four numbers after rollout: hard-bounce rate per destination domain, reset completion rate, expired-token rate, and suppressed-attempt count. The last one is the interesting one. A spike there is either an attack or an address book that quietly went stale, and you can't tell which from the security metrics alone.

This shape isn't a good fit everywhere. A consumer product with fresh, self-selected addresses and a high engagement rate gets far less from per-domain breakers than a carrier does, and the extra machinery is real weight to carry. If your operations team needs recovery inside a minute at 3 a.m., email isn't suitable as the only path regardless of how the tokens are built — pair it with an operator-verified reset or a possession-based factor, and treat the mailbox as what it is: a channel you rent, not an identity you own.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc3463
- https://datatracker.ietf.org/doc/html/rfc6647
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.google.com/mail/answer/81126
- https://resend.com/docs/introduction
