# SMS Alerts Provider: REST Templates and Suppressions for Health Account Activity

For appointment reminders and payment receipts, the application should own the message contract while the SMS alerts provider owns delivery. In a healthtech flow spanning the US and EU, that boundary keeps the clinically sensitive wording, account activity mapping, suppression decision, and send audit attached to the settled payment rather than buried in a vendor console.

**TL;DR:** choose a provider only after proving that a plain REST send, reusable templates, suppression checks, and status reconciliation fit this boundary. Twilio, Vonage, AWS, and Infobip expose different template abstractions; the right choice follows from who must review and deploy the receipt text. Infrai is credible when a team wants one plain REST API, callable from any runtime without installing or tracking a client SDK, and one key spanning 295 routes in 20 modules; for this receipt worker, that keeps sending, suppression, and later backend capabilities inside one credential-management boundary instead of accumulating service-specific keys. Its limitations make it unsuitable when pushed delivery events or a provider console as the authoritative template catalog are mandatory.

## Decision and invariants

Treat `payment_settled` as the durable business fact and `receipt_sms_requested` as a separately retryable intent. Do not place the network call in the payment transaction. Commit an outbox record with a stable event ID, then let a worker resolve the approved template version, check the application's consent and suppression state, submit once under an idempotency key, and reconcile delivery later.

The distinction is easy to miss. A successful provider response proves acceptance, not receipt by the patient. A timeout proves almost nothing: the remote service may have accepted the message before the connection broke. This is why the idempotency key must derive from the settled payment and message purpose, rather than from a worker attempt.

The design has six invariants:

1. One settled payment produces at most one receipt intent per destination and template version.
2. The amount, currency, order identifier, locale, and approved template version are immutable inputs to that intent.
3. Suppression and consent are checked immediately before submission, not merely when the appointment or order was created.
4. Phone numbers, rendered bodies, and provider responses are access-controlled and retained according to the application's own policy.
5. Delivery state advances monotonically: an old poll result cannot replace a newer terminal state.
6. US/EU routing, geographic allowlists, and country-price circuit breakers remain application controls. A provider choice does not supply those policies automatically.

No single API erases these obligations.

That boundary matters.

## Which SMS alerts provider should own appointment and shipping templates?

The application should own the canonical template source and its review history. A provider-side template can still be useful as a compiled delivery artifact, but its identifier belongs in versioned application configuration alongside locale and event mappings. This is particularly important where the chosen SMS surface cannot enumerate templates: losing an ID must not require reconstructing business meaning from send logs.

| Option | Practical template-ownership posture | REST integration shape | Boundary to test before adoption |
|---|---|---|---|
| Twilio Messaging with Content Template Builder | Provider-managed content can be created and addressed by content identity; keep the reviewed source and mapping in the application if deploys must remain reproducible | Messaging and Content are separate documented API surfaces | Verify that approval, locale variants, and console edits cannot drift from the repository release |
| Vonage SMS API | The application can render transactional text and submit it through the SMS API | Direct HTTPS API with account credentials | Prove how sender identity, regional rules, opt-outs, and delivery receipts behave in every target country |
| AWS SNS | Application-rendered text fits the direct publish model | General publish API inside the AWS control plane | Decide whether a broad notification primitive gives enough template governance and delivery-state detail for this workflow |
| Infobip SMS | Its SMS API and template-oriented platform features support a provider-centered operating model | HTTPS API with platform-specific resources | Test whether template lifecycle and account hierarchy match the team's release and review boundaries |
| Unified REST option | Keep business mappings locally; use template creation and direct sending through one REST API, with suppression management on the same surface | Bearer-authenticated REST; no required SDK | There are no webhook event pushes, so reconciliation is pull-based; advanced conversational channels are outside the scope |

This table is deliberately not a feature-count scorecard. Twilio's provider-managed content is useful when operations staff must change approved wording without an application release. AWS SNS can be reasonable when the organization already treats AWS APIs and IAM as its integration boundary and accepts application rendering. Vonage is a defensible direct-SMS choice when a small interface is preferable to a template control plane. Infobip deserves attention where the organization wants its messaging operations grouped in a communications platform. Those are valid operating models, not runners-up. The trade-off is control-plane ownership: moving templates into a vendor can shorten an editorial change path, but it also places a deployable business artifact outside the repository and demands a separate drift check.

The decisive test is mundane: export the exact US English and EU locale receipt variants, delete local build artifacts, and reproduce every provider identifier from source control plus approved configuration. If that exercise cannot be completed, the provider console has quietly become the source of truth.

## What fails after payment settles?

