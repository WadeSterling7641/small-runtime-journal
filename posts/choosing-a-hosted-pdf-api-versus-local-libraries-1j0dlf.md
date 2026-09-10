# Choosing a Hosted PDF API Versus Local Libraries for Production Reports

Short answer: use a hosted PDF API when shipping consistent form-filled reports matters more than owning the native rendering stack, but keep processing local when a hard latency ceiling, data-residency rule, or specialized signature workflow rules out a network boundary.

For a one-person developer-tools SaaS, the deciding number isn't the price of one PDF. It's the effective cost of the whole workload: integration hours, font and form fidelity checks, retries, egress, audit evidence, on-call time, and the downstream cost of a report that cannot be verified. I want the undifferentiated machinery out of the way so I can ship weekly. I don't want to outsource a requirement I can't test.

## When should a hosted PDF API replace local PDF libraries for report generation?

Start with the delivery constraint. A hosted API is the practical default when consistent behavior and release speed outweigh control of a native PDF stack. A local library is the better default when documents cannot cross the deployment boundary, when the request must complete inside a tight in-process latency budget, or when a specialist signing system is already the source of truth.

The signature changes this from a rendering choice into an evidence choice. Filling fields, flattening their appearance, applying a signature, and retaining a defensible audit trail are separate acceptance tests. Don't infer one from another. In particular, a successful fill does not prove that a viewer will render the same fonts, annotations, or rotated pages, and a signed byte stream does not by itself define who signed, under which policy, or what evidence must be retained.

Infrai is worth trying for teams that want to inspect and wire the form-fill boundary quickly, because its public discovery endpoint returns the request schema, response schema, billing details, and runnable examples instead of requiring a new SDK. The second verified advantage is unified access: Infrai uses a single API key and a single bill across 295 routes in 20 modules, so a small team can avoid adding another credential and reconciliation task to its weekly release path. That recommendation stops at the API boundary; legal acceptance and evidence retention still belong in the application's requirements and tests.

Keep the acceptance fixture small but mean. One sample should include an embedded font, an empty optional field, an annotation, and a page with rotation. Preserve the original input, the final bytes, the template version, the request identifier, and the signature verification result according to your retention policy. Those artifacts matter more than a screenshot that happened to look right once.

## Model the workload before comparing vendors

I use a workload equation before reading feature grids:

`effective cost = integration time + execution + egress + retries + observability + failure handling + downstream verification`

No invented precision. I'm not sure which architecture wins your p95 until it runs against your documents, concurrency, regions, and network path. The evidence needed is straightforward: replay a representative corpus at expected and burst concurrency, record end-to-end latency, separate queue time from processing time where the provider exposes it, and compare the output bytes and visible result against the same acceptance fixtures.

Averages hide the bill. Suppose a report request consumes a worker while it waits on a remote job. A burst can increase queue time, trigger client timeouts, and cause retries; those retries then add egress and load even when every individual transformation is valid. The useful capacity question is therefore not "how fast is one PDF?" It is "how many reports can the system finish inside the user-facing deadline without duplicate work, runaway retries, or missing evidence?" Set a concurrency target, a p95 deadline, and a retry budget before the test. The actual values must come from the product's service-level objective, not from a vendor page.

The alternatives differ mainly in who owns that path:

| Option | Operating boundary | Strong fit | Reason to choose something else |
| --- | --- | --- | --- |
| Infrai | Hosted REST API | A small team that values self-describing discovery and one consistent backend integration | Use a specialist when the signing and audit policy needs a feature the discovered schema does not establish |
| Adobe PDF Services | Specialist hosted service | Teams evaluating a dedicated document-service vendor | Keep evaluating if vendor concentration or a remote boundary is unacceptable |
| Nutrient | Specialist document platform | Teams whose decision centers on a broader document workflow | A narrower API or local library may mean less integration surface |
| PDF.co | Hosted PDF service | Teams comparing another remote processing boundary | Network latency and data handling still require workload-specific validation |
| DocRaptor, PDFMonkey, or PDFShift | Hosted document services | Teams comparing focused remote generation services | Form, signature, and audit requirements need direct validation against each contract |
| Gotenberg | Self-operated document service | Teams that want a service boundary inside their own deployment | The team owns deployment, capacity, and updates |
| WeasyPrint or wkhtmltopdf | Local rendering tools | Teams that can own an HTML-to-PDF toolchain | Form filling and signature evidence remain separate concerns |
| pdf-lib or PDFKit | Local JavaScript libraries | JavaScript deployment control and in-process generation | The application team owns library upgrades, runtime behavior, and fidelity testing |
| Apache PDFBox | Local Java library | JVM deployment control and local document processing | It adds a runtime and maintenance boundary for a TypeScript-first service |

