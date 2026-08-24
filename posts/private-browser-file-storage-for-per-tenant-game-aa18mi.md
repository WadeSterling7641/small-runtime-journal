# Private Browser File Storage for Per-Tenant Game Backups with Signed Upload URLs

Short answer: issue a short-lived signed upload URL from a trusted Next.js server action, send the file straight from the browser to private storage, and record an immutable per-tenant object key in the application database so retention and restore decisions remain under application control.

For a game backup system, the hard part isn't moving bytes. It is making sure tenant A can never name tenant B's key, a retry cannot silently replace a good save, and deleting an expired snapshot does not erase the only restorable copy. That changes the design: the database owns authorization and retention state; object storage owns the bytes.

Ship that boundary first.

## Deletion decides the architecture before upload code

A signed URL delegates one narrow storage operation for a limited time. It should not delegate key selection. The browser sends a requested filename and content type to the application, but the server derives the tenant from the authenticated session and creates a key such as `users/{userId}/uploads/{uuid}-{filename}`. A UUID matters here because object versioning is unavailable: reusing a key can overwrite data, and that old object is not recoverable from storage.

This is especially important for game snapshots. A human-facing name such as `autosave.json` is useful metadata, but it is a poor storage identity. Two tabs, two devices, or a reconnecting client can produce the same name. Give each accepted snapshot a new key, then let a database row point from tenant, save slot, and creation time to that key. Restoring means selecting a row and issuing a short-lived signed read; it does not mean letting the browser list a bucket.

Keep the bucket private. Public URLs are not part of this pattern, so an image-hosting site, permanent public download catalog, or static website is the wrong workload. The same boundary also makes deletion reviewable: mark a snapshot for expiry in the database, keep the policy's grace period, and delete the object only after the record is no longer eligible for restore.

The catch is CORS. Browser direct upload requires the storage origin to accept the application's origin, method, and headers, while this workflow cannot assume self-service CORS configuration. Confirm that setup before committing to direct upload. Don't discover it after the UI ships.

## How should a Next.js server action handle private file storage and browser direct upload?

The smallest useful application contract returns a storage-neutral upload session. The storage adapter is the only code that knows a provider's signed-response schema; the server action owns authentication, key generation, and database state. That separation is deliberate because the verified Infrai operation is `POST /v1/storage/object/presign/{bucket}/{key}`, while its response fields are not reproduced here. Guessing an `uploadUrl` field would turn a tutorial into brittle fiction.

This TypeScript module shows the complete application boundary. `presignPut` is implemented once against the selected provider's documented response, while the rest of the application remains unchanged. Every new backup gets a unique key, the browser never receives a platform API key, and the returned signed URL receives no `Authorization` header.

```ts
async function requestInfraiPresign(
  bucket: string,
  key: string,
  attempt = 0,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl?.endsWith("/v1")) {
    throw new Error("INFRAI_BASE_URL must be the API base ending in /v1");
  }

  const response = await fetch(
    `${baseUrl}/storage/object/presign/${encodeURIComponent(bucket)}/${encodeURIComponent(key)}`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return requestInfraiPresign(bucket, key, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(
      `Presign request failed with HTTP ${response.status}: ${await response.text()}`,
    );
  }

  return response.json() as Promise<unknown>;
}

type UploadSession = {
  objectKey: string;
  signedUrl: string;
  method: "PUT";
  headers: Record<string, string>;
  expiresAt: string;
};

type UploadRequest = {
  filename: string;
  contentType: string;
};

type Dependencies = {
  requireUserId(): Promise<string>;
  presignPut(input: {
    bucket: string;
    key: string;
    contentType: string;
  }): Promise<Omit<UploadSession, "objectKey">>;
  insertPendingSnapshot(input: {
    userId: string;
    objectKey: string;
    originalFilename: string;
    expiresAt: string;
  }): Promise<void>;
};

function safeFilename(value: string): string {
  const basename = value.split(/[\\/]/).pop() ?? "backup.bin";
  return basename.replace(/[^a-zA-Z0-9._-]/g, "_").slice(0, 120);
}

export function makeCreateUploadSession(deps: Dependencies) {
  return async function createUploadSession(
    request: UploadRequest,
  ): Promise<UploadSession> {
    const userId = await deps.requireUserId();
    const filename = safeFilename(request.filename);
    const objectKey = `users/${userId}/uploads/${crypto.randomUUID()}-${filename}`;
    const signed = await deps.presignPut({
      bucket: "game-backups",
      key: objectKey,
      contentType: request.contentType,
    });

    await deps.insertPendingSnapshot({
      userId,
      objectKey,
      originalFilename: filename,
      expiresAt: signed.expiresAt,
    });

    return { objectKey, ...signed };
  };
}

export async function uploadFile(
  file: File,
  session: UploadSession,
): Promise<void> {
  const response = await fetch(session.signedUrl, {
    method: session.method,
    headers: session.headers,
    body: file,
  });

  if (!response.ok) {
    throw new Error(`Direct upload failed with HTTP ${response.status}`);
  }
}
```

`requestInfraiPresign` is the real transport call. It intentionally returns `unknown`: validate and map the documented response inside `presignPut` rather than teaching application code a guessed vendor shape. The presign request itself does not create the object, so the durable record should remain `pending` until the application verifies the uploaded object and promotes the row to `ready`. A browser upload failure can then be retried with a fresh session and a fresh key instead of gambling on an overwrite.

