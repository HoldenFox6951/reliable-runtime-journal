# Loyalty Account Deduplication: 3 Identity Resolution Gates Before Enrollment

When a game adds phone one-time-code login to an existing loyalty program, the dangerous moment is before the user row exists. A valid code proves control of a number at that moment; it does not prove that the caller is a distinct member, a human, or entitled to another welcome reward.

Short answer: resolve the phone identity and score abuse risk before creating a loyalty account, then make the decision idempotent so retries cannot mint another member or reward.

That ordering sounds obvious. It is easy to violate when the OTP endpoint, profile service, and rewards ledger were built by different teams. I treat account creation as a commit point, not as a side effect of sending a code.

## What should identity resolution check before a loyalty account exists?

Start with a canonical representation of the phone number. Parse an international number with a well-maintained library, store the normalized E.164 value, and keep the raw input only when a documented audit need justifies it. Never use formatting, carrier labels, or a client-supplied country flag as an identity key.

Then look up existing links in a privacy-preserving index. A keyed digest (for example, HMAC-SHA-256 with a rotatable server key) lets the matcher find a prior member without putting the phone number in every analytics table. The digest is not magic: access to the key still needs separation, rotation, and an audit trail.

The lookup should return a small set of states rather than a boolean:

Keep it boring.

| Match state | Meaning | Safe next action |
| --- | --- | --- |
| no match | No active loyalty identity is linked to the normalized number | Continue risk checks; a new account may be eligible |
| one active match | A member already owns the number | Authenticate or start a recovery flow; do not create a second member |
| several historical matches | Merges, recycled numbers, or old imports need review | Pause enrollment and request a stronger recovery signal |
| suppressed | The number or device is on an abuse blocklist | Reject without revealing which rule fired |

The response should not tell an unauthenticated caller whether a number is registered. OWASP calls this out as account-enumeration risk; a uniform message and similar timing matter as much as the database query.

## How do risk signals change the OTP enrollment decision?

Identity resolution answers “have we seen this number?” Abuse resistance asks “should this attempt be allowed to create value?” Keep those decisions separate so a known member can sign in even when a new-enrollment rule is strict.

For a gaming loyalty flow, three signals are usually enough to make the first version legible:

1. **Velocity:** count sends, verifications, and account-creation attempts per number, device, IP range, and reward campaign. A 429 response after a defined threshold is a control, not an identity result.
2. **Binding quality:** require the OTP to be verified in the same short-lived transaction that requested it. Bind a server-generated nonce to the normalized number, client context, and purpose; expire it after 10 minutes and consume it once.
3. **History:** consider prior chargebacks, referral reversals, device reuse, and number age only as policy inputs. A recycled number is a reason to ask for recovery evidence, not proof of fraud.

I keep the policy explainable. A score of 0–39 can proceed, 40–69 can proceed with a delayed reward, and 70 or higher can require manual review; those bands are starting points to calibrate against false positives, not universal truth. Your mileage may vary by region, carrier, and campaign.

The important invariant is that scoring happens before the write that grants membership benefits. Logging a suspicious event after inserting the member row is observability, not prevention.

## A transaction boundary that survives retries

The API can be ordinary HTTP. The storage semantics cannot be casual. Create an enrollment record keyed by an idempotency token, and place a unique constraint on the canonical phone identity for active memberships. The token should be scoped to the purpose and authenticated session, not accepted as a globally reusable user claim. In practice, this means the request that verifies a six-digit code must carry the same enrollment identifier through the risk decision, membership insert, and reward outbox write; if the client retries after a network timeout, the server reads the existing decision by that token, checks that the nonce was already consumed, and returns the original member state instead of running a second policy evaluation, while a separate reconciliation job checks that every created member has exactly one corresponding reward record and sends an alert when the counts diverge.

Here is a compact Python sketch. It leaves the risk policy injectable and makes the database uniqueness rule the final backstop.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass(frozen=True)
class Enrollment:
    phone_digest: str
    nonce: str
    expires_at: datetime
    purpose: str


def can_create_member(store, enrollment: Enrollment, otp: str, risk_score: int):
    now = datetime.now(timezone.utc)
    if enrollment.purpose != "loyalty_login" or now >= enrollment.expires_at:
        return {"status": "deny", "reason": "expired_transaction"}
    if not store.consume_otp_once(enrollment.nonce, otp):
        return {"status": "deny", "reason": "invalid_code"}
    if store.active_member_for_phone(enrollment.phone_digest):
        return {"status": "sign_in", "reason": "existing_identity"}
    if risk_score >= 70:
        return {"status": "review", "reason": "enrollment_risk"}
    try:
        member = store.insert_member_unique_phone(enrollment.phone_digest)
    except store.UniquePhoneConflict:
        # A concurrent request won the race; read the winner, do not retry the insert.
        member = store.active_member_for_phone(enrollment.phone_digest)
    return {"status": "created", "member_id": member.id}
```

The catch is that a unique phone constraint cannot decide whether a number is trustworthy; it only prevents two writers from winning at once. You still need an atomic reward grant, an outbox event for downstream systems, and a replay-safe consumer. Otherwise a timeout between the member insert and the reward call becomes a support ticket with a suspiciously generous resolution.

## Which architecture trade-offs matter in production?

Teams often reach for a single “identity service” and hide every policy inside it. That centralizes control, but it also makes a campaign launch depend on a queue of unrelated changes. A small resolver plus explicit policy modules is easier to test and to retire.

| Choice | Helps with | Cost or boundary |
| --- | --- | --- |
| HMAC digest index | Limits broad exposure of phone values | Key custody and rotation become security-critical |
| Synchronous risk decision | Gives the client a clear enrollment result | Adds latency and a dependency on scoring data |
| Delayed rewards | Reduces incentive for scripted signups | Product teams must explain the pending state |
| Manual review for high risk | Catches unusual clusters | Review capacity becomes a hard throughput limit |

Do not use device fingerprinting as a sole identity proof. Browsers reset it, households share devices, and accessibility tools can look unusual. Likewise, carrier lookup can enrich a decision but cannot establish account ownership. Standards-oriented controls still win: rate limits, generic authentication errors, short-lived challenges, and explicit recovery paths.

For observability, record decision reason codes, policy version, nonce age, and latency without logging OTPs or raw phone numbers. Alert on spikes in sends and on divergence between “OTP verified” and “member created.” A dashboard that only shows successful logins will miss the attack that burns SMS capacity and never completes verification.

## A staged rollout for an existing loyalty database

First, normalize and backfill phone identities in a shadow index. Measure collisions, blanks, and numbers linked to multiple historical accounts; do not merge automatically just to improve the metric. Next, run the resolver in observe-only mode while the old signup path remains authoritative. Compare its proposed state with support outcomes and reward reversals.

Then enforce the transaction boundary for a small campaign cohort. Keep a kill switch that disables new enrollment while preserving sign-in for existing members. After the cohort is stable, migrate rewards to an outbox-driven grant and remove any code path that creates a member before OTP verification.

This is not suitable when your loyalty program intentionally permits household accounts sharing one number. In that case, keep phone verification as a contact factor and use a separate, policy-approved household key; forcing uniqueness would lock out legitimate members. Stick with a looser model when the business rule is shared ownership, but document how rewards and recovery are attributed.

The design decision is modest: a phone code is a factor, not a person. Put identity resolution, abuse scoring, and an idempotent commit in that order, and the rest of the system has a boundary it can reason about.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://www.rfc-editor.org/rfc/rfc4226
- https://www.rfc-editor.org/rfc/rfc6238
