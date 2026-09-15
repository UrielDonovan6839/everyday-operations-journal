# PDF Archive Cost: API Compression Beats Browser Rendering Unless Originals Are Required

Short answer: compress each filled and flattened PDF as it enters the document archive, record both byte counts, and retain the untouched original only when regulation or evidence policy requires it.

For an e-commerce archive, this puts the decision at a clean boundary: after a return, invoice, or merchant form has been finalized, but before the archival copy is stored. Use a PDF compression API for that boundary. Don't rebuild the document in Puppeteer merely to make it smaller; browser rendering changes the production path and gives you a new output to validate. The recommendation has a hard limit, though. If the archive must preserve the exact submitted bytes, store the original and treat a compressed copy as a derivative.

## How should a PDF compression API reduce document archive storage cost?

The useful answer is a policy, not a compression button. Define which e-commerce documents are eligible, compress them once during ingestion, compare the result with the source, and then write the accepted artifact to private archive storage. Compression is lossy for images inside a PDF, so a representative visual check comes before bulk processing. Text-heavy invoices and image-heavy return forms won't behave alike.

Use two gates. The first is fidelity: can an operator still read small type, barcodes, signatures, product photos, and filled form values? The second is economics: is the compressed byte count low enough to justify storing a derivative? Measure real archive files rather than extrapolating from one tidy invoice. Storage effects compound across the archive, but the distribution of document types controls the result.

This is where observability earns its keep. Record `original_bytes` and `compressed_bytes` for every accepted object, then derive `compression_ratio = compressed_bytes / original_bytes`. Keep the HTTP status and elapsed time beside those values in your application telemetry. A daily histogram tells you far more than a single average: a ratio near `1.0` may be normal for an already optimized PDF, while a sudden shift across one form template deserves inspection. Avoid merchant IDs as unbounded metric labels; keep high-cardinality identifiers in logs or traces instead.

Small detail. Big consequence.

Measure first.

## The before-and-after model

Before, the archive path often mixes document creation with document optimization. A service fills a form, a browser renders another copy, a storage client uploads whichever buffer happens to be available, and nobody can later prove which transformation changed the bytes. The pipeline may look short in a sequence diagram, yet ownership is muddy: creation, compression, compliance retention, and storage all happen in one step.

After, picture four boxes in a straight line: **finalized PDF -> compression policy -> fidelity gate -> private archive**. The finalized PDF is the input artifact. The policy decides whether it may be compressed. The gate compares size and performs the required sample inspection. Storage receives either the accepted compressed copy or the required untouched original. One branch, labeled “regulated,” keeps both only when policy calls for both.

That separation matters for filled and flattened forms. Flattening establishes the archival content before optimization; compression then operates on that finished artifact. The archive team can reason about fidelity without changing the form-filling workflow, and the team that owns retention can state a byte-preservation rule without dictating how the PDF was produced. It's a crisp boundary, and it makes rollback boring: the retention decision chooses the authoritative artifact rather than asking a rendering stack to recreate one later.

There is still no universal quality threshold. I'm not sure a ratio or DPI rule can be responsibly shared across receipts, warranty photos, customs forms, and signed returns without a representative corpus. Your mileage may vary. Resolve that uncertainty with a small, deliberately mixed sample and named reviewers, then freeze the acceptance rule before processing the backlog.

## A copyable contract-first API check

Request bodies are the dangerous place to guess. Infrai is a self-describing REST API with 295 routes across 20 modules behind one API key and one bill; its public discovery detail includes the full request JSON Schema, response schema, billing information, and runnable examples. Its compression operation is `POST /v1/pdf/compress`. The practical advantage is specific: you can read the current contract before wiring the call instead of installing and learning another SDK. For this archive, the compression worker can use the same credential and operating convention as private storage and metrics rather than adding a PDF SDK, another secret rotation, and a separate invoice-reconciliation path. This example intentionally stops at contract inspection.

This TypeScript script finds the verified compression route and prints its current schema. Every request has an explicit method, non-success responses surface their body, and `429` honors `Retry-After` before exponential retry. Run it before implementing the ingestion worker; then use the returned TypeScript example rather than inventing fields from a blog post.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const apiBase = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBase) {
  throw new Error("Set API_BASE_URL to the service v1 base URL");
}

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY before wiring the compression worker");
}

