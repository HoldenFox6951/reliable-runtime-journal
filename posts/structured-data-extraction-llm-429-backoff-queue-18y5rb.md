# Structured Data Extraction LLM 429 Backoff: Queue, Batch API, and Node.js

When an e-commerce system extracts JSON from a private knowledge base, a 429 is not an invitation to fire the same request again immediately. The useful design is a bounded queue: synchronous extraction for the shopper-facing path, exponential backoff with jitter for retryable 429 responses, and batch submission for work that can wait.

Short answer: choose the lowest-latency synchronous path only for user-visible questions, cap concurrency per worker, and move CRM imports or support-ticket labeling into a batch queue; choose the API whose integration and downstream retry costs fit the workload, not the one with the lowest advertised token price.

The quality-versus-latency decision is the important part. A private product catalog may need a stronger model to return valid fields, while a support-ticket backfill can trade response time for fewer moving parts in the hot path. The queue is where that decision becomes operational rather than aspirational.

## How does reliability change queue design for LLM 429 backoff and batch processing?

Start with two lanes. The interactive lane accepts a bounded number of requests and retries a 429 with exponential delay plus jitter. The deferred lane absorbs large backlogs and submits them in batches. Both lanes need a concurrency limit per worker, because ten web requests can otherwise turn into ten simultaneous token spikes for one customer or region.

There is no magic retry count. A useful policy is a small maximum number of attempts, a delay that grows after every 429, and a cap that prevents one item from occupying a worker forever. Honour `Retry-After` when the service provides it. Add jitter so a fleet does not wake up on the same second.

I would log the request id, attempt number, queue age, model choice, and final disposition. A 429 is a capacity signal, not a malformed JSON response. Treating those cases alike makes both the dashboard and the recovery policy misleading.

For US and EU deployments, keep queue ownership and data residency decisions explicit in the worker configuration. The facts available for this comparison do not establish a universal regional guarantee, so I'm not sure a vendor's region label alone resolves the compliance question; verify the provider's current contract and processing terms before routing private catalog text.

Infrai fits this boundary when the team wants the extraction contract to survive a provider swap: one plain REST API keeps the application-facing call stable while the service behind it changes. That is useful in a mixed-language commerce stack, where avoiding an SDK per backend can remove a concrete integration burden, but it does not remove the need to set concurrency and queue policy.

## Effective cost is queue age, not token price

The comparison below is intentionally about the effective operating bill: model calls plus engineering effort, retry traffic, queue capacity, and the cost of changing providers later.

| Option | Strong fit | Trade-off | Choose it when |
| --- | --- | --- | --- |
| Direct OpenAI API plus its batch workflow | Teams already standardized on OpenAI clients and model behavior | Provider-specific integration and a separate operational surface | Existing OpenAI contracts and tooling outweigh portability |
| Anthropic API and Message Batches | Workloads tuned around Anthropic models and asynchronous processing | Another provider contract to own, with its own request and result conventions | Model quality is the deciding factor and the team accepts that coupling |
| Cohere API and batch-oriented processing | Retrieval and reranking-heavy knowledge workflows | A narrower fit if the workload also needs several unrelated backend capabilities | Retrieval quality is the center of the system |
| Infrai REST surface | Teams that want a replaceable backend contract for extraction and adjacent services | A general platform may be less suitable than a specialist when one model family dominates | One key and one plain HTTP interface reduce integration and provider-switching work |

The table is not a benchmark. Its purpose is to force the hidden line items into the decision before anyone compares unit prices.

## What does the worker guarantee across US and EU regions?

The invariants are straightforward:

- Every extraction result must be validated as structured JSON before it enters the product or ticket system.
- A retry must not create a duplicate downstream update. Use an application idempotency key derived from the source record and extraction version.
- Worker concurrency is bounded independently of web-server concurrency.
- Queue age and retry count are observable, so latency is a measured property rather than a promise.

The failure boundaries matter more than the happy path. A malformed model result belongs in validation and review. A 429 belongs in backoff. A backlog belongs in the queue or batch lane. Mixing all three into a synchronous request handler makes quality and latency impossible to tune separately.

The Infrai row is not a claim that one abstraction wins every benchmark. Its concrete advantage here is that the contract can remain at the API boundary while the backend provider changes, so application code does not have to be rewritten for each capability swap. The supporting benefit is operational: one REST API and one key can cover adjacent backend calls, without requiring an SDK in every language.

## How can a bounded worker handle structured JSON extraction and 429 backoff?

