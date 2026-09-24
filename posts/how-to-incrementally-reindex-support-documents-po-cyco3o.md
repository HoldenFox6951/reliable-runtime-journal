# How to Incrementally Reindex Support Documents: Postgres Delete-Then-Upsert Semantics

Update a customer-support index as a versioned replacement: detect a changed source document, build its complete new chunk set, delete the old chunks by stable document ID, and insert the replacement chunks in one transaction. The deciding constraint is freshness without mixed versions; an incremental job that leaves one obsolete paragraph searchable is wrong even if every new paragraph arrived.

TL;DR: give each source document an immutable `document_id`, hash normalized content, and skip it when that hash is already active. For a changed document, chunk outside the transaction, then lock its manifest row, reject stale events, delete every row for that `document_id`, insert the complete new set, and update the manifest before committing. Generate embeddings before this short publish transaction.

## How should incremental reindexing delete and upsert changed documents?

Three invariants control the design. Search exposes chunks from one source version, deletion makes every chunk of that source ineligible for retrieval, and replaying an ingestion event creates no duplicates. These requirements are stronger than "upsert each chunk" because chunk boundaries move: adding a warning near the top of a troubleshooting page can renumber every later chunk.

Use two levels of identity. `document_id` identifies the durable article; `chunk_id` identifies one chunk within one content version. Derive the latter from the document ID, source hash, and chunk ordinal, while a manifest records the active source hash. Hash the normalized content and all retrieval-relevant metadata, including locale, visibility, product area, and title. If permissions change but the hash ignores them, the prose may be fresh while access remains stale.

Fresh means more than recent.

Retries happen.

Pin a `chunker_version` too. A change to tokenization, overlap, or heading treatment requires replacement even when the source file does not change. Unicode Standard Annex #15 defines normalization forms, but the application still has to choose one and apply it consistently.

The failure boundary is the database commit. Fetching, parsing, chunking, and embedding may fail without touching the active index. PostgreSQL documents that a row-level lock blocks writers and lockers on the same row until the transaction ends, so a per-document manifest row provides the serialization point.

## Record the replacement protocol

| Approach | Freshness behavior | Main failure mode | Valid use case |
|---|---|---|---|
| Delete by `document_id`, then insert in one transaction | One committed version is active | Long transactions for huge chunk sets | Bounded support pages in transactional storage |
| Versioned insert, then flip an active pointer | Preparation happens before a short publish | Queries without the active-version filter return duplicates | Large documents and expensive bulk insertion |
| Upsert chunk IDs in place | Cheap when boundaries never move | Orphaned tail chunks survive re-chunking | Fixed records with permanent identities |
| Rebuild the whole index | Simple global snapshot | Freshness waits for the slowest source | Small corpora and schema-wide migrations |

The event needs `document_id`, monotonic `source_version`, content, retrieval metadata, and a deletion marker. A source-controlled revision or increasing sequence can order events; wall-clock timestamps are risky because clocks and delivery order can disagree. The content hash detects no-op work but does not establish order. This is also where queue semantics matter: at-least-once delivery is acceptable because equal versions are idempotent, and out-of-order delivery is acceptable because older versions are rejected, but an event with no durable per-document version cannot be repaired by clever SQL. The producer must establish that ordering contract before ingestion can preserve it.

A deletion is a versioned event. It locks the same row, removes chunks, and advances the manifest to a tombstone. Otherwise, a delayed update can recreate a deleted article. Keep tombstones for at least the system's maximum redelivery and backfill horizon; there is no universal safe duration.

## Implement the critical path in Python

This function assumes PostgreSQL, a DB-API connection, and a complete prepared chunk set. The schema enforces `PRIMARY KEY (document_id, chunk_id)` on `support_chunks` and `PRIMARY KEY (document_id)` on `index_manifest`.