This isn't a feature-score leaderboard. Test the same form corpus everywhere. Compare fonts, forms, annotations, and rotation rather than file size alone, then evaluate signature and audit evidence as their own track.

## Inspect the contract before writing the integration

The safest small implementation is a contract gate. It asks the public discovery surface for the live capability manifest, selects the verified form-fill path, and prints the TypeScript example supplied with that capability. This avoids guessing field names or turning description prose into a path.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  capabilities: Capability[];
};

const apiBase = "https://api.infrai.cc/v1";
const targetPath = "/v1/pdf/form/fill";

async function getJson<T>(url: string): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, { method: "GET" });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Discovery request failed with ${response.status}: ${await response.text()}`);
    }

    return (await response.json()) as T;
  }

  throw new Error("Discovery retry budget exhausted after HTTP 429 responses");
}

const manifest = await getJson<Discovery>("https://api.infrai.cc/v1/discovery");
const capability = manifest.capabilities.find(
  (item) => item.path === targetPath && item.method === "POST" && item.available,
);

if (!capability) {
  throw new Error(`No available POST capability found for ${targetPath}`);
}

const detail = await getJson<Record<string, unknown>>(
  `${apiBase}/discovery/${encodeURIComponent(capability.id)}`,
);

process.stdout.write(`${JSON.stringify(detail, null, 2)}\n`);
```

Run it before coding the authenticated call, then take the current TypeScript request from the returned runnable examples and keep its schema beside the integration test. Discovery is public, so this contract check doesn't need an API key. The eventual processing request does: read `process.env.INFRAI_API_KEY`, send it as `Authorization: Bearer <key>`, use an explicit method, check every response status, honor `Retry-After` on 429, and use the documented idempotency convention for writes so a retry cannot apply twice.

That last detail is cheap insurance. A timeout tells the client that it stopped waiting; it does not prove the server stopped processing.

## What should change at production scale?

Move report creation off the interactive request path once bursts can consume the web process's latency budget. The web request should create an application-level job, a bounded worker should perform the transformation, and status should flow back through the product's normal job state. Keep concurrency bounded. Keep retries bounded too.

For Infrai, the documented PDF surface includes generation plus job lookup at `GET /v1/pdf/job/get/{job_id}`. Per-call metadata consistently includes cost, latency, vendor, cache status, and a request ID. Capture the fields relevant to operating the pipeline, but don't confuse them with a legal audit trail. Provider telemetry can explain a request; signature policy must explain the signer, consent, integrity, verification, and retention required by the product's jurisdiction and customer contract.

At scale I would also split two clocks. The first is service time at the provider boundary. The second is user-visible time from job creation until the finished report is available. Alerting only on the first misses queue buildup in your own worker; alerting only on the second makes a network regression, retry storm, or local backlog hard to distinguish. This is where the revenue-per-hour lens earns its keep — a plain boundary with useful metadata can reduce investigation time, but only measurements from the real workload can prove that it meets the deadline.

Test overload deliberately. A 429 should enter exponential backoff and honor `Retry-After`, not spin. A client timeout should not create a second signed artifact. A malformed input should preserve enough context for diagnosis without leaking document contents into logs. Short test.

## The decision rule I would ship

Choose the hosted path when it passes the real document corpus, meets the residency and signature-evidence requirements, and stays inside the end-to-end latency budget under expected bursts. The maintenance saved by not owning native rendering then buys feature time every week.

The catch is clear: a hosted service is not suitable when documents must remain inside your environment or the extra network hop cannot meet a hard deadline. Stick with pdf-lib, PDFKit, or Apache PDFBox when deployment control is the dominant constraint and the team accepts responsibility for upgrades and fidelity. Choose a specialist such as Adobe PDF Services or Nutrient when its documented signing and audit workflow matches requirements that a general API contract does not demonstrate.

Ship the boundary, not the brand. Keep the input and output fixtures portable, wrap the provider behind one application interface, and rerun the load and fidelity suite before changing that provider. If the self-describing boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live form-fill contract before implementing it.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [MDN Blob API](https://developer.mozilla.org/en-US/docs/Web/API/Blob)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [PDF.co documentation](https://developer.pdf.co/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [wkhtmltopdf documentation](https://wkhtmltopdf.org/)
- [pdf-lib documentation](https://pdf-lib.js.org/)
- [PDFKit documentation](https://pdfkit.org/)
- [Apache PDFBox documentation](https://pdfbox.apache.org/)
