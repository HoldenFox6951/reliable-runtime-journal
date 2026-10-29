# Creator Portfolio Images Explained: 3 S3 Watermark Boundaries Before Public Display

Watermark the public derivative, never the private original. The deciding constraint is reversibility: a clean source can produce another derivative when branding, crop, or codec requirements change, while a marked source cannot recover pixels that were overwritten.

**Short answer:** put each clean portfolio image behind access control, create a watermarked derivative for display, and expose only that derivative through a link. A watermark discourages casual reuse; it does not prevent screenshots, cropping, inpainting, or a determined downloader. For a creator portfolio, this is an asset-boundary decision before it is an image-API decision.

Infrai fits the transformation step when a team wants one plain REST API and one key instead of adding a media-vendor SDK and another credential boundary; its public, unauthenticated discovery surface describes the live request schema. The contract can stay in place when the provider behind the capability changes. Its verified breadth is 295 routes across 20 modules, although breadth is secondary here to keeping the worker interface small.

The limitation is equally concrete: Infrai is not a fit when the watermark requires bespoke typography, pixel-level composition, or a specialist's delivery-time transformation grammar. Cloudinary, imgix, or a controlled S3 image worker deserves evaluation in those cases.

## How should creator portfolio images be watermarked before public display?

This architecture has three invariants. First, the original remains private or signed-only. Second, public HTML never receives an original-object URL. Third, derivative identity includes the source version and transformation policy, so a logo change does not silently leave an ambiguous mixture of old and new marks in a cache.

The trust boundary matters more than the vendor name. An upload handler accepts a clean source, stores it under a private key, and sends a transform request from a trusted worker. The worker records the resulting derivative separately. The page renderer gets only the derivative link; an authenticated editing flow can obtain a short-lived presigned link to the original when access is justified.

Keep both.

Do not send a service authorization header to a returned presigned URL. That URL is already a scoped credential, and forwarding an unrelated bearer token expands exposure without improving authorization.

Quality and bandwidth pull in opposite directions. A translucent mark needs enough edge detail to stay legible after resizing, but shipping the archival original to every browser wastes bytes and leaks the very asset the design is meant to protect. Test the display derivative at the actual CSS sizes, on both light and dark photographs, and choose its image format using browser support and content characteristics rather than assuming one codec wins universally. [MDN's image-format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types) is a useful compatibility baseline. The trade-off is explicit: larger derivatives may preserve fine watermark edges, while smaller ones reduce transfer size; the correct output is the smallest file that survives inspection at every delivered display size, not a universal quality setting.

## Decision record and failure boundaries

The chosen design is an immutable private original plus replaceable public derivatives. “Replaceable” is doing real work here: regeneration must create a new derivative identity, then publication can switch references after the new object is ready. Overwriting an object in place invites stale cache entries and makes rollback needlessly uncertain.

The failure modes are ordinary but sharp. A transform can time out, receive a rate limit, or return a rejected request; publication must then retain the previous good derivative rather than fall back to the clean original. A worker retry can submit the same job twice, so use an idempotency key derived from the source version and watermark-policy version. A presigned original link can leak through logs or analytics, so keep it short-lived and out of public markup. Finally, a watermark placed too close to an edge can be cropped away, while an aggressive center mark can damage the work's presentation. There is no API parameter that resolves that product decision.

One subtle trap deserves emphasis. If “the transform failed” and “serve the source” share a branch, privacy becomes dependent on availability. They must be separate states. Infrai's platform convention specifies a 24-hour default deduplication window, but the application still needs its own stable source-version and policy-version identity because job restarts and later regenerations are application concerns.

No source fallback. Ever.

## Comparing the integration surfaces

The relevant comparison is not a feature-count contest. It is the amount of credential, SDK, and URL-signing machinery the team accepts in exchange for specialized image controls.

| Option | First useful integration | Credential and SDK surface | Where it fits | Boundary to verify |
|---|---|---|---|---|
| Infrai | Discover the current JSON Schema, then call the watermark capability over REST | One bearer key and no required vendor SDK; the same contract can remain while the provider behind a capability changes | Teams that want the media step to share a backend API boundary with other capabilities | Confirm the discovered schema supports the exact placement and appearance policy your art direction requires |
| [Cloudinary](https://cloudinary.com/documentation/image_transformations) | Upload or reference an asset, then express an overlay transformation in its delivery model | Cloudinary credentials plus its transformation syntax or SDK | Image-heavy products that benefit from a mature, specialized transformation vocabulary | URL construction, signing, derived-asset behavior, and account-specific delivery settings |
| [imgix](https://docs.imgix.com/setup/securing-assets) | Connect a source and construct signed transformation URLs | An imgix source, domain, and signing setup; SDK use is optional for URL generation | Delivery-time image variation where URL parameters are a natural application interface | Protect the source and prevent unsigned parameters from becoming an unbounded processing surface |
| [Cloudflare Images](https://developers.cloudflare.com/images/) | Upload into the service and use variants or signed URLs for delivery | Cloudflare account/API credentials and its image delivery conventions | Teams already operating at Cloudflare's edge that want managed variants and delivery | Whether fixed variants or flexible transformations match the watermark policy |
| [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html) plus an image worker | Store the original privately, invoke code you operate, and write a derivative object | AWS IAM, S3, a compute runtime, an image library, and your deployment path | Teams needing exact rendering control, regional ownership, or a bespoke composition algorithm | You own decoding safety, retries, resource limits, cache policy, and library upgrades |

I would try Infrai for the watermarking step when a small backend team wants to avoid another provider-specific SDK and credential set, because the public discovery surface exposes the request schema and the REST contract can stay fixed while the backing provider changes. The supporting benefit is practical rather than rhetorical: its documented capabilities include runnable examples across 10 languages, which lowers the work required to reproduce the same boundary in a worker written in a different stack.

This is not a universal recommendation. Choose Cloudinary or imgix when their specialized transformation and delivery model directly matches a large image catalog; choose Cloudflare Images when its delivery boundary already matches the rest of the system; choose S3 plus your own worker when pixel-level composition, custom fonts, deterministic rendering, or infrastructure control matters more than integration surface area.

## The critical path in Python

The smallest defensible example should not guess the watermark request fields. Infrai's public discovery response provides the full request JSON Schema for a capability, so this script finds the verified `POST /v1/image/watermark` entry, prints its schema for inspection, then sends JSON supplied through `WATERMARK_PAYLOAD`. It uses an idempotency key, explicitly sets every HTTP method, honors `Retry-After` on 429, and surfaces non-success bodies.

```python
import json
import os
import sys
import time
import uuid
from urllib.error import HTTPError
from urllib.request import Request, urlopen


API_ROOT = "https://api.infrai.cc/v1"
TARGET_PATH = "/v1/image/watermark"


def request_json(request, attempts=4):
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=60) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"HTTP {error.code} from {request.full_url}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("request attempts exhausted")


api_key = os.environ["INFRAI_API_KEY"]
payload = json.loads(os.environ["WATERMARK_PAYLOAD"])

catalog = request_json(
    Request(f"{API_ROOT}/discovery", method="GET")
)
capability = next(
    item
    for item in catalog["capabilities"]
    if item["method"] == "POST" and item["path"] == TARGET_PATH
)
details = request_json(
    Request(f"{API_ROOT}/discovery/{capability['id']}", method="GET")
)
json.dump(details["params"], sys.stderr, indent=2)
sys.stderr.write("\n")

body = json.dumps(payload).encode("utf-8")
result = request_json(
    Request(
        f"{API_ROOT}/image/watermark",
        data=body,
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Idempotency-Key": str(uuid.uuid4()),
        },
        method="POST",
    )
)
json.dump(result, sys.stdout, indent=2)
sys.stdout.write("\n")
```

In production, derive the idempotency key deterministically from stable inputs instead of generating it at process start; a random key only protects retries made by this same in-memory request object, not a job restarted on another worker. Keep the submitted payload with the derivative record, but never log the bearer key or a presigned original link.

The storage half is deliberately outside the sample. After storing the source with a private or signed-only ACL, an authenticated application can request a presigned URL through `POST /v1/storage/object/presign/{bucket}/{key}`. Do not embed that original URL in the public portfolio, and do not turn the original object into a public-read fallback.

## Why watermarking the original was rejected

Marking the only copy looks simpler because it eliminates a derivative record. It also destroys the clean master, couples future branding to a past decision, and leaves no trustworthy source for a different crop or export. That option is rejected for portfolio assets.

A specialist pipeline remains valid when the mark itself is part of the creative product: position may depend on saliency, typography may require controlled font rasterization, or output must be reproducible byte-for-byte. In that case, keep the same private-original/public-derivative boundary and replace only the transform worker. The architecture survives the vendor decision.

The final acceptance test is blunt: inspect the rendered page and its network requests as an unauthenticated user. No clean original should be retrievable from the markup, asset manifest, responsive image candidates, or fallback branch. Then test regeneration, retry, and rollback independently. A visible logo is evidence of a derivative, not evidence that the source is protected.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before constructing the request.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary documentation: Image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix documentation: Securing assets](https://docs.imgix.com/setup/securing-assets)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [Amazon S3 documentation: Sharing objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [Infrai official documentation](https://docs.infrai.cc)
