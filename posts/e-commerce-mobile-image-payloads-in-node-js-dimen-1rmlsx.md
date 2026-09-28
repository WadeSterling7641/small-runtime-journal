# E-commerce Mobile Image Payloads in Node.js: Dimension Before Encoding

Short answer: for an e-commerce thumbnail uploaded from a phone, resize to the largest display dimensions first, then compress that smaller image; do it at upload when the catalog needs predictable delivery, and keep an on-demand path for sizes you cannot predict yet.

I run a one-person SaaS, so my unit of measurement is revenue per hour. A thumbnail that makes a product page slow costs support time and conversions; a pipeline that takes a week to operate costs a feature. The useful choice is not “which encoder wins a benchmark?” It is where pixel work happens, and what gets stored after it happens.

## The upload constraint that changes the design

Phone cameras routinely produce an image much larger than a product card needs. Keeping the original for zoom is sensible, but sending those original pixels to every 160-pixel card is wasteful. Compression cannot recover that work. It still has to inspect and encode all the pixels you gave it.

My baseline is a two-object upload: preserve the original under a private key, and write a bounded thumbnail beside it. The bound is a contract, not a guess: `maxWidth` and `maxHeight` come from the layouts the storefront actually renders. A square grid might ask for 320 by 320; a listing detail page might ask for 960 by 960. The resize step keeps aspect ratio and never enlarges a small source.

That order also makes retries boring. A worker can hash the source, derive a deterministic thumbnail key, and safely run the same operation again. If the key already exists with the expected dimensions and content hash, the worker skips the encode. Boring is good when I'm the on-call team.

Measure it.

## Should mobile image payloads resize before compression for predictable delivery?

Yes, when the dimensions are known. Resize first, then choose a format and quality for the reduced pixel count. The browser still negotiates formats with `Accept`, but it should not be asked to download a four-megapixel answer for a thumbnail slot.

The word “predictable” has two parts. First, the byte budget is less sensitive to the phone camera's source resolution. Second, every responsive slot has a known maximum width, so its cache key and content-length distribution are measurable. Compression quality remains a trade-off: too high and bytes rise; too low and edges around packaging or text turn mushy. I sample real catalog images, not a synthetic gradient, before setting a default.

Here is the smallest worker shape. The image library is intentionally an application boundary; the queue and storage adapters can be replaced without changing the decision rule.

```ts
type SourceImage = {
  bucket: string;
  key: string;
  contentType: string;
  sha256: string;
};

type ThumbnailSpec = {
  name: "card" | "detail";
  maxWidth: number;
  maxHeight: number;
  quality: number;
};

type ImageProcessor = {
  resize(input: Uint8Array, width: number, height: number): Promise<Uint8Array>;
  encodeJpeg(input: Uint8Array, quality: number): Promise<Uint8Array>;
};

export async function makeThumbnail(
  source: SourceImage,
  spec: ThumbnailSpec,
  storage: { get: (bucket: string, key: string) => Promise<Uint8Array>; put: (bucket: string, key: string, body: Uint8Array, contentType: string) => Promise<void> },
  images: ImageProcessor,
): Promise<string> {
  const input = await storage.get(source.bucket, source.key);
  const resized = await images.resize(input, spec.maxWidth, spec.maxHeight);
  const encoded = await images.encodeJpeg(resized, spec.quality);
  const key = `thumbs/${source.sha256}/${spec.name}.jpg`;
  await storage.put(source.bucket, key, encoded, "image/jpeg");
  return key;
}
```

The function does not claim that every source becomes exactly `maxWidth` by `maxHeight`; a portrait image should remain portrait. Store the resulting width and height in metadata. That makes layout bugs observable instead of turning them into a guessing game in the client.

## How do upload-time and on-demand thumbnails fail differently?

Upload-time processing spends CPU before the seller sees a published listing. That is the right friction when a catalog promises that every card is ready from the first page view. It is a poor fit when sellers upload ten rarely viewed originals and the product has dozens of experimental breakpoints. You pay for variants nobody requests.

On-demand processing moves that cost to the first request for each size. It works well with a cache and a queue: the first miss returns a placeholder or waits within a bounded budget, then later requests hit the generated object. The catch is a cold-cache spike after a campaign, plus duplicate work if the cache key is not canonical. A query string such as `?w=319` and `?w=320` should not create two almost-identical jobs if the UI only supports a small set of widths. I once chased a burst that looked like a CDN problem; the real cause was a marketing email using three width spellings for the same card. Every spelling missed the cache, queued a resize, and made the p95 look like a storage outage. Canonicalizing widths to the supported set fixed the shape of the graph before I changed any infrastructure.

I use a hybrid rule. Generate the two contractual catalog sizes at upload; generate unusual editorial sizes on demand and expire them by access age. The upload job records a status, so the listing can remain in “processing” without pretending a missing asset is a successful delivery. A retry uses the source hash and spec name as its idempotency key.

This is the part that saves a solo operator a weekend: keep the policy in one function, and test that function with portrait, landscape, tiny, and very large fixtures. Do not scatter width decisions across controllers, templates, and a CDN rule.

## What should a Node.js thumbnail pipeline measure before shipping?

I track p50 and p95 bytes by slot, resize latency, queue age, cache-hit ratio, and the percentage of originals that fail validation. I also record the output dimensions and format. A single average hides the seller who uploaded a transparent PNG with a six-megapixel canvas.

The input gate is small but important. Read the declared media type and the decoded dimensions; do not trust a filename extension. Reject images above an explicit pixel or byte ceiling before handing them to a decoder. Strip metadata that the storefront does not need, especially location metadata. Keep the original in a restricted store if customers may later need the unmodified file.

When a thumbnail is regenerated, keep the processing version in its key or metadata. That gives me a way to compare an encoder change without silently mixing old and new bytes in one cache. I am not sure one quality number transfers across clothing photos, jewelry, and screenshots; your mileage may vary, so the fixture set should reflect the catalog rather than a stock-photo sample.

A short production checklist:

- Bound pixels before decoding and bound bytes before storing.
- Make keys deterministic from source hash and named dimensions.
- Serve an explicit content type and a long cache lifetime for immutable keys.
- Alert on queue age, not only queue length.
- Keep a fallback thumbnail for a failed optional variant.

## Where this approach is not suitable

The recommendation is not suitable when customers demand arbitrary crop coordinates, per-request watermarks, or edits that must appear instantly at every size. Keep a high-quality original and an image transformation service for those workflows; upload-time variants alone will become a product constraint.

It is also the wrong default for a private archive where images are opened once a year. Generate on demand and accept a cold miss. Stick with upload-time generation when the storefront's first render is the business-critical path and the set of sizes is small enough to name.

The implementation choice should follow that boundary, not a vendor's feature list. Outsource the undifferentiated pixel work, keep the dimension policy and metadata in your code, and ship the catalog path weekly with measurements attached.

## References

- MDN Media formats guide: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