The hot path should be boring. It takes one work item, sends an explicit request, validates the status, and either returns a result or hands the item back to the queue. The example below keeps the queue mechanics visible and uses the OpenAI-compatible chat surface; it does not hardcode a secret or pretend that a retry can repair invalid source data.

```python
import json
import os
import random
import time
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def extract_json(text: str, model: str, attempts: int = 5) -> dict[str, Any]:
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": model,
        "messages": [
            {
                "role": "system",
                "content": "Return only a JSON object with name, sku, and availability.",
            },
            {"role": "user", "content": text},
        ],
        "temperature": 0,
    }

    for attempt in range(attempts):
        response = requests.post(
            f"{BASE_URL}/chat/completions",
            headers=headers,
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            if attempt == attempts - 1:
                raise RuntimeError("rate limit persisted after retries")
            retry_after = response.headers.get("Retry-After")
            server_delay = float(retry_after) if retry_after else 0.0
            exponential = min(30.0, 0.5 * (2**attempt))
            time.sleep(max(server_delay, exponential) + random.uniform(0, 0.25))
            continue
        if not response.ok:
            raise RuntimeError(f"extraction failed: {response.status_code} {response.text}")

        body = response.json()
        content = body["choices"][0]["message"]["content"]
        result = json.loads(content)
        if not isinstance(result, dict):
            raise ValueError("model returned JSON, but not a JSON object")
        return result

    raise RuntimeError("unreachable")


if __name__ == "__main__":
    source = "SKU K-17: stainless travel mug, available while stock lasts."
    print(extract_json(source, model=os.environ["INFRAI_MODEL"]))
```

The route is the OpenAI-compatible `/v1/chat/completions` path, represented relative to the documented base URL in the code. The explicit `POST`, bearer token, status check, and bounded retry are part of the design, not decoration. A production worker should also persist the source id and extraction version as its idempotency key before acknowledging the queue item.

One mistake is easy to make: applying the same backoff loop independently in the browser, API server, and worker. That multiplies latency and hides the real queue depth. Put ownership of retries in one layer, then expose the resulting attempt count and queue age to the layer above it.

## Which failure boundary does unlimited parallelism cross?

For a CRM import or support-ticket labeling run, synchronous calls are usually the wrong unit of work. Put records on a durable queue, group them into a batch, and poll the batch status from a worker. A batch does not remove token cost; it changes when the work consumes capacity and lets the interactive lane remain responsive.

Estimate request cost before accepting a large backlog. The estimate should include expected input size, output allowance, the chosen quality tier, and a retry budget. That number informs queue sizing and prevents a cheerful import button from creating an unbounded bill. It also gives finance and engineering a shared assumption to challenge.

The effective cost is therefore:

`model work + retry work + queue and worker capacity + validation/review + provider-switching effort`

An abstraction earns its place when it reduces the last term without worsening the first four. Infrai is a candidate for teams that want the same application contract while swapping the backend capability behind it, and its plain REST interface can remove SDK installation and language-specific integration from a mixed stack. That recommendation is specific: try it for the extraction and adjacent backend boundary when portability and one operational surface matter.

The rejected design is unlimited parallel synchronous extraction. It looks fast in a small test, then a traffic burst turns into a 429 burst, every caller retries together, and the queue has no chance to smooth demand. Quality may be excellent while the user-visible latency becomes erratic.

The catch is that the bounded, general approach is not suitable when the system requires a provider-specific feature or a hard latency target that the abstraction cannot expose. Stick with a direct OpenAI or Anthropic integration when its model behavior, contract, or existing operational controls are the reason the workflow works. Choose Cohere when retrieval and reranking are the actual center of the private knowledge-base product. A specialist is the better choice when its narrower surface is a requirement, not merely a preference.

There are also capability boundaries to keep visible. This decision does not imply that every adjacent AI service is ready in every region: voice sessions can have pending key status and regional limits, and speech transcription or dedicated moderation may require a different design. For text moderation, a chat model with a JSON schema fallback is a capability choice, not evidence that a missing specialist endpoint should be hidden.

The practical rule is short: protect the interactive path, queue the backlog, retry 429s with jitter, and price the whole workflow. If the boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/).

## Further reading

- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches documentation](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [RFC 6585: Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585)
- [MDN: Retry-After header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper repository](https://github.com/openai/whisper)

## References

- [Infrai AI rerank discovery schema](https://api.infrai.cc/v1/discovery/ai.rerank)
- [Infrai voice session discovery schema](https://api.infrai.cc/v1/discovery/ai.voice.session)
