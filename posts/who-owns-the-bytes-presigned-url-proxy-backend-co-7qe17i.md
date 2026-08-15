# Who Owns the Bytes? Presigned URL, Proxy Backend, Cost, and Security

Short answer: use a short-lived presigned URL for ordinary browser uploads when the application can validate the object after arrival; keep a backend proxy for small payloads that must be inspected, transformed, or accepted atomically before storage. The decisive constraint is where the bytes must be trusted, not which path looks simpler in a diagram.

A browser-to-storage path removes the application server from the bulk data path. A relay sends every byte through that server first. This changes bandwidth, latency, failure handling, and the point at which the application is allowed to call an upload complete. It doesn't remove the backend from the design: the backend still authenticates the user, grants a narrow upload contract, and verifies the resulting object.

## Start with the trust boundary, then draw the data path

The useful first question is not "Can the browser upload directly?" It is "Which component is permitted to declare these bytes acceptable?" A presigned URL delegates a limited storage operation to an untrusted browser. The capability should identify the intended object and expire; application authorization happens before issuance, while content validation happens after storage receives the object. A proxy can inspect the stream before committing it, but then the application owns the upload's memory, timeout, connection, and retry behavior.

Treat the upload as a small state machine: `requested`, `transferring`, `received`, `validated`, then `available` or `rejected`. The browser's success response proves only that the transfer endpoint accepted something. It does not prove that the object belongs in a user-visible collection, that its declared media type matches its content, or that a later processing step succeeded. This distinction matters because a refresh, duplicate retry, or abandoned transfer can otherwise create an object with no durable application record.

The path-selection rule can stay small enough to test without importing a storage SDK:

```python
from dataclasses import dataclass
from enum import Enum


class UploadPath(Enum):
    DIRECT = "presigned_url"
    PROXY = "backend_proxy"


@dataclass(frozen=True)
class UploadPolicy:
    inspect_before_storage: bool
    transform_inline: bool
    client_supports_direct: bool


def choose_upload_path(policy: UploadPolicy) -> UploadPath:
    if policy.inspect_before_storage or policy.transform_inline:
        return UploadPath.PROXY
    if not policy.client_supports_direct:
        return UploadPath.PROXY
    return UploadPath.DIRECT


assert choose_upload_path(UploadPolicy(False, False, True)) is UploadPath.DIRECT
assert choose_upload_path(UploadPolicy(True, False, True)) is UploadPath.PROXY
```

Keep this decision separate from capability issuance and object validation. It describes the required data path; it does not pretend that choosing `DIRECT` authenticates a user or that choosing `PROXY` validates a stream.

Keep new objects in a staging namespace and use an application-generated, non-guessable object key rather than a filename supplied by the browser. Record the expected size, intended owner, and upload identifier before granting the transfer. After arrival, read storage metadata, run the required validation, and make the database record visible only when those checks pass. Configure lifecycle management for abandoned staging objects; object lifecycle rules apply actions to objects based on defined conditions, so they are a better cleanup mechanism than hoping every client completes a callback.

This is the hard part.

The backend must also decide what duplicate completion means. A clean contract makes completion idempotent: repeated notification for the same upload identifier either returns the already-established result or rejects metadata that conflicts with the original request. Don't let a second callback silently attach the same object to a different owner.

## How should a beginner choose between a presigned URL and a proxy backend for secure web app file upload?

Use constraints that can be tested. The comparison below avoids invented request prices because the relevant bill depends on provider, region, network direction, object size distribution, and retry rate. Model your workload with those inputs instead.

| Constraint | Presigned browser-to-storage path | Backend proxy path |
|---|---|---|
| Application bandwidth | Metadata and control requests cross the app; file bytes do not | Every file byte crosses the app before storage |
| Latency path | Browser sends bytes to storage directly | Browser-to-app and app-to-storage work sit in the data path |
| Pre-commit inspection | Limited to constraints enforced by the signed operation; deeper checks occur after receipt | The app may inspect or transform the stream before storage commit |
| Failure ownership | Client, storage transfer, finalization, and validation are separate failure domains | The app coordinates the inbound stream and storage write, but owns both failure domains |
| Scaling pressure | Mostly control-plane request load on the app | Connections, bandwidth, buffering, timeouts, and backpressure on the app |
| Best fit | Large or frequent objects with asynchronous validation | Small objects requiring synchronous inspection or transformation |

For cost, write two formulas before comparing architectures. Direct transfer cost is approximately storage operations plus storage capacity plus applicable network transfer plus validation work. Relay cost contains those terms and also the application's ingress handling, outbound path to storage, compute duration, and capacity held for concurrent streams. Exact treatment varies, and I'm not sure a generic estimate is useful until real object sizes and network boundaries are known. Your mileage may vary — especially when the application and storage are not in the same location.

