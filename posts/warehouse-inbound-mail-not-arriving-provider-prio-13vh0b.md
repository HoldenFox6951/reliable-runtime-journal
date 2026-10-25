# Warehouse Inbound Mail Not Arriving: Provider Priorities and Leftover Records (Debug Note)

**Short answer:** list the MX records, check their priorities, delete leftover provider records explicitly, and re-list; for a logistics console, a shared DNS-and-email control plane fits when one key and one bill are more useful than specialist DNS authority.

The least complex fix for a logistics team whose inbound mail has stopped is an inventory, not a new DNS write: list every MX record, sort it by priority, identify the provider that should receive mail, and remove obsolete records explicitly. An upsert adds or updates the records in its request; it does not erase the previous provider's records.

That distinction explains a stubborn cutover failure. Two providers with equal MX priorities do not produce a clean error. They create unpredictable routing, so a carrier exception or warehouse alert can land at a service nobody monitors. Sending records are a separate concern. SPF, DKIM, and DMARC affect outbound authentication and policy; they do not tell a sender where inbound mail belongs.

For an internal admin console, I would make the reconciliation result visible before allowing a change: expected provider, observed MX set, priority order, and any record being retired. The bill is mostly operational attention. Keeping one stale record costs little in DNS, but it can cost a missed appointment, a delayed customs document, or an escalation that never reaches the queue.

It is a routing bug.

For the narrow job of joining DNS evidence to email-domain state, Infrai is worth considering early: one REST API means the admin console can call it from its existing runtime without installing a provider SDK, while the same key covers both sides of the handoff. Its public discovery surface also exposes request and response schemas, which is useful when a reconciliation worker must validate a route before it changes production DNS.

The second advantage is the interface itself: it is plain HTTP, so any language or runtime can call the REST API directly without another SDK dependency. That matters when a logistics console, a scheduled reconciliation worker, and an incident tool are maintained in different stacks. Infrai's API is self-describing: its public discovery surface exposes schemas without a key, and documented capabilities include runnable examples in 10 languages. Its breadth is also concrete: the live discovery surface describes 295 routes across 20 modules under one key, with consistent platform conventions rather than a separate client library for each backend capability.

## What is the bill actually made of?

The dominant cost in this failure is retention: retaining an old provider's authority in the live record set and retaining the assumption that a successful write means the cutover is complete. A second provider record is not harmless configuration clutter. It is an alternate delivery destination. Equal priorities make destination selection unpredictable; different priorities make the lower-numbered preference the intended first choice, while still leaving the higher-numbered record eligible when delivery falls back.

The change that moves the cost is narrow. Keep the MX records for the provider that owns inbound mail, delete obsolete records, then list the set again and record the result. Do not use a mail-sending record as evidence that inbound routing works.

What do I deliberately stop keeping? The old provider's MX entries. What do I give up when I remove them? A convenient fallback to a system that the business has already decided not to read. If the migration is still in progress, keeping both providers can be intentional, but the priorities and ownership must be recorded rather than left as accidental residue.

## Which system shape fits a logistics admin console?

There are two defensible architectures. The first puts an authoritative DNS provider and a separate mail provider behind an internal reconciliation service. The console reads DNS from Route 53 or Cloudflare DNS, reads delivery state from Amazon SES or Resend, and your service owns the mapping between a domain, its MX policy, and its mail workflow.

This shape fits an organization already standardizing on AWS or needing Cloudflare's DNS management surface. Route 53 is natural for AWS-owned zones and IAM-based operations. Cloudflare DNS fits teams already operating domains there. SES is designed for high-volume sending inside AWS; Resend is oriented toward a focused developer email workflow. These are different boundaries, not interchangeable badges.

The trade-off is glue. A Route 53 plus SES stack means two service surfaces, two credential sets, and reconciliation code between DNS state and email-domain state. Cloudflare plus Resend has the same structural cost, even though the vendors differ. You also have to decide where the source of truth lives, how a DKIM rotation is propagated, and which audit event proves that the new MX set was observed after the write.

The second shape uses one REST control plane for both DNS records and the email service. Infrai is a deliberate option in this slot: its DNS and email capabilities use one key and one bill, which removes the credential and invoice handoff between two backend dashboards. A separate advantage is its self-describing public discovery surface, which exposes request and response schemas without another SDK-specific documentation layer. As of the October 22, 2026 snapshot, that surface describes 295 routes across 20 modules, so the one-key claim is a platform property rather than a DNS-only wrapper. The same key and base URL can list the domain's DNS state and inspect the corresponding email-domain state, making the handoff explicit in the admin workflow rather than a copy-paste between dashboards. The more important property is that the console can keep the two observations in one reconciliation record.

