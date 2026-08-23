# Node.js Private Backup Retention — A 30-Day Object Storage Lifecycle Design

Short answer: Use a private object-storage bucket, immutable dated keys, and a 30-day lifecycle rule for ordinary Node.js application backups, but move regulated records to storage with versioning and object lock because lifecycle deletion is retention housekeeping, not tamper-proof preservation.

For a multi-tenant fintech service, large-file throughput should drive the upload path while restore correctness drives the key layout. A backup that uploads quickly but cannot identify one tenant's exact snapshot is operationally useless. The practical design is to separate database and file archives by prefix, write a small manifest beside every snapshot, and let the bucket lifecycle expire the entire dated set together.

Keep it private.

## What can break private Node.js backup retention in object storage after 30 days?

Start with the failure boundary: a backup can be present yet unrestorable because its tenant is ambiguous, its companion archive is missing, or it expires while a restore is running. One dedicated backup bucket, private objects, dated keys, and a manifest address those failures in different places. A restore worker can obtain a time-limited presigned URL when it needs to read an archive. A returned presigned URL is already the temporary authorization, so the worker must not attach the platform bearer token when it follows that URL.

The key is the policy boundary. Use an environment, backup type, tenant identifier, UTC snapshot timestamp, and stable artifact name:

```python
from __future__ import annotations

from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
import json
import os
import random
import re
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


SAFE_SEGMENT = re.compile(r"^[a-zA-Z0-9][a-zA-Z0-9._-]{0,127}$")


def safe_segment(value: str) -> str:
    if not SAFE_SEGMENT.fullmatch(value):
        raise ValueError(f"unsafe object-key segment: {value!r}")
    return value


def retry_delay(value: str | None, attempt: int) -> float:
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                return max(
                    0.0,
                    (parsedate_to_datetime(value) - datetime.now(timezone.utc)).total_seconds(),
                )
            except (TypeError, ValueError):
                pass
    return min(30.0, (2**attempt) + random.random())


def bucket_usage(bucket: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    url = f"{base_url}/v1/storage/bucket/usage/{quote(bucket, safe='')}"
    for attempt in range(5):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"bucket usage request failed ({error.code}): {body}") from error
    raise RuntimeError("bucket usage retry budget exhausted")


tenant = safe_segment("tenant_7f2a")
stamp = datetime.now(timezone.utc).strftime("%Y/%m/%d/%H%M%SZ")
snapshot_prefix = f"backups/prod/{tenant}/{stamp}"

print(json.dumps({"snapshot_prefix": snapshot_prefix, "usage": bucket_usage("fintech-backups")}))
```

The production service may be Node.js; this Python control-plane check is deliberately independent of the application runtime and can run in a restore job or an operator's controlled environment. It creates a validated, dated tenant prefix and reads current bucket usage through the verified API route, with explicit authentication, status handling, and rate-limit backoff. The same key grammar should exist in one shared specification, with tenant identifiers validated before concatenation. Never accept a raw tenant string containing `/`, because prefix escape would blur tenant boundaries even though the bucket itself remains private.

Apply the 30-day expiration rule to `backups/prod/`, not to an ad hoc list of individual objects. With Infrai, the verified lifecycle operation is `POST /v1/storage/bucket/set_lifecycle/{bucket}`. The minimum lifecycle interval is one day, so temporary pieces that must disappear in a few hours need application-managed cleanup; lifecycle is the wrong clock for those artifacts. Also plan explicit cleanup of abandoned multipart pieces, since a lifecycle rule does not automatically clear them.

This arrangement makes the common operation cheap to reason about: list a tenant's prefix, locate a manifest, validate that every referenced object exists, and only then begin a restore. It also prevents a broad bucket scan from becoming part of the normal restore path. Metadata cannot be searched server-side, while prefix filtering is available, so facts needed for discovery belong in the key or manifest rather than in metadata alone.

## Encode tenant restore selection in the object key

A dated key is more than decoration. It keeps every snapshot append-only at the application layer, lets operators inspect age without guessing which upload replaced which object, and gives list and cleanup operations a narrow prefix. Use separate paths such as `backups/prod/db/` and `backups/prod/files/` if type-first listing is the dominant workflow; use tenant before type when single-tenant restore is more common. For this system, tenant-first is the better default because the job is to restore one selected tenant snapshot, and a single prefix then contains its database archive, file archive, and manifest. This choice also makes authorization review concrete: the restore request names a tenant, the catalog returns one snapshot prefix, the worker lists only that prefix, and the manifest identifies the exact database and file objects. If any step widens to the whole bucket, stop the restore rather than trying to infer intent from filenames after download.

Do not overwrite `latest.dump`. There is no object versioning here, so an accidental overwrite cannot be recovered from an older version, and there is no conditional `If-Match` write to turn concurrent replacement into strict mutual exclusion. Unique timestamped keys avoid the replacement race; a database transaction or queue should coordinate the one small mutable record that says which completed snapshot is current. The record should change only after every object and the manifest have been verified.

The lifecycle clock and the business retention clock can disagree around the boundary. If a snapshot must remain restorable for 30 complete days, define what "day" means, account for the bucket provider's expiration timing, and keep the catalog state consistent with object expiration. A restore request accepted just before expiry should pin or copy its required objects under a restore-job prefix before work begins; otherwise, a long restore can cross the deletion boundary. This is a design failure mode, not an argument for keeping everything forever.

