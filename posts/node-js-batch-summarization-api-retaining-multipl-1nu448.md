# Node.js Batch Summarization API: Retaining Multiple Call Documents in 2026

TL;DR: Retain the original transcript once, store each summary and CRM action as a small immutable derivative, and export a manifest that binds every derivative to its source by digest. The dominant term in a sales-call summarization bill is not automatically inference: measure transcript bytes, audio bytes, generated tokens, retention days, retries, and egress before choosing what to discard. A portable design keeps queue state, evidence, and provider-specific responses outside the model call, so a Node.js API can switch inference providers without changing the job or export contract.

The bill is a sum, not a slogan: `audio_bytes * audio_days`, `transcript_bytes * transcript_days`, `derivative_bytes * derivative_days`, model input and output, retry work, and export egress. Exact rates depend on the storage and inference contracts in force, so hard-coding a public price into architecture is brittle. Record quantities per job, join them to the applicable rate card, and find the dominant measured term. If retained audio dwarfs every other byte class, shortening audio retention moves storage consumption; trimming a compact JSON summary does not. If token processing dominates instead, retention tuning is theater.

I would deliberately stop keeping provider response envelopes after validation and a short diagnostic window. Keep the normalized result, prompt revision, model identifier, source digest, timestamps, usage counters, and validation errors. That choice removes provider-shaped payloads from long-term storage, but it costs forensic detail when a later dispute depends on an undocumented field in an old response. State that loss in the retention policy before an incident, not after one.

## What must survive when the provider changes?

The durable boundary is a media workflow, not a model SDK. A call becomes a transcript; the transcript becomes one or more bounded inputs; those inputs produce a summary and proposed CRM actions; a human or policy layer accepts, edits, or rejects those actions. Each transition needs an identifier and a digest. Without them, an exported `schedule_demo` action cannot prove which transcript revision produced it.

Use a neutral job record and treat vendor payloads as transient adapters. The API may be implemented in Node.js, while the worker below is Python because the contract is language-independent. `input_uri` points to controlled object storage, never to an expiring upload URL. `result_uri` remains unset until the manifest and all referenced objects exist.

| Durable object | Minimum useful fields | Failure mode it contains |
|---|---|---|
| Job | job ID, state, schema revision, created time | duplicate submission and incompatible readers |
| Source | object URI, media type, byte count, SHA-256 | silent replacement or truncated upload |
| Transcript | source digest, language, segments, transcript digest | edits detached from their recording |
| Derivative | transcript digest, prompt revision, model ID, normalized output | irreproducible CRM actions |
| Export manifest | member paths, digests, sizes, completion time | partial or mixed-version download |

A digest proves byte identity, not truth. It cannot establish that a speaker was attributed correctly or that a proposed follow-up reflects the call, so those remain evaluation and review problems.

Proof has limits.

## Measure retention before changing it

The first runnable component is an inventory pass. It totals actual classes of retained objects and calculates deletion candidates under explicit policy; it does not guess a currency amount. Feed its JSON output to the billing system that owns current rates.

```python
from collections import defaultdict
from datetime import datetime, timezone

POLICY_DAYS = {
    "audio": 30,
    "transcript": 365,
    "derivative": 730,
    "provider_envelope": 7,
}

def retention_inventory(objects, now=None):
    now = now or datetime.now(timezone.utc)
    totals = defaultdict(lambda: {"objects": 0, "bytes": 0, "expired_bytes": 0})
    for item in objects:
        kind = item["kind"]
        if kind not in POLICY_DAYS:
            raise ValueError(f"unknown retention class: {kind}")
        size = int(item["bytes"])
        created = datetime.fromisoformat(item["created_at"].replace("Z", "+00:00"))
        age_days = (now - created).total_seconds() / 86400
        totals[kind]["objects"] += 1
        totals[kind]["bytes"] += size
        if age_days >= POLICY_DAYS[kind]:
            totals[kind]["expired_bytes"] += size
    return dict(totals)
```

Run this against object metadata, then compare retained byte-days with recorded model input and output counts, retries, and downloaded export bytes. Use the relevant tokenizer only as an estimator for the encoding it supports; the tiktoken project describes itself as a BPE tokenizer and exposes model-aware encoding selection, but a provider's reported usage remains the settlement record for that provider. Token estimates should never decide whether evidence can be deleted.

Keep the policy small. Four retention classes are easier to audit than dozens of prefixes whose names merely imply lifecycle rules.

The concrete policy above uses four classes and three long-lived windows: 30 days for audio, 365 for transcripts, and 730 for derivatives. Those values are an executable example, not universal legal advice. A team must replace them when consent terms, deletion requests, litigation holds, or its governing rules require different periods. The trade-off is explicit: a shorter audio window reduces retained byte-days and exposure, while also removing the artifact needed to recheck diarization or transcription after that window closes. Keeping audio indefinitely avoids that particular loss but expands the material subject to access control, deletion, and audit. Neither choice repairs weak provenance.

## Implement an idempotent worker and normalized result

A queue can deliver work again, and a network failure can hide a successful remote response. The worker therefore derives a stable attempt key from the job, transcript digest, prompt revision, and adapter revision. It validates the normalized document and publishes with a create-only operation supplied by the storage adapter. The model adapter is the only provider-specific code.