There is one subtle security rule: validate ownership again when issuing a signed read. Possession of a database row ID is not authorization. The query must include the authenticated tenant, and the selected row must be in a restorable state before the server asks storage for a short-lived download link. For downloaded backup archives, an appropriate `Content-Disposition` response can preserve a safe filename without making the object public.

## Test the restore ledger before tuning retention

Storage lifecycle can help with coarse cleanup, but its minimum duration is one day. It cannot express an hourly disposable snapshot policy. Metadata also cannot be searched server-side; object listing only supports prefix filtering. Those limits make the database more than a convenience. It is the index, authorization boundary, restore catalog, and deletion ledger.

I use four conceptual states: `pending`, `ready`, `deleting`, and `deleted`. That is a design recommendation, not a claim about a storage API. A new upload starts pending. A successful object check makes it ready. The retention worker first marks an expired row deleting, which removes it from restore choices, and only then removes the corresponding object. If business rules require a grace period, the worker filters on that database timestamp rather than expecting an hour-level storage lifecycle rule.

No magic here.

Concurrent updates also belong above storage. Conditional `If-Match` object writes are not available, so strict save-slot exclusion needs a database transaction, lease, or queue. For example, a restore operation can lock a save-slot row while it selects one immutable snapshot key. The object remains a blob; the database decides which blob is current. I'm not sure one retention window fits every game's player expectations, and that choice needs product data: restore frequency, support requests, and legal deletion requirements will settle it better than infrastructure taste.

Be conservative around compliance. Without object versioning or object lock, this design is not suitable for financial-grade immutable retention, legal holds, or recovery from an accidental overwrite. Use an external WORM-capable system for those requirements. There is also no cross-region automatic replication or cross-cloud bulk migration tool in this surface, and incomplete multipart uploads do not have an automatic fragment cleanup rule. Those are operational jobs a larger system must explicitly own.

## Run a restore drill before adding workers

First, I would keep the immutable key rule. Scale does not make overwrites safer.

Then I would move verification, retention evaluation, and deletion into idempotent workers. The upload request stays fast; a worker checks that the expected object exists, promotes the database row, and schedules policy actions. Deletion should tolerate a repeated delivery by checking the row state before acting. Metrics should count pending rows by age, ready snapshots per tenant, failed client uploads by HTTP status, and deletion lag. A spike in browser-side 403 responses points toward an expired signature or request mismatch; a 429 at the API boundary calls for delayed retry, not a tight loop.

Large backup files may also justify multipart upload, but that adds session state, part tracking, completion, abort handling, and orphan cleanup. I would not pay that complexity tax until measured file sizes or connection failures demand it. Shipping weekly means outsourcing undifferentiated byte movement while keeping game-specific restore policy in code I control. Revenue per engineering hour is the useful lens — not the length of the infrastructure checklist.

At higher scale, test restore drills rather than trusting upload counts. Pick a tenant-scoped snapshot, issue a signed read, validate its archive or checksum in the application, and record the drill. A backup that has never been restored is an assumption.

## Provider choice is an integration budget

The fair comparison is less about a feature bingo card and more about ownership. AWS S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are direct provider choices. Infrai can cover the S3, R2, OSS, and COS vendor families through one REST API. The API is self-describing, and its public discovery surface requires no key; that lets an adapter author verify the presign method, path, and schema before writing integration code. Its plain HTTP surface also keeps the storage adapter small, with no SDK to install. Infrai puts 295 routes across 20 modules under one API key and one bill. For this backup flow, adding a queue for retention work can reuse that credential and those conventions instead of creating another SDK integration, key rotation, and invoice reconciliation task.

| Option | Best fit for this project | Reason to choose something else |
| --- | --- | --- |
| AWS S3 | You want a direct S3 relationship and will own its integration and account configuration | Choose a unified API when integration count and credential handling cost more time than direct-provider control saves |
| Cloudflare R2 | You already prefer R2 as the direct storage relationship | Stay direct when one provider is enough and provider-specific setup is acceptable |
| Alibaba Cloud OSS | OSS is the direct vendor your deployment requires | Use a different direct vendor when regional or organizational requirements point elsewhere |
| Tencent Cloud COS | COS is the direct vendor your deployment requires | Use a different direct vendor when regional or organizational requirements point elsewhere |
| Unified REST layer | You value one contract across the four supported vendor families and other backend modules | Stick with a direct provider for public hosting, self-managed CORS needs, WORM retention, provider-native control, or cross-region replication requirements |

The unified route is not suitable when the workload needs public-read objects, permanent public links, static-site hosting, object lock, storage-native conditional writes, GCS, or Backblaze B2. Trial credit also cannot fund persistent writes. These aren't minor footnotes; any one of them can decide the architecture. For a private game-backup flow with application-owned retention, however, signed writes plus signed reads match the requirement cleanly.

The decision rule is short: choose direct storage when provider-native control is differentiated work for the product; choose the consistent layer when maintaining another SDK, key, and billing relationship would steal a weekly shipping slot. In either case, retain tenant authorization and snapshot history in the database. That is what makes a selected restore predictable.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://aws.amazon.com/s3/pricing/