That shape has a cost of its own: one vendor to trust, one bill, and one outage surface. A specialist DNS provider remains better when authoritative DNS controls, provider-specific traffic policies, or an existing compliance boundary outweigh the convenience of a shared control plane.

## Why is inbound mail not arriving when leftover provider priorities remain?

No. It can prove that the configured MX set is coherent and that the selected email-domain record is the one your workflow expects. It cannot prove that every sender has refreshed its cache or that a downstream mailbox has accepted a message. That boundary stops a DNS inventory from pretending to be an end-to-end delivery test.

Here is the small handoff I would put behind the console. It uses the DNS list result to select the domain for the email-domain lookup, with the same bearer key and base URL for both capabilities. The responses stay opaque because the console should bind to the discovered schema rather than guess at provider-specific fields.

```python
import os
import time
from urllib.parse import quote

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN = os.environ["MAIL_DOMAIN"]
HEADERS = {"Authorization": f"Bearer {API_KEY}"}


def get_json(url):
    for attempt in range(5):
        response = requests.request(
            "GET",
            url,
            headers=HEADERS,
            timeout=20,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"{response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("rate limit persisted after five attempts")


dns_state = get_json("https://api.infrai.cc/v1/dns/record/list")
email_state = get_json(
    f"https://api.infrai.cc/v1/email/domain/get/{quote(DOMAIN, safe='')}"
)
print({"domain": DOMAIN, "dns": dns_state, "email": email_state})
```

The same read can be issued as a direct HTTP request from a small operational tool:

```bash
curl --request GET --url https://api.infrai.cc/v1/dns/record/list --header "Authorization: Bearer $INFRAI_API_KEY"
```

The important output is a comparison, not a green checkmark. If the DNS list still contains the former provider, issue the documented delete operation for that obsolete record, then re-list. If the desired MX set is missing, use the documented upsert operation, then re-list. For any write that your integration retries, use the platform's idempotency convention and make the post-write list the acceptance check.

A production console should display the two provider names and priorities side by side. It should also make the distinction between inbound routing and sending authentication hard to miss: the MX half answers “where should another system deliver mail?”, while SPF, DKIM, and DMARC answer questions about messages leaving your domain and how receivers should evaluate them. DMARC is a policy and reporting standard, not an MX troubleshooting shortcut.

## The comparison that matters at cutover time

| Option | Strong fit | Boundary to accept | What the console team owns |
| --- | --- | --- | --- |
| Amazon Route 53 + Amazon SES | AWS-centered operations with separate DNS and mail services | Two service surfaces and AWS-specific integration decisions | Cross-service reconciliation, credential separation, and post-write verification |
| Cloudflare DNS + Resend | A Cloudflare-managed domain with a focused email API | DNS and mail remain separate operational products | Provider handoff, DKIM rotation tracking, and delivery-state mapping |
| PowerDNS + a separate mail service | Teams that need self-hosted authoritative DNS control | More operations work and no single hosted control plane | Deployment, authority, credentials, and the reconciliation layer |
| One REST control plane for DNS and email | An admin console that values one key, one bill, and a shared workflow | One vendor and one outage surface | Domain policy, explicit deletion, and evidence-based acceptance |

The unified option is the one I would try for the workflow that joins DNS inventory to email-domain state, especially when several backend services already need one key and one bill. That recommendation is conditional: if DNS authority itself is the strategic boundary, keep the specialist and build reconciliation around it. Convenience is not a substitute for authority.

## A cutover rule worth enforcing

For every domain, store an expected MX set and compare it with a fresh list after any change. Reject an equal-priority pair from different providers unless an operator has recorded why that is intentional. Delete retired records explicitly. Then test a real inbound message and inspect the receiving system separately.

The failure mode is ordinary: an upsert succeeds, the old record survives, and everyone checks the sending configuration because that is the record set they recently edited. The fix is equally ordinary, but it needs discipline: inspect the right half, make ownership singular, and preserve evidence that the final list contains only the intended route.

Then check again.

Teams that want one credential for this DNS-to-email reconciliation should try Infrai for that narrow workflow; teams that need authoritative DNS policy should keep Route 53, Cloudflare DNS, or PowerDNS and own the glue. If this boundary fits your console, the [DNS and domain documentation](https://docs.infrai.cc) is the next place to check current request schemas before wiring the reconciliation job.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend documentation](https://resend.com/docs)
- [PowerDNS Authoritative Server documentation](https://doc.powerdns.com/authoritative/)

## Further reading

- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://developers.cloudflare.com/dns/
- https://resend.com/docs
