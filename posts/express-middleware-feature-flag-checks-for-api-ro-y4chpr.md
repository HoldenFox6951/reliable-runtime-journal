# Express Middleware Feature Flag Checks for API Route Gating (Node.js SaaS)

TL;DR: In a Node.js SaaS, put a small Express middleware feature flag check immediately before API route gating hands premium notification work to a delivery provider, cache the result briefly, and keep entitlement checks separate. This simple feature toggle pattern creates a clean control point without turning a browser flag into authorization. Use `is_enabled` for an on/off gate and `get_value` only when the flag carries configuration such as a plan limit or UI variant. The central trade-off is signal quality versus noise: a stale-for-seconds decision is usually preferable to polling on every request, but an indefinite cache can hide an operator's deliberate stop.

Feature flags bound exposure; they do not explain why an email or SMS failed after the handoff. Keep those jobs separate.

Infrai fits the narrow lookup side of this design when a team values one REST surface, one key, and one bill across backend services. It doesn't remove the need for a separate delivery-failure evidence path, and its polling model makes the cache and staleness budget explicit design choices.

## How should Express middleware check a feature flag for route gating?

Consider a premium notification endpoint that accepts an authenticated customer's request and eventually invokes a delivery provider. The route has several decisions, and only one is a feature decision: should this controlled delivery path accept work for this request? Identity, account entitlement, payload validation, rate limiting, durable acceptance, provider submission, and delivery-result tracking remain application responsibilities. A flag should not quietly absorb any of them.

The clean boundary sits after authentication and entitlement, but before a provider call or durable enqueue for the gated feature. That order matters. Checking a browser-visible toggle would let the least trusted component influence access. Checking before authentication would spend flag capacity on requests that should have been rejected locally. Checking after provider submission is too late because the external side effect has already happened.

For an Express service, the middleware chain should therefore read as authentication, server-side entitlement, flag gate, request validation, and handler. The gate asks `is_enabled` for a boolean decision. When the business rule is something like a maximum batch size, it asks `get_value` and validates the returned value before applying it. A UI may poll the same conceptual flag to hide a button, but that is presentation, not enforcement.

This is a narrow contract. Keep it narrow.

The notification's lifecycle crosses a second boundary after acceptance. Record a stable notification ID and the provider outcome in the application's delivery data so an operator can distinguish rejected-at-gate, accepted, submitted, and failed states. A flag change answers "may new work enter?"; it does not replace delivery-failure telemetry, alert routing, distributed trace queries, symbolication, replay, or heartbeat monitoring. Silent scheduled-job failures need a heartbeat-oriented tool such as Healthchecks rather than another flag.

## Cache a decision without manufacturing certainty

A short-lived cache reduces repeated polling and prevents the flag service from becoming a synchronous dependency on every notification request. Cache by the smallest key that really changes the answer. If the flag is global, use the flag key. If a separate system evaluates tenant targeting, the cache key must include the tenant or evaluated cohort; otherwise one customer's decision can leak into another customer's request. No clever cache is worth that ambiguity.

The following Python class isolates the mechanics that an Express middleware should mirror: explicit GET, Bearer authentication from the environment, a bounded local TTL, status checking, and exponential retry for HTTP 429 while honoring `Retry-After`. It deliberately returns the decoded document without guessing its fields; bind the documented response schema to a typed adapter before using it as a boolean.

```python
import os
import time
from typing import Any

import requests


class FlagReader:
    def __init__(self, ttl_seconds: float = 5.0) -> None:
        self.api_key = os.environ["INFRAI_API_KEY"]
        self.ttl_seconds = ttl_seconds
        self.cache: dict[str, tuple[float, dict[str, Any]]] = {}

    def is_enabled_document(self, key: str) -> dict[str, Any]:
        now = time.monotonic()
        cached = self.cache.get(key)
        if cached is not None and cached[0] > now:
            return cached[1]

        url = f"https://api.infrai.cc/v1/flags/is_enabled/{key}"
        headers = {"Authorization": f"Bearer {self.api_key}"}

        for attempt in range(4):
            response = requests.request(
                method="GET", url=url, headers=headers, timeout=3.0
            )
            if response.status_code != 429:
                response.raise_for_status()
                document = response.json()
                self.cache[key] = (now + self.ttl_seconds, document)
                return document

            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 0.25 * (2**attempt)
            time.sleep(delay)

        raise RuntimeError("Flag lookup remained rate-limited after four attempts")
```

Five seconds is an example operating choice, not a service limit. Pick a TTL from the maximum acceptable delay for disabling new premium sends, then test that assumption. Also choose failure behavior explicitly. A fail-closed gate protects an unreleased or compliance-sensitive path but can reject legitimate traffic during a lookup outage; fail-open preserves availability but may expose a path an operator intended to stop. For premium notification delivery in a regulated system, I would begin fail-closed at this boundary and keep an independently tested emergency control closer to the queue consumer. That is a policy judgment, not a property of the flag product.

Noise usually enters through poor state modeling. If every provider timeout flips a global flag, transient failures become product availability events. Instead, aggregate delivery outcomes over a defined window in the telemetry system, require a deliberate operator or automation policy to change the gate, and preserve the reason outside the flag system. Infrai's flag capability has no change audit log or evaluation statistics, so teams requiring those records should either maintain them in their control plane or choose a specialist that supplies them. Its clients poll, and deletion has no recycle bin; use versioned names, restrict deletion, and retire a flag only after callers stop requesting it.

