# Private SaaS Documents: Browser Uploads, Signed URLs, Postgres, Tenant Image Exports

The constraint that changes this design is not image size; it is the tenant boundary during export. Short answer: keep the browser-to-object-storage transfer direct, keep authorization and export membership in PostgreSQL, and issue short-lived signed links only after the service has checked the tenant context. Delivery becomes simple at the edge, while the application still owns the decision about which images belong in an export.

This is a control-plane decision. Object storage holds bytes and metadata about an object. It should not decide that an image belongs to tenant A because its key happens to start with `tenant-a/`. That rule belongs in a transactionally queryable system, where the export request, image row, tenant, and authorization check can be observed together.

## Reliability failure modes in the export snapshot

Record intent before transfer. A request to add `hero.png` to a tenant's product catalog should create an image row with an opaque object key, tenant ID, content type, declared size if available, and an explicit state such as `uploading`. The service then signs an upload for that exact key. The browser never chooses a key that can overwrite another tenant's object, and it never receives the service credential used to sign the request.

The export path starts from PostgreSQL, not from an object listing. Select images with the tenant predicate, the catalog or product predicate, and a state that means the upload has been confirmed. Store an export row and its selected image IDs, or store a reproducible query plus a snapshot boundary if the product requires that behavior. A later download should be a consequence of that export record, not a fresh interpretation of a bucket prefix.

That distinction prevents a quiet class of authorization bugs. A user can have permission to view one product while lacking permission to export the tenant's full catalog; a prefix-based implementation tends to collapse those decisions into one string comparison. The Postgres query makes the stronger operation visible in code and review. It is a private-documents pattern, even when the payload happens to be product imagery.

## Should a SaaS use browser upload and signed URLs for private documents?

Treat every signed link as a short-lived bearer capability. For upload, bind the link to the one opaque key and the expected operation; for download, mint it only after checking that the caller can access the particular export. Do not put a download URL in a durable export record, an analytics event, or a support ticket. Store the object key and the authorization facts; generate the capability when it is needed.

The browser's success callback is not a storage receipt. A tab can lose the response after the service has accepted the bytes, and a progress event reports transport progress rather than application state. The confirmation endpoint should re-check the object, then make the state transition from `uploading` to `ready`. A retry must address the same operation ID. Otherwise two tabs can create two image rows for one user action, and a later export may include one copy or both.

Use a deliberately boring state machine:

`uploading -> ready -> included`

An invalid or abandoned upload can become `rejected` or `expired` without being eligible for export. The exact retention policy is a product decision, but the state must exist before cleanup starts. I'm not sure any single expiry interval fits every catalog workflow; the answer depends on how long users are allowed to resume an upload and how the team reconciles abandoned objects. Keep it private.

## Compare delivery paths for private image exports

The useful comparison is where bytes travel and where policy lives. It is not a race to find the shortest endpoint.

| Design | Access-control surface | Delivery simplicity | Failure boundary |
|---|---|---|---|
| Stream through Node.js | Every byte passes through application code | Low for the browser, high for the service | Application bandwidth and timeout handling become part of the export path |
| Public object URLs | Policy is mostly delegated to URL possession | High | A leaked URL can outlive the intended tenant decision |
| Direct transfer with signed links | PostgreSQL authorizes; storage enforces the temporary capability | High after the signing flow is correct | Expiry, leakage, and confirmation races need explicit handling |
| Prebuilt tenant export archive | Authorization is evaluated while assembling a fixed artifact | High for repeated downloads | Revocation and stale membership need an export lifecycle |

For product images, direct transfer with signed links is the default when exports are assembled from a modest set of already-authorized objects. A prebuilt archive is a better fit when the same tenant downloads the result repeatedly or when one download must be a stable snapshot. Streaming through Node.js remains reasonable for a small response that must be transformed on demand, but it is a poor default for a large image set because the application becomes the delivery bottleneck.

The catch is that signed links do not replace authorization. They compress a prior authorization decision into a time-limited capability. They are not suitable for a permanent public gallery, a requirement for immediate revocation after issuance, or a workload that needs object-level policy the selected storage interface cannot express. Choose a proxy or a prebuilt, access-controlled archive when those requirements dominate. The extra hop is sometimes the honest price of control.

## Code path for a small Postgres state machine

The application-side transaction should be idempotent. The signer is intentionally represented as a dependency because its concrete request shape varies by storage implementation; the important invariant is that it signs the server-generated key, never a client-supplied path.

```python
import uuid


def begin_image_upload(db, tenant_id, product_id, operation_id, filename, content_type):
    object_key = f"images/{tenant_id}/{uuid.uuid4()}"
    with db.transaction():
        row = db.fetch_one(
            """
            INSERT INTO product_images
                (tenant_id, product_id, operation_id, object_key,
                 filename, content_type, state)
            VALUES (%s, %s, %s, %s, %s, %s, 'uploading')
            ON CONFLICT (tenant_id, operation_id) DO UPDATE
                SET operation_id = EXCLUDED.operation_id
            RETURNING id, tenant_id, object_key, state
            """,
            (tenant_id, product_id, operation_id, object_key,
             filename, content_type),
        )
    upload_url = signer.presign_upload(row["object_key"])
    return {"image_id": row["id"], "upload_url": upload_url}
```

There is a subtle issue in this compact example: a retry must return the original row's key, not generate a new key and pretend the conflict update solved the whole problem. Production code should branch on the returned state, load the existing record when the operation already exists, and sign that existing key. It should also reject a retry whose tenant, product, or content type does not match the original request. I use a 409 for that mismatch; it's a request conflict, not permission to create a second object. This is the kind of edge case that looks harmless in a happy-path test and becomes duplicate catalog data in production.

After the browser completes the transfer, the service checks the object and advances only the matching tenant row. To create an export, it locks or snapshots the selected ready rows according to the product's consistency requirement. To download, it checks the export's tenant against the authenticated caller, then signs the object or archive key. The browser can show upload progress with standard upload events, but the server remains the source of truth for readiness.

## Test isolation under retries and expiry

Test the boundaries, not just a successful upload. A useful integration matrix includes a wrong-tenant confirmation, a repeated operation ID, a lost browser response followed by retry, an expired upload link, an object missing at confirmation, and an export created while an image is still `uploading`. Add a cross-tenant property test for every read and export query: changing only the authenticated tenant must never return another tenant's object key.

Log operation ID, tenant ID, image ID, export ID, state transition, and object key hash. Do not log the signed URL. Metrics should separate upload initiation, confirmation, expiration, reconciliation, and download issuance; one aggregate “storage error” counter hides the difference between a client abandoning a tab and an authorization check rejecting a request.

The rejected shortcut is “list the prefix and zip whatever appears.” It is acceptable for a bounded maintenance reconciliation that compares storage inventory with database rows. It is not a tenant export algorithm. Prefixes are naming conventions, not relational authorization, and an object can exist before its database state is ready or after its row has been deleted. Keep the ledger authoritative, make reconciliation explicit, and let delivery remain a narrow capability granted after the policy decision.

## References

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html
- https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest
