# Postgres User Email Lookup Returns Nothing Under Case Sensitivity and Verified State Filters

Short answer: An empty admin search for an email address does not establish that the account is absent. Before changing login or deleting an identity, compare the search predicate with the stored address, its verification state, and the identity linked to the account. In a developer-tools app adding phone one-time-code login, retain a narrow, access-controlled email lookup long enough to diagnose migration failures; do not retain raw one-time codes or indefinite search logs for that purpose.

The storage bill here is not primarily the email index. It is the accumulated event trail: a row per code request, delivery attempt, search, and login attempt, multiplied by retention time. For illustration, a system producing 100,000 such events per day keeps 3 million event rows after 30 days and 36.5 million after 365 days, before accounting for indexes and replicas. Those are arithmetic examples, not measured workload or price estimates. Shortening raw-event retention from 365 to 30 days changes that dominant term; dropping the email lookup first changes the wrong term and makes the empty-result complaint harder to investigate.

## How do I debug user lookup when email returns nothing?

Start at the lookup boundary. An admin form may send a raw string to a database equality predicate, while signup stored a normalized value; a filter may silently require a verified address; or the phone-login migration may have left email on a linked identity rather than the primary user row. These are distinct failure modes with different repairs. A blank result alone cannot tell them apart.

Nothing has been deleted yet.

Record the search input after trimming surrounding whitespace, then inspect the actual predicate and its parameters in a restricted diagnostic context. Compare the submitted address against both the stored email and any verified-email field or identity relation. Check the account identifier on each match before joining or merging anything. A case-folded diagnostic match is evidence for investigating normalization, not permission to treat two records as the same person. Email local parts have historically been case-sensitive in SMTP, even though many real systems treat them otherwise; a blanket lowercase migration needs a documented policy and collision review. Domain names are case-insensitive. The standards matter here because a convenience search must not quietly become an identity proof.

Verification introduces a second trap. A search scoped to `verified = true` can hide a real, unverified address, while a search across every address can expose account existence to an unauthorized operator. Keep those states distinct in internal diagnostics and return only what the operator's role permits. OWASP's authentication guidance calls out account enumeration through differing responses; admin search is an access-controlled operational tool, not a public account-discovery API.

The limitation of a broad diagnostic search is precisely that it can surface addresses the normal admin view intentionally excludes. Give its operator a specific permission and audit access to the diagnostic itself; otherwise, fixing a missed result trades away the boundary that kept an unverified address out of ordinary search. A case-insensitive match also cannot settle the ownership question. Where two stored identities fold to the same search key, investigate the provenance of both and halt any automatic merge, even if that delays rollout. A support agent needs a reliable account ID and verification evidence, not an attractive match that might attach a phone credential to someone else's account.

## Which records are worth retaining through the phone migration?

Keep the durable mapping among account ID, linked login identities, normalized search key under an explicit policy, original email value where required, and verification state. An indexed lookup on a separate, permissioned email-identity table can stay useful after phone becomes a login method. Keep the verification timestamp or equivalent provenance when the application needs to distinguish an asserted address from one that completed verification; the exact schema depends on the existing identity model.

| Record | Keep for | Boundary |
| --- | --- | --- |
| Account and linked identity IDs | Reconcile email and phone logins against one principal | Never infer a merge from matching contact strings alone |
| Email and verification state | Explain exact-match and verified-only search misses | Restrict who can query or export personal data |
| One-time-code verifier | Complete a short-lived challenge | Expire it; do not keep plaintext codes for support |
| Raw diagnostic events | Investigate a bounded incident window | Set a documented retention period and access controls |

The focused diagnostic below assumes `email_identities` is already a restricted table and `email_search_key` was computed under the application's documented policy. It deliberately returns candidate account IDs and status for an authorized investigation; it does not authenticate a user, infer ownership of an unverified address, or merge records. Run it against a controlled snapshot or with read-only credentials, and review collisions before applying a unique index to historical data.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class EmailCandidate:
    account_id: str
    stored_email: str
    verified: bool


def inspect_email_candidates(connection, submitted_email: str):
    search_key = submitted_email.strip().casefold()
    with connection.cursor() as cursor:
        cursor.execute(
            """SELECT account_id, email, verified
               FROM email_identities
               WHERE email_search_key = %s""",
            (search_key,),
        )
        return [EmailCandidate(*row) for row in cursor.fetchall()]
```

Python's `casefold()` is a diagnostic illustration, not a universal email canonicalization rule: Unicode case folding and mailbox-provider behavior can differ from the application's stored key policy. Use the same versioned normalization routine that produced the index, or a mismatch in the diagnostic itself can create another false negative. Also test the exact stored value separately; a match produced only by a broad fold should be flagged for review, not silently accepted.

No match is proof of absence.

## How should the investigation change the rollout?

Add a test matrix with the original-case address, a case variant, an unverified address, a verified address, and two accounts whose proposed search keys collide. Check the admin role boundary and the response for an unauthorized caller as carefully as the query result. For the phone flow, test code expiration, replay rejection, throttling, and a user whose phone and email identities are linked to the same account; a new phone credential must not bypass the existing session policy merely because the SMS was delivered. NIST's digital identity guidance describes restrictions on out-of-band authentication, including the risks of the PSTN channel. Delivery is not account ownership evidence by itself.

Deploy the lookup change behind a migration check that counts conflicting search keys and compares exact versus policy-normalized matches, without exporting full addresses into general logs. Monitor authorized empty-result rates by search mode and verification state using bounded, privacy-conscious aggregates. If the rate shifts after migration, sample records through a controlled support workflow before changing the normalization policy. Stop the rollout on collisions. The failure to watch for is a false positive account match, not merely an empty screen.

This retention approach has a limit: a short diagnostic window makes older, individual search failures impossible to reconstruct. Extending that window may help an investigation but holds sensitive contact information longer, increases the growing event count, and expands the set of people and systems that must be trusted with it. Choose the window against the time it actually takes the team to detect and investigate a problem, then verify that the access policy and deletion job enforce the choice.

Finally, expire code verifiers and discard raw search strings and delivery traces after the operational window set by the team's security and data-retention requirements. That lowers the growing storage term and limits exposure of sensitive identifiers. The cost is real: after the window closes, an individual old delivery attempt may no longer be reconstructable. Preserve aggregate counts and durable identity state so the team can still diagnose systemic regressions without keeping every raw event forever.

## Further reading

- References: OWASP Authentication Cheat Sheet, https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- RFC 5321, Simple Mail Transfer Protocol, https://www.rfc-editor.org/rfc/rfc5321
- RFC 1035, Domain Names - Implementation and Specification, https://www.rfc-editor.org/rfc/rfc1035
- NIST SP 800-63B, Digital Identity Guidelines: Authentication and Lifecycle Management, https://pages.nist.gov/800-63-3/sp800-63b.html
