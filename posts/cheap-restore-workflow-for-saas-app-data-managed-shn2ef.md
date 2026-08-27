# Cheap Restore Workflow for SaaS App Data: Managed Database Backups or Object Storage

For ordinary SaaS recovery, make the database service's snapshot and point-in-time mechanism the primary path, and add independently governed object-storage exports only when portability or a separate administrative boundary is a stated requirement. The deciding constraint is restore ownership: a stored backup has little value until an authorized operator can turn it into a quarantined, validated database within a measured recovery objective.

This is an architecture decision record, not a storage-price comparison. A small retained-byte price can conceal export workers, key management, catalog maintenance, target provisioning, import time, index construction, and the human time needed to prove that a restore is usable. The simplest credible workflow is usually the one with the fewest team-owned transitions, provided its control plane is inside an acceptable failure boundary.

## Decision: define the recovery invariants before choosing a medium

The decision is to use the managed database recovery path for routine operator mistakes and to maintain logical exports in object storage only for a named independence, portability, or retention need. The application should declare four invariants: a recovery point with a known consistency boundary; a new target that stays isolated from application traffic; integrity and business validation before promotion; and a durable operation identity so a retry cannot promote twice.

Those invariants expose the failure modes that tend to get hidden by the word "backup." A usable set of bytes can still be inaccessible after credential loss. A snapshot can be restored successfully while an incompatible schema migration prevents the application from reading it. A logical dump can be complete as a file yet omit writes outside its declared boundary. Encryption keys, the backup catalog, and production administration can share one authority, turning separate copies into one correlated loss.

Keep the operational evidence separate from the target being restored. Record the selected recovery point, manifest identifier, target identifier, approving identity, validation result, and elapsed time. A recovery status endpoint or manifest viewer should not be cacheable by intermediaries; MDN documents that `Cache-Control: no-store` directs caches not to store a response. It does not replace authorization, and it should not be mistaken for a data-protection policy.

For regulated workloads, contingency planning needs to account for data availability, access control, and tested procedures rather than treating retention as the whole control. NIST SP 800-66 Rev. 2 is useful context for applying the HIPAA Security Rule, while the exact safeguards still depend on the organization and the data it handles.

## Should a SaaS app use object storage or managed database backups for PostgreSQL and MySQL?

Use managed backups when the required restore stays in the same database environment and the priority is a short, provider-operated recovery sequence. Use object storage for exported PostgreSQL or MySQL backup artifacts when recovery must cross environments, retention must be independently governed, or an inspectable portable format is required. These are different promises, so a layered design is often more honest than declaring a universal winner.

| Decision axis | Managed database recovery | Logical exports in object storage |
| --- | --- | --- |
| Restore authority | The database control plane performs more of the state transition | The application team owns catalog lookup, retrieval, decryption, import, and validation |
| Consistency boundary | Defined by the database recovery mechanism | Must be created and recorded by the export process |
| Portability | Usually tied to the source database environment | Can be imported into a deliberately prepared compatible environment |
| Failure isolation | May share production identities and administration | Can use a separately governed account, key policy, and retention policy |
| Time to usable service | Often shorter for same-environment recovery | Depends on transfer, import, index build, and validation |
| Operating cost | Includes the service's recovery capability and rehearsal capacity | Includes retained data, requests, retrieval, transfer, workers, and rehearsal infrastructure |

The catch is that managed recovery can be a poor fit when the recovery scenario must survive loss of the same provider account or administrative plane that runs production. Exports alone are also a poor default for a small team that has no tested import automation: they move every important transition into code and a runbook the team must own indefinitely. Price can inform a decision, but it cannot establish recovery time, consistency, or authority separation. Those claims require a drill.

## How should the restore workflow make retries safe?

Treat restore as a state machine, not as a command pasted into an incident channel. The runner consumes an immutable manifest, verifies each artifact before import, provisions a fresh target, validates structural and application-level checks, and promotes once. The state record must live outside the target database, because replacing the target must not erase the evidence that controls a retry.

Don't skip the states.

The code below only validates a local manifest and artifact digests. It deliberately leaves the database importer outside the example, since PostgreSQL and MySQL dumps differ in privileges, extensions, roles, character settings, and restore ordering; a generic one-line import command would obscure the boundary that needs review.

```python
from __future__ import annotations

import hashlib
import json
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class Artifact:
    path: str
    sha256: str


def load_manifest(path: Path) -> tuple[str, list[Artifact]]:
    document = json.loads(path.read_text(encoding="utf-8"))
    operation_id = str(document["operation_id"])
    artifacts = [Artifact(**item) for item in document["artifacts"]]
    return operation_id, artifacts


def verify_artifact(root: Path, artifact: Artifact) -> None:
    payload = (root / artifact.path).read_bytes()
    digest = hashlib.sha256(payload).hexdigest()
    if digest != artifact.sha256:
        raise ValueError(f"digest mismatch for {artifact.path}")


def prepare_restore(manifest_path: Path, artifact_root: Path) -> str:
    operation_id, artifacts = load_manifest(manifest_path)
    for artifact in artifacts:
        verify_artifact(artifact_root, artifact)
    return operation_id
```

In production, persist states such as `verified`, `imported`, `validated`, and `promoted` against that operation identifier. A second request with the same identifier should return the recorded state rather than repeat promotion. Validation needs more than checksums: test tenant counts, foreign-key relationships, a bounded sample of recent business events, and domain rules that row totals cannot express. If validation fails, keep the target quarantined and preserve the diagnostic record without exposing decrypted rows or secrets.

## What does a recovery rehearsal need to measure?

Start from documented access rather than an engineer's existing shell session. Measure queue delay, target provisioning, transfer, import, index construction, validation, and promotion separately. A single duration cannot reveal which stage violates the recovery objective, and a successful checksum proves only that the intended artifact arrived.

Test the whole path.

An end-to-end rehearsal belongs after a meaningful database, schema, encryption, retention, or backup-tooling change, and it starts before any restore command: confirm that the operator can discover the right recovery point without relying on a private bookmark, that the required key authority is available under the incident role, that the selected source and target are unmistakably named, and that the target's network and credentials keep it out of production traffic until promotion. During the run, capture each stage separately, then compare the observed durations with the stated recovery objective and inspect the slowest stage rather than averaging it away. A catalog check can show that a manifest exists; a digest check can show that bytes were not altered; neither establishes that the database engine can apply the bytes, that the application can use the restored schema, or that the operator can complete the promotion under the permissions available during a real incident. Those are different claims, and collapsing them into a green backup status is how recovery plans gain confidence they have not earned.

Keep lighter catalog and checksum checks between full drills. The limitation is real: production-scale rehearsals consume capacity and staff time. A team unable to support them should reduce the number of team-owned restore stages or select a narrower recovery objective; it shouldn't call a metadata check a restore test.

## Rejected default and valid use cases

The rejected default is exports-only recovery for every SaaS application. It has a valid use case when cross-environment portability is the primary invariant and the team already operates import, validation, and promotion automation against a separately provisioned target. Managed recovery alone is also a valid stopping point when same-environment restoration meets the risk model and the organization accepts the correlated administrative boundary.

The decision should be reviewed when recovery objectives, tenant isolation, database engines, legal retention, or account boundaries change. The acceptance test remains plain: an authorized operator selects a declared recovery point, restores into quarantine, proves integrity and application behavior, and promotes exactly once within the measured objective.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://csrc.nist.gov/pubs/sp/800/66/r2/final
