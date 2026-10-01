# Email Deliverability Fallback Strategy for SMS Alerts (Postgres Polling Without Webhooks)

TL;DR: Treat a bounce as evidence to update recipient state, not as an instruction to send a text. Poll the email event source, normalize each event into an append-only Postgres ledger, suppress a permanently invalid address, and allow one SMS alert only when the shipment message is still actionable, the phone number is eligible in that region, and an idempotency record wins a database uniqueness check.

For a US/EU logistics SaaS, the rule is narrow: a terminal delivery failure may change the channel, while a temporary failure should normally remain in the email retry path. Preserve the raw evidence and its source timestamp because an event feed is an observation boundary, not a perfectly ordered queue.

Order matters.

## How Should Email Deliverability Strategy Trigger an SMS Fallback Alert?

The design has four invariants. A provider event identifier is applied at most once. A permanent failure suppresses future email attempts for the same normalized recipient and tenant until an authorized address change reverses that state. An SMS escalation is unique for the business message and recipient, rather than merely unique for one bounce event. Finally, channel eligibility is evaluated at send time from the applicable consent and regional policy record; possession of a phone number is not permission to use it.

RFC 3463 defines a 4.X.X status as a persistent transient failure and a 5.X.X status as a permanent failure. Keep the raw diagnostic anyway: a parser can encounter an unknown code, a source can omit an enhanced code, and policy should fail closed instead of guessing that every event containing `bounce` means the mailbox is dead.

The poller may crash anywhere. It can crash after fetching a page, halfway through a transaction, after committing state, or while an SMS request is in flight. Database atomicity covers the first three. The last one requires a durable outbox and downstream idempotency when the transport offers it; without downstream deduplication, exactly-once external delivery is not a defensible claim.

Crashes are ordinary.

## Record the observation before taking the action

Use a cursor as a performance hint, never as the only proof that an event was consumed. Cursor persistence and event insertion belong in the same transaction, while a unique source-event key makes replay harmless. Poll with overlap if the source contract permits it, because timestamp pagination can otherwise lose events at a shared boundary or an event that becomes visible late. The safe overlap depends on the event API's ordering and retention guarantees.

This email deliverability strategy does not depend on receiving a webhook. A Node.js polling worker can fetch bounce events and write the same ledger even though the focused example below is Python; language choice does not move the transaction boundary. No webhook also means latency is bounded by the polling interval plus source visibility delay, so an operator must choose that interval from the shipment alert deadline and published rate limits rather than from a generic real-time claim.

The minimum useful records are an immutable event ledger, the current email suppression projection, a message record with its actionable deadline, and an outbox row. Store recipient identifiers with tenant scope. Limit access to raw diagnostics, which can contain addresses or remote-system text, and apply a documented retention period. GDPR Article 5's data-minimization and storage-limitation principles argue against retaining an unlimited troubleshooting archive merely because storage is cheap.

| Option | Replay behavior | Failure exposure | Appropriate use |
|---|---|---|---|
| Cursor only | A rewound cursor repeats side effects | Commit ambiguity can duplicate SMS | Read-only analytics where duplicate rows are acceptable |
| Event dedupe plus direct SMS | Duplicate events are contained | Crash around the external call remains ambiguous | Low-consequence alerts with downstream idempotency |
| Event ledger plus transactional outbox | State change and intent commit together | Dispatcher still needs retry and downstream dedupe | Operational shipment alerts requiring an audit trail |

A ledger costs storage and needs retention work. I still choose it here: support staff must be able to answer why an address was suppressed and why a shipment update crossed channels, while a mutable `last_bounce` column destroys the sequence that would answer both questions.

Replays are normal.

## Put the critical path in one transaction

This Python sketch assumes the polling adapter has authenticated, validated the response schema, and converted source-specific fields into a conservative internal event. It does not infer permanence from free-form text. Parameter binding belongs to the driver, and unique constraints provide concurrency control.

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Literal

BounceClass = Literal["transient", "permanent", "unknown"]

@dataclass(frozen=True)
class DeliveryEvent:
    source: str
    event_id: str
    tenant_id: str
    message_id: str
    recipient_key: str
    occurred_at: datetime
    bounce_class: BounceClass