```python
import hashlib
import json
from dataclasses import dataclass

@dataclass(frozen=True)
class WorkItem:
    job_id: str
    transcript_sha256: str
    prompt_revision: str
    adapter_revision: str

def attempt_key(item):
    material = "\n".join([
        item.job_id, item.transcript_sha256,
        item.prompt_revision, item.adapter_revision,
    ]).encode("utf-8")
    return hashlib.sha256(material).hexdigest()

def process(item, transcript, model_adapter, object_store):
    key = attempt_key(item)
    final_key = f"results/{item.job_id}/{key}.json"
    existing = object_store.get_if_present(final_key)
    if existing is not None:
        return json.loads(existing)

    raw = model_adapter.summarize(transcript, idempotency_key=key)
    normalized = {
        "schema_version": "1.0",
        "job_id": item.job_id,
        "source_sha256": item.transcript_sha256,
        "summary": str(raw["summary"]),
        "crm_actions": list(raw["crm_actions"]),
        "model_id": str(raw["model_id"]),
        "usage": dict(raw.get("usage", {})),
    }
    payload = json.dumps(normalized, sort_keys=True, separators=(",", ":")).encode()
    object_store.put_if_absent(final_key, payload, content_type="application/json")
    return normalized
```

`put_if_absent` must map to a real conditional-create primitive in the chosen store; a read followed by an unconditional write has a race. Likewise, an idempotency key narrows duplicate effects but does not prove that every remote service honors it. Record attempts separately, cap retries, and send terminal validation failures to a review queue rather than turning malformed actions into CRM writes.

The normalized schema should reject unknown action types, missing evidence references, impossible timestamps, and overlong field values before publication. Keep generation separate from mutation: the summarizer proposes CRM actions, while another component applies approved actions with its own idempotency key. This boundary limits the damage from hallucinated or duplicated output.

Don't auto-apply guesses.

## Export results without publishing a half-built batch

An export is complete only when its manifest is visible. Build members under a private staging prefix, calculate SHA-256 and byte length from the final bytes, write the manifest last, and expose only the manifest URI. Readers verify every member before import.

```python
import hashlib
import json
from datetime import datetime, timezone

def publish_export(job_id, result_documents, object_store):
    prefix = f"exports/{job_id}"
    members = []
    for index, document in enumerate(result_documents):
        body = json.dumps(document, sort_keys=True, separators=(",", ":")).encode()
        path = f"{prefix}/items/{index:06d}.json"
        object_store.put_if_absent(path, body, content_type="application/json")
        members.append({
            "path": path,
            "bytes": len(body),
            "sha256": hashlib.sha256(body).hexdigest(),
        })

    manifest = {
        "schema_version": "1.0",
        "job_id": job_id,
        "completed_at": datetime.now(timezone.utc).isoformat(),
        "members": members,
    }
    body = json.dumps(manifest, sort_keys=True, separators=(",", ":")).encode()
    object_store.put_if_absent(
        f"{prefix}/manifest.json", body, content_type="application/json"
    )
    return manifest
```

This ordering handles a worker crash during assembly: orphaned staged members can expire later, while consumers never see a completion marker that references unwritten objects. A second publisher may collide on conditional creation; it should compare the existing digest and succeed only when the bytes match. Do not overwrite a manifest in place. Publish a new export revision.

Observability should follow the same boundaries. Emit counts for queued, running, retried, validation-failed, review-required, and complete jobs; histograms for queue delay and processing duration; byte counters by retention class; model usage as reported by the adapter; and export verification failures. Logs carry job and attempt IDs, but no transcript text or CRM notes. Alerts should distinguish a growing queue from a rising validation-failure ratio because their remedies differ.

## Test portability as a property, not a promise

Run the same fixed transcript fixtures through every adapter, then validate normalized shape, evidence links, and policy constraints. Semantic equivalence needs a task-specific rubric: required commitments captured, dates preserved, speaker attribution retained, unsupported actions rejected, and no CRM mutation before approval. Exact string equality is the wrong test for summaries.

Deployment needs three gates. First, replay stored, consented fixtures without writing CRM data. Second, shadow a bounded slice and compare normalized outputs plus usage counters. Third, canary the adapter while holding schema and prompt revisions explicit. Rollback selects the prior adapter revision; it does not rewrite completed evidence.

The portability claim fails if prompts depend on proprietary response fields, token limits are assumed rather than discovered from the configured model, or provider errors leak into public job states. It also fails when transcription and summarization are fused into one opaque request. The ElevenLabs documentation is useful primary evidence for its own API surface, while its boundaries should remain inside a transcription adapter; it is not evidence that a generic export contract is correct.

This architecture has limitations. It is unsuitable for a workflow that requires synchronous answers inside one request, because queueing, conditional publication, and manifest verification add delay and operational machinery. It also cannot make two models semantically interchangeable: schema version `1.0` standardizes the envelope, not the judgment inside a summary. Teams unable to operate a queue, object lifecycle rules, and reconciliation should choose a simpler synchronous pipeline until batch volume or audit requirements justify those moving parts. The price is lower throughput isolation and weaker recovery from partial work.

A defensible acceptance rule is blunt: switch adapters only if both pass the same schema checks, fixture rubric, retry tests, retention accounting, and export verification. Latency and cost matter after correctness thresholds are met. Keep original audio only for the policy window that legal, quality, and dispute requirements justify; keep transcripts and accepted action evidence longer when policy calls for it; discard transient envelopes on schedule. The price of that last deletion is reduced historical debugging detail, a known loss that the manifest and normalized audit record cannot recreate.

## Further reading

- https://github.com/openai/tiktoken
- https://elevenlabs.io/docs
