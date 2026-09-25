# Node.js Logistics Avatar Upload: Replace Existing Files Safely Without Overwrite Races

TL;DR: To replace an existing avatar or logistics media file safely, give every upload a new tenant-scoped object storage key and publish that key through a conditional database update. Never overwrite the object currently named by the record. For a one-person SaaS, this strategy keeps large proof-of-delivery videos out of Node.js memory while preventing a slow, older request from silently becoming current.

| Choice | Concurrent replacements | Tenant boundary | Operational burden |
|---|---|---|---|
| Stable object key | Last completed write can win | Depends on every caller | Low until requests overlap |
| Unique key plus unconditional pointer update | Objects stay distinct; stale publication remains possible | Visible in key and authorization | Moderate |
| Unique key plus conditional pointer update | One acknowledged version becomes current | Enforced before signing and publishing | Moderate |

**Recommendation:** use the third option. Store immutable upload attempts under keys such as `tenants/{tenantId}/shipments/{shipmentId}/media/{uploadId}`. Keep one database pointer to the current object, and advance it only if the row still has the version the request originally read. The object store handles bytes. The application owns authorization and publication order.

## How can an avatar upload replace an existing file safely?

Unique keys stop two clients from writing into the same object name. They do not decide which completed avatar upload or shipment video should appear on the page. Consider two dispatcher tabs. Both read media version 12. Tab A starts a 900 MB upload, then tab B starts a corrected 40 MB upload. B finishes first and publishes version 13. If A later performs an unconditional update, the old file becomes current even though no object was overwritten.

That is a pointer race. Fix it where the pointer lives. A compare-and-swap style update changes the row only when its version still matches the version observed before upload. The losing request receives a conflict and its object remains unreferenced until cleanup. HTTP defines conditional requests and the `If-Match` precondition for avoiding lost updates; the same precondition idea applies when the authoritative comparison happens in a database transaction.

This distinction matters to a solo operator. Immutable names outsource byte durability and concurrent object creation to storage, while a small SQL condition preserves the business meaning I need: the latest accepted correction wins, not the slowest network transfer. It is one extra column, not a distributed lock service.

No overwrite required.

## Tenant isolation comes before the signed upload

A key prefix helps audit and lifecycle rules, but a prefix is not authorization. The server must derive `tenantId` from the authenticated session, verify that the shipment belongs to that tenant, allocate an unpredictable `uploadId`, and then issue a narrowly scoped upload capability. Never accept a complete object key supplied by the browser.

The publication request needs the same checks. Bind the upload record to tenant, shipment, expected media type, expected version, and object key before any client receives permission to send bytes. After upload, publish only that recorded key. This closes a cross-tenant hole: a user cannot paste an object name from another account and ask the API to attach it.

Use private objects. Return downloads through a short-lived authorized URL or an authenticated delivery path, according to the threat model. Signed URLs are bearer capabilities, so possession grants the access encoded in the URL until it expires. Keep their scope and lifetime narrow, and avoid logging the full query string.

The upload intent also gives operations a clean ledger. `pending` means a capability was issued, `published` means the database pointer references the object, and `abandoned` means cleanup may remove it after a conservative grace period. Do not delete the previous object inside the request that publishes its replacement. Readers, caches, and retrying jobs may still hold the old key. Retire old versions asynchronously after the retention window.

This is deliberately boring infrastructure.

## A focused Node.js publication path

The example leaves signing and object transport behind interfaces. The important boundary is the update predicate. The client sends the opaque upload ID and the version it observed; it cannot choose a tenant or object key.

