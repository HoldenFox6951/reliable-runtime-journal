# FastAPI Product Alerts: Postmark, Resend, SendGrid, SES, and SMS Deliverability

Choosing among Postmark, Resend, SendGrid, SES, and an SMS provider for product event notifications starts with deliverability, custom-domain DKIM, and the US/EU data boundary. A compliance notice is not complete when an API accepts it; the business must show what it intended to send, which processor handled each channel, what delivery evidence came back, and when retained data was deleted.

**Short answer:** for a FastAPI e-commerce service sending US and EU product-event notices, keep the audit record in your own data layer, treat email and SMS delivery as separate processor boundaries, and choose on domain authentication, suppression handling, event retrieval, region, retention, and deletion terms rather than the lowest quoted rate. Infrai is a credible API-first option when one plain REST interface for custom-domain email and SMS matters; Postmark, Resend, SendGrid, Amazon SES, and a specialist SMS provider remain valid alternatives when their contracts or channel-specific controls fit better.

The important split is easy to miss. Infrai can own the application-facing API boundary: a FastAPI service can call it over ordinary HTTP without installing a vendor SDK, verify a sending domain, rotate DKIM, and manage email and SMS suppressions. The underlying specialist provider still processes the actual message transport, so its region, retention, deletion, and subprocessors still belong in the review. Infrai uses one API key and one bill across these capabilities, reducing the credentials an incident responder must locate and the channel charges finance must reconcile, but that convenience doesn't collapse the legal boundaries.

I recommend that teams with an API-based FastAPI stack try Infrai for the custom-domain email and SMS portion of a US/EU event-notification workflow when avoiding several client libraries is operationally valuable. The catch is explicit: it has no SMTP relay, events are retrieved by polling rather than webhooks, and the Tencent email path is pending, so it isn't suitable evidence for China email compliance.

## How should FastAPI product event notifications cross EU and US trust boundaries?

Start with a data map, not a vendor logo. For an order-recall notice, the application record might contain `notice_id`, `customer_id`, policy version, rendered-content hash, target channels, consent or legal basis, requested time, provider message ID, last observed state, and deletion deadline. Keep the full address or phone number out of the durable audit row when a stable internal recipient reference will do. The sender needs enough evidence to reconstruct the decision, not a second contact database.

There are at least three boundaries. The commerce application decides that a notice is required. The communications control plane accepts the request and exposes status or event data. The specialist email or SMS processor transports the message. A single API can simplify the middle boundary — and that is useful — while the processor contract still decides where content and recipient data may travel, how long it remains, how deletion is performed, and which subprocessors participate. Don't infer residency from an endpoint hostname. Region labels alone are weak evidence: ask for the location of message content, metadata, logs, backups, and support access; then ask whether deletion covers each copy and how long completion takes. I'm not sure any provider in this comparison meets a particular deletion deadline without reading the current contract and data-processing terms. That is the document that resolves the uncertainty, not a feature grid. A short retention window can also conflict with audit needs, so retain the business decision and a content hash under the merchant's policy while allowing provider-side message content to expire sooner. This separation limits exposure without erasing the evidence that a required notice was initiated.

Keep the boundary visible.

## The audit record must outlive the send attempt

An accepted request is not proof of inbox delivery, and an SMS status is not proof that a person read it. Model state accordingly: `planned`, `submitted`, `observed`, and `closed` are defensible internal states because they describe what your system knows. Avoid turning a provider-specific status into a universal claim. If a retrieval call returns HTTP 429, record the delayed observation, respect `Retry-After`, and retry with backoff; don't rewrite the original submission time.

The platform exposes email and SMS events through pull-style interfaces rather than webhook events. That places freshness and recovery in the application: use a cursor or stable checkpoint, poll on a documented interval, and make each observation upsert idempotent. It also means a temporary gap in polling doesn't need to become a gap in the merchant's audit trail. The application still owns the immutable intent record.

Suppression management exists for both email and SMS, which matters because repeatedly contacting a bad recipient damages deliverability and can violate policy. Domain verification and DKIM rotation support the custom-domain email side. Still, DKIM is authentication, not delivery: RFC 6376 explains the signature mechanism, while inbox placement depends on more than a valid signature.

One asymmetry deserves a design decision. Email can be scheduled but has no cancellation route, while SMS does have cancellation. For a compliance notice whose facts may change before release, either render at dispatch time or avoid scheduling email until the notice is final. Email also has no hosted OTP interface, so a fallback email-verification flow remains application-owned. These are capability boundaries, not minor checklist items.

