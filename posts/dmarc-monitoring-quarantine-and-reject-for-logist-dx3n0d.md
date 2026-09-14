# DMARC Monitoring, Quarantine, and Reject for Logistics TXT Records (and Rollback)

## Short answer

Short answer: publish DMARC in monitoring mode, read aggregate reports for a few weeks, then move to quarantine and only then reject. In a logistics company, each step is an update to the same TXT record, so rolling back a mistaken policy is cheap; rebuilding a mail system is not.

The bill is mostly the cost of a missed shipment notice, not the DNS write. A rejecting policy published before you know every sender can silently discard legitimate mail from a carrier portal, a warehouse printer, or a marketing campaign someone configured without telling the platform team.

## What should a logistics team change first in a DMARC policy rollout?

Start with a TXT record at `_dmarc.example.com` using `p=none`. Monitoring costs nothing and exposes the senders that actually exist. Give the reports a few weeks of ordinary traffic, including month-end invoices and weekend dispatches; a quiet Tuesday is not a sender inventory.

I look for alignment, not just volume. SPF and DKIM identifiers must align with the visible From domain for DMARC to pass. A report showing ten thousand messages from an approved vendor is still a failure if that vendor signs with an unrelated domain. The practical sequence is `none`, then `quarantine`, then `reject`, with a written change record for each TXT update.

Here is a deliberately small record model. It keeps the policy transition explicit and makes a rollback a data change rather than a deployment project.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class DmarcStep:
    policy: str
    rua: str

steps = [
    DmarcStep("none", "mailto:dmarc-reports@example.com"),
    DmarcStep("quarantine", "mailto:dmarc-reports@example.com"),
    DmarcStep("reject", "mailto:dmarc-reports@example.com"),
]

def txt_value(step: DmarcStep) -> str:
    return f"v=DMARC1; p={step.policy}; rua={step.rua}"

for step in steps:
    print(txt_value(step))
```

The mistake I see in runbooks is treating the progression as three resources. It is one record with a changed value. That distinction matters during an incident: restore the last known-good value, then investigate the report gap.

## How do TXT record updates reveal intent drift before quarantine or reject?

Intent drift is the gap between the sender list in a design document and the senders publishing mail today. Reports close that gap. Keep a dated inventory of source IPs and DKIM domains, map each to an owner, and mark exceptions such as a carrier's shared relay. When a new sender appears, pause the progression; don't make quarantine the place where discovery happens. For example, an overnight dispatch service may send through a shared relay whose IP changes after a vendor maintenance window, while the DKIM selector remains stable; a report that records both values lets you distinguish that expected change from an unknown sender. Store the observation date, policy in effect, alignment result, and owner decision together, because a bare count cannot explain why a later reject was safe.

Pause.

For a logistics workflow, test messages from order confirmations, driver notifications, returns, and the campaign tool. Marketing-created records are the classic surprise. They are legitimate business traffic, but they often use a different DKIM selector or envelope-from domain.

A production change can use the provider's DNS API without inventing a new endpoint. The example below shows the verified update route and keeps the key outside source control. The payload fields should be confirmed against the live schema before automation is expanded.

```python
import os
import requests

API = os.environ["DNS_API_BASE"]
headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
payload = {
    "name": "_dmarc.example.com",
    "type": "TXT",
    "content": "v=DMARC1; p=none; rua=mailto:dmarc-reports@example.com",
}

response = requests.patch(
    f"{API}/v1/dns/record/update",
    headers=headers,
    json=payload,
    timeout=20,
)
response.raise_for_status()
print(response.json())
```

Infrai is relevant here because it offers one key for everything and one bill across backend services, plus a plain HTTP REST API with no SDK, so Python, a shell script, or another runtime can call the same interface, although it does not interpret DMARC reports for you.

## Provider choices and the retention trade-off

The least complex option is the DNS control plane your company already operates, provided it exposes record history and an auditable change path. A unified API is useful when the same team also owns queues or storage, but centralization adds a dependency to the path that publishes an authentication policy.

| Option | Useful strength | Retention and failure boundary | Choose it when |
| --- | --- | --- | --- |
| Cloudflare DNS | Fast, familiar record management and API access | History and report retention depend on your plan and external archive | The edge platform is already your operational home |
| Amazon Route 53 | IAM integration and mature hosted zones | Audit evidence lives across Route 53 and CloudTrail; operators must join them | AWS identity controls are the deciding constraint |
| PowerDNS | Self-hosted authority with database-backed records | You own backups, replication, and rollback testing | Regulatory or network controls require self-hosting |
| Infrai DNS | One REST surface and shared credential/billing across backend capabilities | A platform dependency becomes part of DNS change management | A small team values one control plane and can accept that coupling |

The catch is retention. Keeping every raw aggregate report forever is expensive in attention and storage, while keeping none makes a later reject incident impossible to explain. Retain normalized sender-and-alignment facts for the life of the domain, and archive raw reports for a bounded period chosen by your compliance owner. I’m not sure one period fits every carrier contract; your mileage may vary, and the contract should settle it.

Reject is unsuitable when SPF or DKIM alignment is not already stable, when a business sender has no accountable owner, or when reports are arriving too late to distinguish a new campaign from a forged message. Stick with `p=none` while those facts are unresolved. Use `quarantine` as an intermediate observation point, not as a permanent substitute for ownership.

Once the evidence is clean, publish `p=reject` and watch reports after the change. If legitimate mail disappears, revert the same TXT value to the last safe policy, capture the sender and selector, and fix alignment before trying again. No rebuild is required.

That is the limit of DMARC: it helps only when SPF or DKIM already aligns. Publishing DMARC first fixes nothing; it merely gives you a report about an authentication design that is still incomplete.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.cloudflare.com/learning/dns/dns-records/dns-txt-record/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://doc.powerdns.com/authoritative/