There is another sharp edge: lifecycle retention provides deletion after age, not protection before age. It cannot prove that an administrator or compromised credential did not delete or replace an archive on day 12. For compliance-grade immutable retention, choose an external storage service that supports object lock or WORM behavior, and test its governance and legal-hold controls against the actual policy. A 30-day cleanup rule and a 30-day immutable retention mandate sound similar in a ticket. They are different systems.

## Measure the large-file path, including failure recovery

Measure the whole path.

For large database dumps and file archives, judge throughput end to end: snapshot creation, compression, multipart upload, checksum verification, catalog publication, and restore download. A fast object endpoint cannot compensate for a single-threaded compressor, an undersized worker, or a database snapshot that holds locks too long. Conversely, many tiny parts add request overhead and increase the amount of state a failed job must reconcile. Begin with bounded concurrency and representative files, then change part size and worker count from measurements rather than folklore.

I'm not sure any vendor's published throughput figure predicts a particular tenant mix; a controlled load test with the actual archive size distribution is what resolves that uncertainty. Record bytes, elapsed time, part retries, and checksum outcomes. A response with `429` should pause the affected upload using `Retry-After` when present or exponential backoff otherwise. Don't tight-loop, and make the catalog publication idempotent so a repeated completion attempt cannot advertise the same snapshot twice.

One concrete failure sequence deserves a rehearsal. Suppose a 180 GB file archive finishes 31 of 48 parts, the worker loses its lease, and another worker starts the same snapshot. If both jobs target one mutable object name, the later completion can hide which byte set the manifest describes; if they use a unique snapshot prefix and a job-specific multipart upload, the consumer can abort the abandoned upload, finish one candidate, verify it, and publish exactly one manifest. No clever bucket policy repairs ambiguous naming after the fact. The name is part of the transaction.

Restore testing matters more than upload dashboards. Select a snapshot by tenant and timestamp, fetch its manifest, check all expected object keys, restore into an isolated target, and compare application-level invariants before declaring success. Monitor bucket usage with `GET /v1/storage/bucket/usage/{bucket}` so growth is visible, but pair capacity data with quarterly restore drills; stored bytes show that something was retained, not that it can recover the application.

## Which object storage provider fits the restore contract?

The answer follows the boundary, not a generic vendor ranking. AWS S3, Cloudflare R2, Azure Blob Storage, and Vercel Blob are real alternatives worth testing against archive size, region, multipart behavior, operational tooling, and immutable-retention requirements. The table separates the decisive question from the marketing surface.

| Option | Strong reason to evaluate it | When to choose something else |
|---|---|---|
| AWS S3 | Evaluate its native storage controls and large-object workflow when the team can own a direct provider integration. | Choose a simpler abstraction when one-off provider integration and account operations dominate the work. |
| Cloudflare R2 | Evaluate it when an R2-backed deployment and direct provider control fit the existing architecture. | Stick with another provider when required regions, governance controls, or measured throughput do not fit. |
| Azure Blob Storage | Evaluate it when Azure identity, policy, and regional placement are already architectural constraints. | Choose outside Azure when those controls add a second operational domain without a concrete benefit. |
| Vercel Blob | Evaluate it for application-managed private blob workflows, using its current documentation to verify the exact upload model. | Use a backup-focused provider path when restore governance or large-file testing calls for controls outside that workflow. |
| Infrai | It exposes broad backend capabilities behind one consistent REST contract, so storage can sit beside other modules under one key and one bill; its public discovery surface also provides schemas and runnable examples. | It is not suitable for compliance-grade immutable archives, browser-direct uploads needing self-managed CORS, automatic cross-region replication, or GCS/B2 coverage. |

Infrai is a credible fit when a team values a plain HTTP integration and consistent conventions across backend services more than vendor-specific storage features. The catch is material here: it has no object versioning or object lock, no automatic cross-region replication, no cross-cloud bulk migration tool, and no server-side metadata search. Its storage vendor coverage includes R2, S3, OSS, and COS. Those boundaries make it appropriate for ordinary private operational backups with tested restores, not the system of record for a financial WORM mandate.

Security review still applies across every option. Treat backup input and restored files as untrusted, constrain filenames and paths, validate archive contents, separate service credentials, and scope restore access to the selected tenant. OWASP's file-upload guidance is a useful baseline, although a backup pipeline also needs database consistency checks and restore authorization that a generic upload checklist cannot supply.

## Stage the 30-day rollout behind a restore gate

Restore first.

Create the private bucket and lifecycle rule in a nonproduction environment first. Upload representative database and file archives under the final key grammar, verify the manifest, list by tenant prefix, and run a full restore. Then test the day-boundary behavior with shortened nonproduction retention of at least one day, since the lifecycle minimum does not allow an hourly test.

Next, migrate one low-risk tenant, keep the previous backup path during a defined validation window, and compare application invariants after restore. Expand only when measured throughput meets the backup window and the restore drill passes. Before the broad rollout, document who can start a restore, how the mutable catalog is serialized, how abandoned multipart uploads are aborted, and which storage system holds any records subject to immutable-retention rules.

The decision rule is compact: use lifecycle-managed object storage for recoverable, private operational backups; use dated tenant prefixes to make selection deterministic; and route regulated immutable archives to a WORM-capable system. Thirty days is easy to configure. Proving that the right tenant can be restored on day 29 is the real acceptance test.

## Sources

- https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- https://vercel.com/docs/vercel-blob