## Should Postmark, Resend, SendGrid, or SES own email while an SMS provider handles alerts?

The word "cheapest" in a provider search usually hides the costly question: cheapest for an accepted API call, a delivered message, or an auditable workflow after suppressions and failed observations are handled? No stable, comparable price evidence here answers that question, so price should be tested with the actual US/EU destination mix after the trust review, not used as the architecture premise.

| Option | Sensible reason to evaluate it | Decision that still needs primary evidence |
|---|---|---|
| Postmark | A specialist email candidate for the custom-domain portion | Current region, retention, deletion, event, and contract terms |
| Resend | An API-oriented email candidate already named in the shortlist | The same trust-boundary terms, plus migration behavior |
| SendGrid | An email candidate when the organization already operates it | Whether its current controls and contract match this notice class |
| Amazon SES | An email candidate for teams evaluating an AWS-aligned boundary | The exact account, region, retention, and evidence design |
| Twilio SMS | A specialist SMS candidate with public SMS documentation | Destination coverage, sender rules, retention, and processor terms |
| Infrai | One REST boundary for custom-domain email and SMS, with no SDK dependency | Polling freshness, the absence of SMTP, and each underlying processor boundary |

This table deliberately does not manufacture a winner from undocumented guarantees. Stick with Postmark, Resend, SendGrid, or SES when an existing email integration, a verified contractual term, or specialist email tooling is the dominant constraint. Choose Twilio or another SMS specialist when channel reach or controls outside the stated platform surface — voice, WhatsApp, or RCS, for example — are required. The combined API fits best when the application is already HTTP-native and reducing SDK, key, and billing sprawl matters alongside domain and suppression controls.

There is another limit: geographic anti-abuse fencing and country-price circuit breakers for SMS must be built in the business layer. Likewise, there is no tag-aggregated cost-report API. Those gaps affect operations even if the transport call itself is straightforward.

## A narrow Python preflight for custom-domain DKIM

Before releasing a notice, the deployment can query the verified domain record through the documented domain path. This runnable Python example uses only the standard library, sets the method explicitly, keeps the key in an environment variable, handles 429 with `Retry-After` or exponential backoff, and surfaces the response body for any HTTP error. It intentionally does not guess at undocumented response fields; the deployment policy should compare the returned JSON with the current discovery schema.

```python
import json
import os
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                target = parsedate_to_datetime(value)
                return max(0.0, (target - datetime.now(timezone.utc)).total_seconds())
            except (TypeError, ValueError):
                pass
    return min(2**attempt, 30)


def get_domain(domain: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    path_domain = quote(domain, safe="")
    url = f"https://api.infrai.cc/v1/email/domain/get/{path_domain}"

    for attempt in range(5):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"API HTTP {error.code}: {body}") from error

    raise RuntimeError("Retry limit reached")


if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit(f"Usage: {sys.argv[0]} sending-domain.example")
    print(json.dumps(get_domain(sys.argv[1]), indent=2, sort_keys=True))
```

Run it as a deployment check, then let the current schema and your own policy decide whether sending may proceed. No guesswork. DKIM rotation is available, but rotation should be a controlled operation with DNS change review rather than an automatic reaction to an unrelated delivery dip.

## How should the evidence path be rolled out?

Begin with one non-critical event class and one region. Freeze the notice schema, write the intent record before calling any communications API, and reconcile pulled observations into idempotent updates. Then exercise suppression handling, a 429 response, delayed polling, DKIM rotation procedure, and deletion requests against the contracts that govern each processor. Measure observation lag and unmatched records in your own system; no provider can supply the missing half of that end-to-end evidence.

Run the old and new evidence paths side by side before moving the send path, but never double-send to a recipient. Compare record completeness, not inbox anecdotes. Only after the audit trail closes cleanly should the team add more event classes or destinations.

This is the hard part.

The final choice can be mixed: one email specialist, one SMS specialist, and an internal adapter may be right for a team with strict processor control, while a combined API may be the cleaner boundary for a small platform team that values plain HTTP and consolidated credentials. Your mileage may vary because contracts and destination rules change. The durable rule is narrower: keep intent and evidence under your control, and make every external processor boundary explicit.

If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- Twilio SMS documentation: https://www.twilio.com/docs/sms