def apply_event(conn, event: DeliveryEvent, now: datetime) -> bool:
    """Commit evidence, suppression, and any SMS intent atomically."""
    with conn.transaction():
        inserted = conn.execute(
            """
            INSERT INTO delivery_event
                (source, event_id, tenant_id, message_id, recipient_key,
                 occurred_at, bounce_class)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (source, event_id) DO NOTHING
            RETURNING event_id
            """,
            (event.source, event.event_id, event.tenant_id, event.message_id,
             event.recipient_key, event.occurred_at, event.bounce_class),
        ).fetchone()
        if inserted is None or event.bounce_class != "permanent":
            return False

        conn.execute(
            """
            INSERT INTO email_suppression
                (tenant_id, recipient_key, reason, effective_at)
            VALUES (%s, %s, 'permanent_bounce', %s)
            ON CONFLICT (tenant_id, recipient_key) DO UPDATE
            SET reason = EXCLUDED.reason,
                effective_at = LEAST(email_suppression.effective_at,
                                     EXCLUDED.effective_at)
            """,
            (event.tenant_id, event.recipient_key, event.occurred_at),
        )

        eligible = conn.execute(
            """
            SELECT 1 FROM outbound_message m
            JOIN channel_eligibility e
              ON e.tenant_id = m.tenant_id
             AND e.recipient_key = m.recipient_key
             AND e.channel = 'sms'
            WHERE m.tenant_id = %s AND m.message_id = %s
              AND m.purpose = 'shipment_status'
              AND m.actionable_until > %s AND e.allowed = TRUE
            """,
            (event.tenant_id, event.message_id, now),
        ).fetchone()
        if eligible is None:
            return False

        queued = conn.execute(
            """
            INSERT INTO notification_outbox
                (tenant_id, message_id, recipient_key, channel, state, created_at)
            VALUES (%s, %s, %s, 'sms', 'pending', %s)
            ON CONFLICT (tenant_id, message_id, recipient_key, channel)
            DO NOTHING RETURNING message_id
            """,
            (event.tenant_id, event.message_id, event.recipient_key, now),
        ).fetchone()
        return queued is not None
```

Two unique indexes are implied: `(source, event_id)` on the ledger and `(tenant_id, message_id, recipient_key, channel)` on the outbox. The latter expresses the business invariant. Dedupe on event ID alone is insufficient because one message can produce several distinct events.

The `LEAST` expression retains the earliest known permanent-failure time when an older event arrives late. An authorized address change should create a new address identity or explicit suppression-release event. Quietly overwriting the projection would make the audit trail disagree with present state.

One row is not a history.

## Operate the state machine, not just the poller

The polling loop needs bounded retries with randomized backoff for transient transport failures, explicit handling for authentication or schema failures, and a page limit that prevents one tenant's backlog from starving others. Advance the cursor only with committed event rows. If the source provides no stable event identifier, use a documented composite key and test collision cases against retained fixtures; do not hash mutable diagnostic prose and call it identity.

Watch age, not merely counts. Useful signals include the age of the oldest unprocessed event, cursor progress by tenant and region, unknown classifications, suppressed send attempts, outbox age, dispatcher retries, and the interval between the email event timestamp and SMS acceptance. Counts can look healthy while one regional partition has stopped moving.

Testing should force ugly sequences: the same page twice; page two before page one; two permanent events for one message; a transient event followed by a permanent one; an event after the shipment deadline; withdrawn SMS eligibility; a rollback after ledger insertion; and two pollers racing on one tenant. Test the dispatcher against a fake transport that times out after recording a request, since this exposes external-call ambiguity that an in-process mock usually hides.

Deploy with SMS dispatch disabled while the ledger and projection run in shadow mode, compare classifications with sampled source evidence, then enable a narrow cohort with an emergency disable control. That switch should stop new dispatch claims without deleting pending evidence. It is an operational brake, not a substitute for idempotency.

## Why reject immediate fallback?

The rejected design sends SMS directly from the poll loop as soon as any bounce-like event appears. It has fewer tables and lower apparent latency. It also couples an unreliable observation boundary to an irreversible external side effect, treats transient and permanent outcomes alike unless every adapter is flawless, and makes a cursor replay customer-facing.

Immediate fallback is valid for a low-consequence internal alert where recipients already opted into that channel, duplicates are tolerable, classification is authoritative, and the downstream request accepts a stable idempotency key. A shipment exception is stricter. The message can expire, permissions can change, and duplicate texts erode trust even when every component reports success.

The architecture is deliberately plain: poll, record, classify, project, authorize, enqueue, dispatch, and reconcile. **Reliability comes from keeping those state transitions inspectable and replayable.** SMS is an escalation path, not a side effect of parsing the word `bounce`.

## References

- RFC 3463, Enhanced Mail System Status Codes: https://www.rfc-editor.org/rfc/rfc3463
- RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- PostgreSQL transaction isolation: https://www.postgresql.org/docs/current/transaction-iso.html
- PostgreSQL `INSERT` and `ON CONFLICT`: https://www.postgresql.org/docs/current/sql-insert.html
- European Union, GDPR Article 5 principles: https://eur-lex.europa.eu/eli/reg/2016/679/oj
- IETF RFC 9457, Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc9457
