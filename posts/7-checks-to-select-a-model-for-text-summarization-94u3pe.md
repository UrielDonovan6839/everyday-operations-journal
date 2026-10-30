# 7 Checks to Select a Model for Text Summarization API in Node.js

Short answer: choose the summarization model that preserves the hiring rubric under your real input limits, then prove that choice with per-tenant traces, quality checks, and a retry policy. For a gaming studio scoring candidates, the output is an evidence packet, not merely a shorter resume. The runtime should show which tenant submitted the document, which rubric version shaped the result, what the request consumed, and why a summary was rejected.

Before, a Node.js worker sends text to a model and stores a paragraph. After, it records a bounded request, validates a structured result, attaches usage and tenant dimensions, and routes uncertain work to review. The model is one replaceable part of that pipeline.

## 1. Start With the Rubric, Not the Model

A gaming company may score a candidate against live-service operations, multiplayer systems, or release coordination. Those criteria need evidence, not adjectives. “Strong communicator” is a weak target. “Describes incident communication, player impact, and follow-up ownership” can be tested.

Write the rubric as data before selecting a model. Give every criterion an identifier and an instruction about missing evidence. A missing claim should remain missing. The summarizer must not turn a quiet resume into a confident score.

Include short resumes, long portfolios, repeated project descriptions, mixed date formats, and documents that say nothing about a criterion in the evaluation set. Keep reviewed fixtures while changing the model, prompt, or truncation rule. That makes a model choice a repeatable experiment.

Here is a generic boundary. The endpoint is pseudonymous because the contract matters more than a vendor SDK.

```ts
type Criterion = {
  id: string;
  description: string;
};

type SummaryRequest = {
  tenantId: string;
  rubricVersion: string;
  criteria: Criterion[];
  document: string;
};

type CriterionResult = {
  criterionId: string;
  evidence: string[];
  score: number | null;
};

type SummaryResponse = {
  results: CriterionResult[];
  usageUnits: number;
  requestId: string;
};

async function scoreCandidate(input: SummaryRequest): Promise<SummaryResponse> {
  const response = await fetch('https://summary.example.invalid/v1/ai/batch/submit', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify(input),
  });

  if (!response.ok) {
    throw new Error('summary request failed: ' + response.status);
  }

  return (await response.json()) as SummaryResponse;
}
```

The null score is deliberate. It represents insufficient evidence instead of forcing a precise-looking number. A TypeScript cast does not validate untrusted JSON at runtime, so the caller still needs schema validation.

## 2. How Should Node.js Teams Select a Model for Text Summarization APIs?

Measure the path, not just the answer. A useful trace connects tenant, job, rubric version, request ID, latency, retry count, input size, output size, and usage units. Keep document text out of ordinary metric labels. High-cardinality labels make dashboards noisy and can create a privacy problem.

Per-tenant cost visibility is a join, not a single magic field. Store usage with the request record, store tenant and rubric metadata beside it, and aggregate later. If an upstream API exposes usage, preserve its units and response request ID. If it does not, record the input and output measurements your contract defines and label them as estimates. Do not silently call an estimate a bill.

The operational picture should answer three questions quickly: did this tenant receive a result, was the result acceptable, and what resource usage did that attempt report? Logs answer the individual case. Metrics show the shape. Traces connect the two.

A compact event is enough to start:

```ts
type SummaryEvent = {
  event: 'candidate_summary_completed' | 'candidate_summary_rejected';
  tenantId: string;
  rubricVersion: string;
  requestId: string;
  usageUnits: number | null;
  latencyMs: number;
  reason?: string;
};

function emitSummaryEvent(event: SummaryEvent): void {
  process.stdout.write(JSON.stringify(event) + '\n');
}
```

The event has no resume text. That is useful in a hiring workflow where the source document may contain personal information. Define retention and access rules outside the model call; observability is not permission to copy every payload everywhere.

The queue does not get to decide the rubric.

## 3. How Do You Catch Summary Drift?

A summary can be fluent and still fail the rubric. Test evidence coverage, unsupported claims, missing criteria, and stable handling of long inputs as separate checks. One blended quality score hides the failure you need to investigate.

I initially treated a low score as the only alert. That made debugging slow. A candidate could receive a plausible summary with one criterion silently omitted while the aggregate score looked healthy. The better rule is a gate: reject output when a required criterion is absent, evidence cannot be traced to the input, or a score falls outside the contract.

Retries need the same discipline. Retry transient transport failures with a bounded attempt count and backoff. Do not retry a schema failure forever; that turns bad output into duplicate usage and hides a contract problem. Record attempt number and final disposition so a tenant's usage can be explained later.

```ts
function validateResult(
  result: SummaryResponse,
  criteria: Criterion[],
): string | null {
  const expected = new Set(criteria.map(({ id }) => id));
  const received = new Set(result.results.map(({ criterionId }) => criterionId));

  if (received.size !== expected.size) return 'criterion coverage mismatch';
  for (const criterionId of expected) {
    if (!received.has(criterionId)) return 'missing criterion: ' + criterionId;
  }

  for (const item of result.results) {
    if (item.score !== null && (item.score < 0 || item.score > 5)) {
      return 'invalid score: ' + item.criterionId;
    }
  }

  return null;
}
```

This does not prove that evidence is true. It creates a clean handoff for human review and makes regressions visible. Sampled review remains necessary for grounding, rubric interpretation, and sensitive decisions.

## 4. Which Trade-Off Actually Decides the Runtime?

The deciding trade-off is the failure you can detect and recover from. A larger context limit may help with a portfolio, but only if truncation policy, latency budget, and usage accounting remain legible. A faster model may help a queue, but only if the evidence gate does not degrade. Structured output may simplify storage, but only if rejected output has a clear path.

Evaluate candidates with the same prompt, rubric version, fixtures, and acceptance gates. Compare quality dimensions separately from runtime dimensions. Keep the adapter narrow so the worker depends on your contract rather than a provider-specific response shape. That makes a later switch a controlled experiment instead of a rewrite.

For a multi-tenant gaming workflow, make the decision rule explicit: select the lowest-complexity option that clears evidence and coverage gates on the reviewed set, stays within the latency budget, and reports usage at the same accounting boundary for every tenant. It is a boring rule. Boring rules are easier to audit.

Use a five-point score only when the rubric can support five distinct levels. Otherwise, a smaller scale with an explicit unknown state is easier to defend, especially when two reviewers disagree about one sentence of evidence.

Two objections are common. “Can’t I pick the model with the best public benchmark?” Only as a starting signal; resume structure, rubric wording, and missing evidence can change the result. “Can’t I just log the final text?” You can, but that will not explain a timeout, retry, missing criterion, or tenant-level usage spike. Capture the event around the text, with privacy controls.

The model is replaceable. The rubric contract, acceptance gates, and accounting trail are the durable engineering work.

## Further reading

- LangChain ChatOpenAI integration documentation: https://python.langchain.com/docs/integrations/chat/openai/
- OpenTelemetry observability concepts: https://opentelemetry.io/docs/concepts/observability-primer/
- MDN Fetch API: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