```ts
type PublishInput = {
  shipmentId: string;
  uploadId: string;
  expectedVersion: number;
};

type Session = { tenantId: string; userId: string };

interface Transaction {
  findPendingUpload(input: {
    id: string;
    tenantId: string;
    shipmentId: string;
  }): Promise<{ id: string; objectKey: string } | null>;
  publishShipmentMedia(input: {
    tenantId: string;
    shipmentId: string;
    objectKey: string;
    expectedVersion: number;
  }): Promise<boolean>;
  markPublished(uploadId: string): Promise<void>;
}

interface Database {
  transaction<T>(work: (tx: Transaction) => Promise<T>): Promise<T>;
}

class ConflictError extends Error {}
class NotFoundError extends Error {}

async function publishShipmentMedia(
  db: Database,
  session: Session,
  input: PublishInput,
): Promise<void> {
  await db.transaction(async (tx) => {
    const upload = await tx.findPendingUpload({
      id: input.uploadId,
      tenantId: session.tenantId,
      shipmentId: input.shipmentId,
    });

    if (!upload) throw new NotFoundError("Upload intent not found");

    const updated = await tx.publishShipmentMedia({
      tenantId: session.tenantId,
      shipmentId: input.shipmentId,
      objectKey: upload.objectKey,
      expectedVersion: input.expectedVersion,
    });

    if (!updated) {
      throw new ConflictError("Shipment media changed; reload before publishing");
    }

    await tx.markPublished(upload.id);
  });
}
```

`publishShipmentMedia` should translate to an update whose predicate includes both tenant ownership and the expected version, then increments that version. A zero-row update is a normal concurrency result, not a retry signal. Blindly retrying with the same stale expectation would erase the protection. Return a conflict response and let the user review the newer media before trying again.

Conflicts are data.

Before publication, verify completion using storage metadata or a server-controlled completion signal. Treat the browser's claim as untrusted. Validate type and size against the recorded intent, and run any required scanning or media processing before making the pointer visible. The exact checks depend on the logistics data involved; the invariant does not.

## Operate the protocol, not just the happy path

Test two replacements that start from the same version and complete in reverse order. Exactly one publication should succeed. Also test a cross-tenant upload ID, an expired upload capability, duplicate completion calls, a missing object, and cleanup racing with publication. These cases are more valuable than another unit test of key formatting.

Track counts of created intents, completed uploads, successful publications, conflicts, and abandoned-object deletions. Correlate logs with `tenantId`, `shipmentId`, and `uploadId`, but exclude signed query strings. A rising gap between completed and published uploads points to client exits, validation failures, or publication conflicts. Put an alert on age as well as count; a small pile that never drains is still broken.

Ship the change in two phases. First, write new immutable keys while retaining the existing read path. Then switch the database pointer and reader together. Keep old objects through a defined rollback window. This costs temporary storage, but it buys a straightforward rollback, a better revenue-per-hour trade than trying to reconstruct deleted delivery evidence during support hours.

Cleanup must be idempotent. Select expired, non-published upload intents, recheck that no current pointer references each key, delete the object, and record completion. Lifecycle policies can provide a backstop, yet the application database remains the source of truth for whether an upload was published.

There is a real trade-off: immutable attempts consume more storage and require a cleanup job, while conditional publication adds a version column and a conflict state to the UI. The design is not suitable when every byte must immediately replace a fixed external URL, when the database cannot be the publication authority, or when callers cannot handle a conflict response. In those cases, serialization through a single queue is easier to reason about, though it adds queue latency and another component to operate. For tiny generated artifacts with no audit or rollback value, a stable key may also be the more honest choice. The extra protocol should earn its keep.

## When is the simpler runner-up enough?

Unique keys with an unconditional pointer update can be reasonable when replacement requests are serialized elsewhere, or when product semantics explicitly define "last completion wins." A back-office import queue with one consumer may fit. Document that rule and test it.

A stable object key is defensible for content that is genuinely replaceable and never needs version-aware reads, retention, rollback, or audit. An internal generated thumbnail may qualify. Customer-submitted logistics evidence usually does not: retries, mobile networks, parallel tabs, and retention requirements make identity worth preserving.

The decision rule is short. If losing the user's intended replacement would create support work or weaken an audit trail, pay the small schema cost for immutable keys and conditional publication. Keep locks out of the design, keep bytes out of Node.js, and ship the concurrency rule where every caller must obey it.

## Further reading

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