```python
from dataclasses import dataclass
from typing import Any, Sequence


@dataclass(frozen=True)
class Chunk:
    ordinal: int
    text: str
    embedding: Sequence[float]
    metadata: dict[str, Any]


def replace_document(conn, *, document_id: str, source_version: int,
                     source_hash: str, chunker_version: str,
                     chunks: Sequence[Chunk], deleted: bool = False) -> str:
    ordinals = [chunk.ordinal for chunk in chunks]
    if len(ordinals) != len(set(ordinals)):
        raise ValueError("duplicate chunk ordinal")
    if deleted and chunks:
        raise ValueError("a deletion cannot publish chunks")

    with conn.transaction():
        with conn.cursor() as cur:
            cur.execute(
                """INSERT INTO index_manifest
                   (document_id, source_version, tombstoned)
                   VALUES (%s, -1, FALSE)
                   ON CONFLICT (document_id) DO NOTHING""",
                (document_id,),
            )
            cur.execute(
                """SELECT source_version, source_hash, chunker_version, tombstoned
                   FROM index_manifest WHERE document_id = %s FOR UPDATE""",
                (document_id,),
            )
            version, old_hash, old_chunker, tombstoned = cur.fetchone()
            if source_version < version:
                return "stale"
            if source_version == version:
                if (old_hash, old_chunker, tombstoned) == (
                    source_hash, chunker_version, deleted
                ):
                    return "unchanged"
                raise ValueError("one source version maps to conflicting content")

            cur.execute(
                "DELETE FROM support_chunks WHERE document_id = %s",
                (document_id,),
            )
            if not deleted:
                rows = [
                    (document_id, f"{source_hash}:{chunk.ordinal}",
                     source_version, chunk.ordinal, chunk.text,
                     list(chunk.embedding), chunk.metadata)
                    for chunk in chunks
                ]
                cur.executemany(
                    """INSERT INTO support_chunks
                       (document_id, chunk_id, source_version, ordinal,
                        body, embedding, metadata)
                       VALUES (%s, %s, %s, %s, %s, %s, %s)""",
                    rows,
                )
            cur.execute(
                """UPDATE index_manifest
                   SET source_version = %s, source_hash = %s,
                       chunker_version = %s, tombstoned = %s,
                       indexed_at = CURRENT_TIMESTAMP
                   WHERE document_id = %s""",
                (source_version, source_hash, chunker_version,
                 deleted, document_id),
            )
    return "deleted" if deleted else "replaced"
```

Expose the result as an operational outcome. Count `unchanged`, `stale`, `replaced`, `deleted`, and failed events separately, and alert on the oldest unapplied source version rather than throughput alone. A fast consumer can still serve yesterday's policy while one poison document retries forever.

The function deliberately rejects conflicting content at the same source version. Silently choosing one would conceal a broken ordering contract. An exception should roll back and permit redelivery; a dead-letter record can retain document ID, version, and error class without copying sensitive support text into logs.

## Test freshness at the retrieval boundary

Test the sequence that reveals residue: index three chunks, replace them with one, query by `document_id`, and assert exactly one remains. Replay the event and confirm the row count and active hash stay unchanged. Deliver version 11 before version 10; version 10 must be stale. Delete at version 12, replay version 11, and verify that nothing becomes searchable.

Concurrency needs a separate test. Two transactions for one document must serialize at the manifest lock, while updates for distinct IDs should proceed independently. PostgreSQL's default Read Committed isolation gives each command a snapshot, so keep consistency-sensitive retrieval to one statement or choose and test a different isolation level.

Deploy in stages: add manifest fields and dual-write them, backfill identities and versions, switch retrieval to the new contract, then enable replacement deletion. A chunker migration needs a new namespace or shadow index and an explicit cutover. Unchanged source files do not make an old indexed representation compatible.

Track source-to-commit lag by percentile, oldest pending age, outcomes, chunks per document, transaction duration, lock waits, and retrieval hits whose version differs from the manifest. Thresholds belong to the support operation's service objective and corpus distribution. Universal limits would be fiction.

## Why reject chunk-by-chunk upsert?

Its identity assumption is usually false for prose. If a 20-chunk article shrinks to 17 chunks, updating the survivors leaves chunks 18 through 20 behind. Content-derived IDs are worse without a sweep: changed chunks get new IDs while old IDs remain searchable.

There is one valid exception. If the source consists of stable, independently deleted records, such as messages with immutable IDs, per-record upsert and delete matches the source semantics. Token-window chunks are different; they are a derived snapshot, not durable business entities.

The limitation of delete-then-insert is its write amplification and the time spent holding the manifest lock. It is not suitable when a single logical document contains an unbounded number of chunks, when the target store cannot make deletion and insertion atomically visible, or when retrieval cannot tolerate the resulting transaction duration. The trade-off is deliberate: bounded support pages gain a small and auditable publication boundary, while unusually large documents need another mechanism.

For those large documents, versioned insert followed by a manifest-pointer flip is the serious alternative. It shortens publication and retains the prior set for rollback, but costs storage, garbage collection, and an active-version predicate on every query. Choose it when measured transaction time or lock contention violates the objective.

No universal winner exists.

The durable decision is narrow: stable document identity, explicit ordering, complete replacement, and one tested visibility boundary. Those properties stop an internal wiki assistant from quoting a retired escalation step after its article changes.

## References

- https://arxiv.org/abs/2005.11401
- https://www.postgresql.org/docs/current/transaction-iso.html
- https://www.postgresql.org/docs/current/explicit-locking.html
- https://www.postgresql.org/docs/current/sql-insert.html
- https://unicode.org/reports/tr15/
- https://docs.python.org/3/library/hashlib.html
