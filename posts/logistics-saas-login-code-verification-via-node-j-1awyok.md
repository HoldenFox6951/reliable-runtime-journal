# Logistics SaaS Login Code Verification via Node.js SMS API — Retention Boundaries

TL;DR: The simplest reliable SMS OTP design is a small state machine backed by an append-only verification record and a separate suppression record. Count each send attempt, expire codes quickly, retry only transport failures, and stop sending to numbers that repeatedly fail validation or delivery. Keep enough evidence to investigate a driver locked out at a depot, but do not retain the OTP itself.

A logistics login has an awkward failure shape. A dispatcher may need access from a new device while a carrier's phone is roaming, a number has been reassigned, or a handset is temporarily unreachable. Treating every negative response as a retryable error turns one bad destination into a message storm. Treating every error as permanent strands legitimate users. The design decision is therefore about evidence and state, not finding a magic API.

## What does delivery reliability actually cost?

The destination is the cost center.

Keep this state separate.

The bill is made of events: OTP creation, delivery attempts, provider responses, suppression checks, and retained evidence. The dominant operational term is usually repeated attempts to the same unusable destination, because each attempt adds delay and another ambiguous outcome. The change that moves that term is a durable recipient state: after a confirmed invalid number or a bounded sequence of delivery failures, the login flow stops sending and asks for an alternate recovery path.

That means retaining less than many teams first expect. Store a hash of the OTP, an expiry timestamp, a request identifier, a normalized destination fingerprint, the attempt count, and the final provider classification. Do not store the plaintext code after verification, and do not keep message content in ordinary application logs. What gets deliberately discarded is the secret. The cost of an incident is that a support engineer cannot reconstruct it from the code; the retained state still shows when a code was issued, whether it was accepted, and why delivery stopped.

| Record | Retain | Why |
| --- | --- | --- |
| Verification attempt | request ID, expiry, outcome, timestamps | Reconstructs the login path |
| Recipient state | normalized hash, failure class, suppression time | Prevents repeated sends |
| OTP secret | salted hash only | Limits exposure if storage is read |
| Message body | never in application logs | Reduces sensitive-data spread |

I initially treated retention as a housekeeping detail. That was the wrong boundary: deleting delivery evidence too early makes a reliability incident look like a user mistake, while retaining the code creates a security problem. The useful record is the decision trail between those two extremes.

## How should a SaaS login use a Node.js SMS OTP API?

A bounce is not a single state. Normalize the number before lookup, then classify the result into at least four buckets: invalid or unreachable destination, throttling, transient transport failure, and accepted submission with unknown handset delivery. Only the first bucket should immediately advance suppression. A carrier rejection can be permanent; a timeout cannot tell you that.

For a logistics account, suppression must be scoped carefully. A number can be bad for one account record while still being valid for another, and a shared operations phone can serve several people. Keep the global destination signal separate from the account-level login state. A global hard suppression should require a strong signal; an account-level cooldown can be much more conservative.

NIST treats SMS or voice over the public switched telephone network as a restricted authenticator and calls out risks such as number reassignment and interception. That does not make SMS useless for a bounded login workflow, but it does make a second channel, recovery process, and abuse controls part of the design rather than optional polish.

This approach is a poor fit for high-assurance authentication where phishing resistance or hardware-backed identity is required. It is also a poor fit when the team cannot operate suppression review and recovery; in that case, a simpler non-SMS login method may create fewer dangerous false positives.

The retry worker should be boring.

## How should Node.js retry without amplifying failure?

Retries need a budget and a reason. A verification request should have one active code, one expiry, and a small send-attempt ceiling. Retry a timeout or a clearly transient upstream response after jittered backoff. Do not retry an invalid destination, a rejected sender identity, or a rate-limit response without honoring the supplied delay. A new OTP should invalidate the previous one only when the user explicitly requests a replacement; otherwise concurrent requests create confusing races.

The following Python example shows the policy boundary behind a Node.js service. The storage and transport interfaces are intentionally generic: the important part is that classification happens before another send.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta


@dataclass
class DeliveryResult:
    category: str  # accepted, transient, throttled, invalid
    retry_after_seconds: int | None = None


def next_action(result: DeliveryResult, attempts: int) -> str:
    if result.category == "invalid":
        return "suppress"
    if result.category == "throttled":
        return "wait" if result.retry_after_seconds else "stop"
    if result.category == "transient" and attempts < 2:
        return "retry_with_jitter"
    if result.category == "accepted":
        return "record_pending_delivery"
    return "stop"


def expired(issued_at: datetime, lifetime_seconds: int, now: datetime) -> bool:
    return (now - issued_at).total_seconds() >= lifetime_seconds
```

In production, the equivalent Node.js worker should make the attempt update conditional on the request version or an idempotency key. That protects against two workers both observing attempt zero. The user-facing response should stay deliberately vague: a caller should not learn whether a number is registered, suppressed, or simply rate-limited.

That distinction is the boundary.

## Which evidence survives a retention boundary?

Keep a short-lived verification record for the active challenge and a longer-lived delivery event with redacted destination data. The exact duration belongs in the team's threat model and retention policy; it should be a deliberate choice, not the default lifetime of the database. Partition access so support can see a failure class and request ID without reading authentication secrets.

Metrics should distinguish send requests, accepted submissions, delivery-confirmed events when available, invalid destinations, throttles, expiries, and successful verifications. A single success rate hides the useful failure mode. Alert on a change in invalid-destination ratio by region and on repeated attempts for the same destination fingerprint. For US and EU traffic, retain the region and policy version used for classification so an investigation can explain which rule ran.

Test the ugly cases directly: duplicate clicks, two devices requesting codes, a timeout after the provider accepted the message, a driver changing numbers between shifts, and a delayed response arriving after suppression. A fake transport that returns only success and failure will miss the hardest race.

## Decision rule

Choose the least complex implementation that gives the login service four durable facts: what was requested, what was attempted, why the destination was classified, and whether the code was verified before expiry. That can sit behind a managed transport, a direct carrier integration, or an internal gateway; the interface is secondary to the state and evidence contract.

For this logistics workflow, reliability means refusing to send again when the evidence says the destination is invalid, while preserving enough redacted history to correct a bad classification. A short retry budget, explicit suppression states, and hashed OTP records provide that balance without turning a login endpoint into an unbounded messaging queue.

The trade-off is visible: fewer retries protect delivery reliability and abuse limits, but they increase the number of users sent to recovery. That is acceptable only if recovery is staffed and observable.

## Further reading

- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