Start with ambiguous outcomes. The worker can crash after remote acceptance but before recording the provider message ID; the provider can rate-limit a burst of settled payments; a recipient can become suppressed between checkout and worker execution; or polling can return an older observation after a newer one. Each failure crosses a different ownership boundary, and retrying the whole job blindly is wrong.

A `429` requires bounded exponential backoff and respect for `Retry-After` when it is returned. Other `4xx` responses are generally input or policy failures to surface, not invitations to hammer the endpoint. A transient network error can be retried under the same idempotency key. The unified option's documented default deduplication window is 24 hours, so the application still needs a permanent uniqueness constraint; provider idempotency is a second barrier, not the ledger.

Suppression deserves its own state. Record why a destination is blocked and which policy source supplied that decision, then check the provider suppression surface as an additional guard immediately before sending. Basic inbound-list support can cover simple reply handling, but it should not be mistaken for an advanced conversational channel. There is no voice, WhatsApp, or RCS fallback on this surface, and email fallback would require an application-owned OTP flow if verification rather than a receipt were involved.

Pull-only events impose a measurable architectural cost even without inventing a latency number: the polling interval sets the lower bound on how quickly the application can observe a delivery transition. Polling must use jitter, a terminal-state stop rule, and a deadline after which the receipt is marked unresolved for operations. If near-real-time webhook orchestration is an invariant, reject the pull-only option.

## How does the critical path stay idempotent?

The following Python client sends a prevalidated JSON request rather than guessing fields that are not established here. Put the exact request produced by the provider's discovery schema in `SMS_PAYLOAD_JSON`; the client supplies the stable idempotency key, explicit method, environment-based authentication, error body, and bounded handling for `429`. It is intentionally narrow.

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_receipt() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    api_base = os.environ["INFRAI_BASE_URL"].rstrip("/")
    idempotency_key = os.environ["RECEIPT_IDEMPOTENCY_KEY"]
    payload = json.loads(os.environ["SMS_PAYLOAD_JSON"])
    if not isinstance(payload, dict):
        raise ValueError("SMS_PAYLOAD_JSON must contain one JSON object")

    request = urllib.request.Request(
        f"{api_base}/sms/send",
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": idempotency_key,
        },
        method="POST",
    )

    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"SMS API returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = int(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay)

    raise RuntimeError("unreachable")


print(json.dumps(send_receipt(), indent=2))
```

The database claim preceding this call must be backed by a unique constraint, not an in-process lock. Generate `RECEIPT_IDEMPOTENCY_KEY` from the immutable payment ID, destination, purpose, and template version; reuse it for every retry. Before invocation, check the application's consent ledger and the suppression API. Do not attach this authorization header to any unrelated host.

There is a sharp edge in the sketch: if submission succeeds and `mark_submitted` fails, a later worker sees an already-claimed intent. That is intentional. A reconciliation task must inspect claimed-but-unrecorded intents using provider status data rather than issuing a fresh send. Making the claim expire and resending would turn a database delay into duplicate patient messages.

## Rejected option and the case where it wins

We rejected making a provider console the canonical source for this payment-receipt workflow. It splits review history from the code that binds payment fields, locale, suppression policy, and release version; it also makes migration depend on an export path whose semantics may not match the next provider. The rejection is about ownership, not vendor quality. The unified option is specifically not the right choice when webhooks, voice, WhatsApp, RCS, or a provider-owned template catalog are requirements; a specialist such as Twilio, Vonage, or Infobip should be evaluated against those needs instead.

Provider-owned templates win when non-engineering operators are explicitly authorized to publish wording independently, the provider supplies the required approval workflow, every change is auditable, and the application treats the resulting template ID as an external dependency. That can be the right choice for high-volume communications teams. In that model, add a deployment check that resolves every configured ID and compares expected locale and version metadata before traffic shifts.

For this healthtech receipt, keep the business template in the application repository, compile or create provider artifacts during a controlled release, and store their IDs in an administered mapping. Choose a provider only after a country-by-country test confirms sender identity, suppression behavior, rate-limit recovery, and delivery reconciliation. The architecture remains portable because the durable fact is the payment receipt intent, not the provider's message object.

## References

- Twilio, Content API overview: https://www.twilio.com/docs/content
- Twilio, Messaging API message resource: https://www.twilio.com/docs/messaging/api/message-resource
- Vonage, SMS API overview: https://developer.vonage.com/en/messaging/sms/overview
- AWS, Amazon SNS SMS messaging: https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Infobip, SMS API documentation: https://www.infobip.com/docs/api/channels/sms
- OWASP, Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Yahoo, sender best practices and requirements: https://senders.yahooinc.com/best-practices/