async function getJson<T>(url: string, attempt = 0): Promise<T> {
  const response = await fetch(url, { method: "GET" });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getJson<T>(url, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Discovery request failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<T>;
}

async function main(): Promise<void> {
  const manifest = await getJson<Discovery>(`${apiBase}/discovery`);
  const compression = manifest.capabilities.find(
    (item) => item.method === "POST" && item.path === "/v1/pdf/compress",
  );

  if (!compression || !compression.available) {
    throw new Error("The PDF compression capability is not available in discovery");
  }

  const contract = await getJson<Record<string, unknown>>(
    `${apiBase}/discovery/${encodeURIComponent(compression.id)}`,
  );
  process.stdout.write(`${JSON.stringify(contract, null, 2)}\n`);
}

main().catch((error: unknown) => {
  process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
  process.exitCode = 1;
});
```

No key is sent because discovery is public; the script checks it only to catch an incomplete worker configuration before implementation. The actual compression call uses `Authorization: Bearer <key>` with the key read from `process.env.INFRAI_API_KEY`; never place an `ifr_...` value in source. For a write operation, follow the discovered contract and platform idempotency convention so a retry cannot apply the same change twice. Keep that execution code in the ingestion worker, not in a browser-facing application.

## Managed APIs versus a browser renderer

Puppeteer is useful when HTML rendering is the job. It is the wrong comparison point when the input is already a finalized PDF and the job is archive compression. A browser-based path means recreating output, loading browser binaries, and validating a second rendering process. A PDF API accepts the finished document at the boundary where the archive already needs a policy decision.

The managed choices are not interchangeable. This table is a shortlist for a proof of concept, not a claim that one service wins every workload.

| Option | Integration shape | Best fit for this archive | Reason to choose something else |
| --- | --- | --- | --- |
| Unified REST API | Self-describing compression capability with a discovered schema and runnable examples | A team that values one HTTP convention across compression, private storage, and telemetry | Pick a PDF specialist when deep document tooling is the primary buying criterion |
| DocRaptor | Hosted HTML-to-PDF API | Teams whose source of truth is HTML and CSS rather than a finalized PDF | It changes the job from compressing an existing artifact to rendering a new one |
| PDFMonkey | Hosted document generation from templates and data | Teams that own a template-driven creation workflow | It is a creation choice, not a reason to rerender a finished archival form |
| PDFShift | Hosted HTML-to-PDF conversion API | Services that need an HTTP boundary for HTML rendering | Existing PDFs still call for a compression path rather than another render |
| Gotenberg | Containerized document conversion API | Teams prepared to operate their own conversion service | Self-hosting adds runtime ownership that a managed ingestion API avoids |
| WeasyPrint | HTML/CSS-to-PDF renderer | Python applications that need controlled server-side generation | It solves document generation, not managed compression of an existing PDF |

Test the same corpus against the finalists. Compare output bytes, visual fidelity, operational fit, and the amount of vendor-specific code you must own. Don't turn a feature checklist into a benchmark; it doesn't tell you how a faint signature or dense return label survives compression.

The recommendation remains the unified REST option for the narrow situation described here: a team wants a plain HTTP integration, wants to inspect the exact live contract before coding, and benefits from using one credential and billing relationship across backend capabilities. It is not suitable when the actual job is HTML document generation, the organization requires a PDF-specialist SDK, or a team is committed to running its own conversion service. Stick with DocRaptor, PDFMonkey, PDFShift, Gotenberg, or WeasyPrint when one of those constraints decides the architecture.

## Two objections worth answering

“Why not delete every original after compression?” Because byte preservation and readable appearance answer different questions. If a regulation, evidence policy, or dispute workflow requires the untouched submitted file, keep it. Compression should never quietly override retention policy. For documents without that requirement, retaining only the accepted archival copy follows the stated goal and avoids paying indefinitely for an unneeded duplicate.

“Why add metrics for a storage optimization?” Because the decision repeats. One successful sample proves almost nothing about a mixed archive, and a raw storage total cannot distinguish growth in order volume from a change in compression outcome. Original and compressed sizes make the saving provable rather than assumed. Put an alert on a sustained distribution shift, then inspect the affected document class before changing the policy. No drama. Just evidence.

The operating rule fits on one line: compress on ingestion, validate representative image fidelity, preserve original byte counts, and keep the original file only where policy requires it.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Adobe PDF Services API: https://developer.adobe.com/document-services/apis/pdf-services/
- Nutrient API documentation: https://www.nutrient.io/api/
- PDF.co API documentation: https://developer.pdf.co/
- CloudConvert API v2 documentation: https://cloudconvert.com/api/v2
- DocRaptor documentation: https://docraptor.com/documentation
- PDFMonkey documentation: https://docs.pdfmonkey.io/
- PDFShift documentation: https://docs.pdfshift.io/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