For latency, measure from the browser's first byte until the object becomes available, not merely until the transfer endpoint responds. Report transfer and validation separately. A direct path often shortens the byte route, but asynchronous scanning can lengthen time-to-availability; a proxy can reject earlier, yet saturation at the application tier may create queueing. Those are hypotheses to load-test, not universal rankings.

Measure both.

## Make the upload contract narrower than the user session

A presigned URL is a bearer capability. Whoever possesses it can exercise the signed operation until its conditions or expiry prevent use, so don't treat it like a harmless redirect. Authenticate the request that creates it, authorize the target collection, generate the object key on the server, choose a short lifetime that still tolerates expected client conditions, and avoid logging the full URL.

The contract should bind what the storage signing mechanism can reliably enforce, while the application record carries the rest. Record maximum expected size, owner, purpose, and expiration. Treat filename and media type as untrusted display metadata. If the product requires malware scanning, archive expansion limits, image decoding, or document conversion, keep the object unavailable until that pipeline finishes.

A proxy doesn't make those checks automatic. It gives the application a place to perform them before storage commit, at the cost of becoming a streaming service. Buffering an entire upload in memory is the obvious failure mode, but the quieter ones are inconsistent request limits between edge and application, a timeout shorter than the slow-client transfer, retry after a partial storage write, and backpressure that arrives only after worker capacity has already been consumed. Stream with explicit limits, abort both sides on failure, and make partial-object cleanup part of the design.

Security review should cover both paths with the same questions: Can one tenant select another tenant's key? Can a client overwrite an existing object? Does a failed validation leave a reachable URL? Are download authorization and upload authorization separate? Do logs capture secrets? Can lifecycle cleanup delete a newly valid object because state propagation raced with the rule? The last question deserves a deployment test — policy and naming must make active and abandoned objects unambiguous.

## Test failures and observe the state transition

A happy-path upload proves very little. Consider one ordinary failure chain in detail: the application creates an intent, the browser sends the object successfully, and the network disappears before the completion request reaches the backend. The user retries, perhaps from a refreshed page. If object keys come from filenames, the second attempt may overwrite the first; if every attempt gets a new key, both objects may remain in staging; if completion is not idempotent, two database rows may point at one object. Now let the first completion arrive late, after the intent has expired but before lifecycle cleanup evaluates the staging prefix. The correct result depends on the contract chosen in advance, yet every branch must preserve tenant ownership, prevent an unvalidated object from becoming readable, and leave cleanup enough state to distinguish abandoned data from a valid upload. This example has no exotic storage failure. It is only message loss plus retry, which is exactly why the state machine, server-generated key, idempotency rule, and staging lifecycle belong in the initial design rather than a later hardening sprint. Test that sequence, then test a browser that disconnects midway, lies about media type, sends an empty body, reaches the size boundary, and calls completion twice. For a proxy, add slow readers, slow storage writes, and application shutdown during transfer. For a direct path, add an expired capability and an object that fails post-upload validation.

Instrument an upload identifier across the control plane and validation pipeline, but keep signed query parameters out of logs. Useful measures include bytes requested versus bytes validated, time in each state, abandoned staging age, rejection reason, duplicate completion count, and time-to-availability percentiles by size band. A single "upload failed" counter collapses unrelated causes and tells the operator almost nothing.

Deployment deserves restraint. Put size and media policies in one versioned application contract, then verify that edge limits, proxy limits, signing conditions, validation workers, and lifecycle rules agree with it. Roll out with a small traffic slice and compare completion, rejection, abandonment, and availability latency. Don't infer success from lower application CPU alone; a direct path can move work into validation queues and cleanup.

## How can the data path change without a risky migration?

Begin with one upload-intent endpoint and one completion model even if the initial implementation uses a proxy. Keep the data path behind that contract. The browser asks for an upload intent, receives either direct-transfer instructions or a relay target, performs the transfer, and observes application state until the object is available. This keeps UI behavior and domain state stable when the transport changes.

Migrate by object class, not all at once: move a bounded class of larger, asynchronously validated objects to direct transfer; retain the proxy for payloads that need synchronous transformation. Compare state-transition metrics, exercise rollback, and only then widen eligibility.

The catch is that direct upload is not suitable when policy requires the application to inspect every byte before any storage commit, when the target storage cannot express a sufficiently narrow temporary capability, or when clients cannot perform the required direct request. Stick with a proxy in those cases. Conversely, a proxy is a poor default for sustained large-object traffic unless the team deliberately wants to own streaming capacity, backpressure, and retry semantics. A hybrid selected by object policy is often the honest design, because "file upload" is rarely one workload.

## Further reading

- [Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloud Storage documentation](https://cloud.google.com/storage/docs)