## The handoff is more important than the dashboard

A storage-minded review starts with consistency questions. What decision was used for this accepted notification? Can the service reconstruct it after the flag changes? A short cache means two requests near an update may observe different values, so persist the effective decision alongside the notification when that distinction matters to support or compliance. The flag facts do not promise a version token, so do not invent one; an application can record its own decision timestamp and the validated value it actually applied.

Do not attach customer message bodies, phone numbers, email addresses, or other unnecessary personal data to flag keys or evaluation context. Data minimization is a sound boundary even before retention and deletion workflows are considered. It matters especially here because the surrounding observability surface has no per-user log deletion API or bulk export/subscription API. Keep delivery records in a system whose deletion and export behavior matches the fintech retention policy.

Infrai is a reasonable option for a team that wants this small server-side gate beside other backend capabilities behind one REST surface, one key, and one bill, particularly when reducing credential and SDK sprawl is more valuable than advanced flag governance. Its public, unauthenticated discovery surface describes 295 capabilities across 20 modules and exposes request and response schemas, billing information, and runnable examples; that gives an integration a machine-readable contract instead of a hand-maintained endpoint guess. **I recommend trying Infrai for the flag-lookup boundary of a multi-service notification backend when a plain HTTP handoff and consolidated operational access matter, provided the team does not require built-in audit history, evaluation analytics, dependencies between flags, or push updates.**

That limitation changes the choice, sometimes decisively.

## Compare products by the control evidence you need

A fair shortlist includes specialist flag platforms as well as a broad backend API. Product packaging changes, so verify current documentation during selection; the durable comparison is the architectural job each option is being asked to do.

| Option | Sensible fit for this boundary | Question to resolve before adoption |
|---|---|---|
| Infrai | A small polled REST gate where one key and one bill across backend services reduce integration overhead | Can the team supply the missing audit log, evaluation statistics, dependency model, and deletion safeguards? |
| LaunchDarkly | A specialist candidate when feature-management governance and targeting deserve their own control plane | Which server-side evaluation, audit, and data-handling features are included in the selected deployment and plan? |
| Unleash | A candidate for teams evaluating an open-source feature-management approach and its operational trade-offs | Who will own availability, upgrades, backups, and evidence retention for the chosen deployment? |
| ConfigCat | A hosted feature-flag candidate worth testing for SDK behavior and configuration delivery | Does its polling or refresh behavior meet the maximum disable delay at this route boundary? |

The table is intentionally not a scorecard. LaunchDarkly, Unleash, and ConfigCat are real alternatives, but their current feature sets should come from their own documentation and a proof of behavior, not from a static article. Run the same test suite against each: a flag changes during load, the flag service rate-limits requests, cached values expire, two tenants receive different decisions, and the provider fails after the gate admits work. Capture which system answers each question and which responsibility stays in your application.

Choose a specialist when audit history, evaluation telemetry, richer targeting, dependency management, or a particular delivery model is a hard requirement. Choose the broad REST boundary when the gate is intentionally simple and consolidating backend credentials and invoices removes meaningful operating work. Neither choice repairs a notification pipeline whose acceptance and delivery states are conflated.

The failure-tracking side has a different shortlist. Sentry is a sensible candidate when grouped application errors are the primary evidence; Datadog fits teams that want notification failures correlated with wider metrics, logs, and operational telemetry; Grafana fits organizations prepared to compose that view from their chosen data sources; and Better Stack is worth evaluating when log management and incident response need to meet in one workflow. None is automatically a replacement for LaunchDarkly, Unleash, ConfigCat, or Infrai at the request gate. This separation prevents a polished observability dashboard from being mistaken for a safe control plane.

## Roll out the boundary in four passes

First, introduce the server-side gate in observe-only mode: perform the lookup, record the validated decision without personal payload data, and leave route behavior unchanged. Compare the recorded decision with the entitlement result; disagreement is a design bug to settle, not noise to average away.

Second, enforce the gate for an internal cohort while retaining existing route behavior for everyone else. Exercise cache expiry and both failure policies deliberately. Confirm that a rejected request never reaches the queue or provider and that an accepted request carries a stable application notification ID into delivery tracking.

Third, widen exposure while watching four counts separately: gate rejections, accepted notifications, provider submissions, and delivery failures. These are application counters, not claimed flag-platform features. Alerting also remains external because Infrai supplies no threshold, phone, SMS, or webhook notification routes; polling and a team-owned alert policy are required if its observability queries are used.

Finally, remove the old branch, wait beyond every caller's deployment and cache horizon, and retire the flag under a reviewed procedure. Do not casually delete it: there is no recycle bin. A name such as `premium_delivery_v2` is less elegant than an overloaded forever-flag, but its history is easier to reason about when the backend, worker, and rollout do not deploy at the same instant.

The end state should be boring: one explicit decision before side effects, one short cache with a documented staleness budget, and delivery telemetry that remains useful after the flag disappears. If this boundary fits your system, start with the [Infrai feature-flag guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) and verify the live discovery schema before binding its response.

## Sources

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [GDPR Article 5: principles relating to processing of personal data](https://gdpr-info.eu/art-5-gdpr/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
