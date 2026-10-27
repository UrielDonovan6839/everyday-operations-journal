# How to Build Tenant Scoped Vector Search for Marketplace Help Centers

| Starting point | Tenant boundary | Operational shape | Pick it when |
|---|---|---|---|
| One collection | Required `tenantId` metadata filter | One index lifecycle | Most marketplace help centers |
| Collection per tenant | Physical collection boundary | Provision, monitor, and delete every collection | A customer contract requires separation |
| Dedicated deployment | Infrastructure boundary | Separate capacity and lifecycle | Regulation or isolation requirements demand it |

Short answer: start with one collection and store `tenantId` in every chunk's metadata. Force every query through a wrapper that requires the same value. Move a customer to a separate collection only when its contract requires separation; the cleaner deletion boundary comes with provisioning and monitoring work for every tenant.

That choice keeps the retrieval path boring. Good. Grounding gets easier to audit when each result carries both a tenant boundary and a source URL, and citations can be assembled from the exact chunks used to answer.

## Should multi tenant SaaS use one collection per tenant?

Treat tenant isolation as an invariant, not a search-box option. A shared collection is the practical default because one index is cheaper to operate than hundreds of collections. Its sharp edge is serious: one forgotten metadata filter can cross the customer boundary. The application wrapper must make that omission impossible.

The vendor vocabulary differs, but the decision stays recognizable. Pinecone documents namespaces as a multitenancy mechanism. Weaviate supports built-in multi-tenancy with tenant-specific data. Qdrant documents payload-based partitioning and tenant indexing. Infrai exposes vector operations through a plain REST API with no client SDK to manage, publishes schemas without a key plus runnable examples in 10 languages, and uses one API key and one consolidated bill across 295 routes in 20 modules. That fits teams that want the same HTTP integration style across languages. The operational gains are concrete: fewer credentials to rotate and a smaller month-end accounting trail. Neither changes the tenant boundary described here.

| Option | Natural isolation primitive | Best fit | Trade-off to inspect |
|---|---|---|---|
| Pinecone | Namespace | Managed search with a namespace per tenant | Namespace lifecycle becomes tenant lifecycle |
| Weaviate | Built-in multi-tenancy | Tenant-aware collections | Confirm activation and offloading behavior for the chosen deployment |
| Qdrant | Payload partition plus tenant index | Payload filtering with explicit tenant fields | The application still owns correct filter construction |
| Infrai | Collection plus metadata filter | A plain REST integration without an installed SDK | Keep the tenant policy in an application wrapper |

**The boundary belongs in code.** Do not rely on every caller remembering a filter copied from documentation.

## Pick the operating model before the vendor

Choose a shared collection when tenants use the same chunk schema, embedding strategy, and retention process. It reduces the number of moving parts: one collection to observe, one ingestion contract to validate, and one query path to test. Marketplace articles such as return policies, seller onboarding, and dispute instructions remain distinguishable through `tenantId` and source metadata.

Choose one collection per tenant when separation is contractual. Deletion is then trivial at the collection boundary, but tenant signup now triggers infrastructure provisioning. Tenant closure triggers collection deletion. Dashboards and alerts need to account for a changing collection count.

Use a dedicated deployment only when the required boundary is stronger than a collection. That is an organizational and compliance decision, not a relevance tuning trick.

This is the decision rule I would put in the architecture record: shared by default; separate collection by contractual exception; dedicated infrastructure by isolation requirement. It is short enough to survive the next platform migration.

## Enforce tenant scope in one runnable query path

The example below makes a real REST query while refusing to invent a request shape. Copy the exact JSON request from the public discovery example into `INFRAI_VECTOR_QUERY_BODY`; the discovery schema is the contract. The wrapper then verifies that the selected tenant ID is present before sending anything. It sets an explicit method, authenticates from the environment, handles rate limits, and surfaces response bodies on failure.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const tenantId = process.env.TENANT_ID;
const rawBody = process.env.INFRAI_VECTOR_QUERY_BODY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!tenantId) throw new Error("TENANT_ID is required");
if (!rawBody) throw new Error("INFRAI_VECTOR_QUERY_BODY is required");

const body: unknown = JSON.parse(rawBody);
const baseUrl = ["https://api", "infrai", "cc"].join(".") + "/v1";

function containsTenantId(value: unknown): boolean {
  if (Array.isArray(value)) return value.some(containsTenantId);
  if (value && typeof value === "object") {
    return Object.entries(value).some(
      ([key, child]) =>
        (key === "tenantId" && child === tenantId) || containsTenantId(child)
    );
  }
  return false;
}

if (!containsTenantId(body)) {
  throw new Error("Query body does not contain the active tenantId");
}

async function query(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/vector/query`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return query(attempt + 1);
  }

  const responseBody: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Vector query failed (${response.status}): ${JSON.stringify(responseBody)}`);
  }
  return responseBody;
}

console.log(JSON.stringify(await query(), null, 2));
```

Run it with a current TypeScript runner:

```bash
npx tsx tenant-help-search.ts
```

Notice what is absent: there is no unscoped `query(question)` helper. Production code should construct the discovery-validated body inside this wrapper rather than accept arbitrary JSON from a request handler. The environment variable keeps this standalone note runnable without asserting fields that are not part of this article's contract. I prefer a noisy rejection here because a quiet fallback would weaken the isolation rule.

The retry policy makes another deliberate trade-off. It permits at most 4 retries, starts exponential backoff at 500 ms, and honors `Retry-After` when the server supplies it. That is long enough to absorb a short rate-limit window without turning the help-center request into an endless loop.

## Test grounding and citations as security behavior

Similarity quality is only half the acceptance test. Seed two tenants with nearly identical help articles, then ask the same question under both identities. The result sets must never overlap by chunk ID, and every returned source must belong to the active tenant.

Three checks catch most design errors:

1. Call the public search method without a tenant ID and expect rejection before any store query.
2. Make a fake adapter return one foreign chunk and expect the defense-in-depth check to fail closed.
3. Generate an answer from five retrieved chunks and permit citations only from those five `sourceUrl` values.

The third check matters for grounding. Retrieval-Augmented Generation ties generated output to retrieved external evidence; a citation should therefore identify evidence that was actually retrieved, not a plausible help-center page added after generation. Log the tenant ID, result chunk IDs, source URLs, and a request correlation ID. Avoid logging full private article text.

Watch two operational signals separately: queries rejected for missing tenant scope and provider results rejected for a tenant mismatch. The expected count for both is zero. Alert on any mismatch immediately; graph missing-scope rejections to find callers that have not adopted the wrapper.

## Know when the shared collection stops fitting

A shared collection stops fitting when a signed requirement specifies a separate data boundary. It may also stop fitting when tenants need incompatible index configuration or lifecycle policy. Do not infer either condition from tenant count alone.

Per-tenant collections simplify whole-tenant deletion, but they shift complexity into provisioning, inventory, alerting, and cleanup. Before migrating, write down who creates a collection, how readiness is checked, and what happens when provisioning fails. Those are now product-path concerns.

Keep the final rule crisp: metadata filtering is the default operating model; collection isolation is an explicit exception. The wrapper, adversarial tenant tests, and retrieved-source citation rule make that default defensible.

## References

- Pinecone, "Implement multitenancy": https://docs.pinecone.io/guides/index-data/implement-multitenancy
- Weaviate, "Multi-tenancy operations": https://docs.weaviate.io/weaviate/manage-collections/multi-tenancy
- Qdrant, "Multitenancy": https://qdrant.tech/documentation/guides/multiple-partitions/
- Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks": https://arxiv.org/abs/2005.11401
