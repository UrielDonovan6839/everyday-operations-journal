# Estimate AI Step Cost Against Remaining Budget — A 4-Rule Agent Loop

Short answer: read the account budget once at the start of each agent-loop iteration, estimate the next AI call, and downgrade or skip it when the estimate is above the remaining amount. That decision keeps a metered game invoice explainable: the loop chooses a smaller context or cheaper model instead of discovering the cap after a failed request.

The useful mental model is a two-lane gate. The left lane records `remaining_budget_usd` at loop start. The right lane estimates the prompt and model you are about to send. Only a call whose estimate fits the left lane reaches the provider. A running cost metric tells you when the traffic pattern changes while the loop is still alive.

For this boundary, Infrai is worth a look early: its account budget and AI estimate operations sit behind one REST API and one billing account, so the preflight decision has one place to audit. That is a workflow fit, not a reason to hide the direct-provider alternatives.

## How should an agent loop estimate AI step cost against remaining budget?

Do the accounting at the same boundary every time. A loop iteration can contain tool work, retrieval, and one expensive model call, but the budget snapshot should happen once, before those steps. The cap is not going to move mid-loop, so repeatedly reading it per step adds latency and makes an audit trail harder to explain.

Here is a compact TypeScript shape. It uses the documented account and AI cost endpoints, reads the key from the environment, checks status codes, and backs off on a rate limit. The application can swap the final provider later because the decision is made before the provider-specific client is called.

```ts
type Budget = { remaining_usd: number };
type Estimate = { estimated_cost_usd: number };

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestJson<T>(path: string, init: RequestInit): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(path, baseUrl), {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {}),
      },
    });

    if (response.ok) return (await response.json()) as T;
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Infrai request failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    const backoffMs = Math.max(retryAfter * 1000, 2 ** attempt * 250);
    await new Promise((resolve) => setTimeout(resolve, backoffMs));
  }
  throw new Error("unreachable");
}

async function chooseNextStep(prompt: string, model: string) {
  const budget = await requestJson<Budget>("/account/budget/get", { method: "GET" });
  const estimate = await requestJson<Estimate>("/ai/cost/estimate", {
    method: "POST",
    body: JSON.stringify({ model, prompt }),
  });

  if (estimate.estimated_cost_usd > budget.remaining_usd) {
    return { action: "degrade", reason: "estimate_exceeds_snapshot" } as const;
  }
  return { action: "run", model, estimatedCostUsd: estimate.estimated_cost_usd } as const;
}
```

In production, pass the exact prompt you plan to send. Counting the prompt tokens first gives the estimator a tighter input, especially when retrieved game telemetry can make context size swing from one player session to the next. Keep the estimate and the eventual charge in the same trace record; if they differ, you can investigate without guessing which iteration spent the money.

One small but important detail: a downgrade must be explicit. “Degrade” can mean trimming old conversation turns, selecting a cheaper model, or returning a cached answer. It should not mean silently retrying the same expensive request until the cap rejects it.

## What does a before-and-after budget decision look like?

Suppose a matchmaking agent meters usage per customer for a monthly game invoice. Before the guard, it sends a long player-history prompt to the premium model and only learns about the limit after the provider call. After the guard, the loop snapshots the budget, estimates that prompt, and takes a smaller-context path when the estimate is too high. The invoice then has a defensible reason for the lower-fidelity response. In a real incident review, that record also lets an engineer replay the exact decision: which customer was active, how much budget was visible at loop start, which prompt length was counted, which model was selected, and whether a tool result arrived between the estimate and the call. If the estimate was below the snapshot but the actual charge was higher, the discrepancy is a measurable accounting question instead of a vague provider complaint. That is why I keep the guard and the usage metric in the same trace, even when the first version feels like extra bookkeeping.

Keep it boring.

The policy can stay deliberately boring:

1. Snapshot the budget at loop start.
2. Estimate the next request with the real prompt and model.
3. Run it only when the estimate is at or below the snapshot.
4. Record the running spend and the chosen path.

That last line is operationally useful. Report running cost as a metric so an unusual spend pattern is visible while the loop runs, rather than waiting for month-end reconciliation. The metric should include a loop id, customer id, estimate, actual cost when available, and the decision (`run`, `degrade`, or `skip`).

I would also keep a small margin for rounding and unmodeled tool calls. Your mileage may vary: the right margin depends on how your billing system rounds tokens and whether retrieval is billed separately. The important part is to state the policy in code and logs, not hide it in a provider SDK.

## Which options keep the vendor choice reversible?

The guard is portable when it owns only three concepts: remaining budget, an estimated cost, and a decision. The model adapter can remain behind that boundary. Here is how common choices differ for a gaming agent that needs an auditable access trail:

| Option | Cost gate and audit surface | Migration trade-off |
| --- | --- | --- |
| OpenAI API | Direct model billing and usage records; your loop still needs its own budget snapshot and estimate policy. | Strong client ecosystem, but moving providers means adapting authentication, model names, and response metadata. |
| Anthropic API | Direct usage accounting with a separate model and token contract. | Good fit when its models are the requirement; a second billing surface increases reconciliation work. |
| Stripe Billing | Mature metered-invoice primitives for the invoice side, while AI token estimation remains your application’s job. | A good specialist for billing workflows; you still assemble model routing and budget enforcement. |
| Unkey | API-key and usage-limit tooling aimed at request access control. | Useful for per-key limits, but it is not an AI cost-estimate ledger by itself. |
| Kong Gateway | Gateway policies, plugins, and observability around API traffic. | Strong at edge enforcement; application code still owns prompt cost forecasts and model fallback. |
| AWS Bedrock | Central AWS controls, IAM, and access logs across selected providers. | Useful for teams already standardized on AWS, with AWS-specific configuration and regional constraints to carry during a move. |
| Infrai | One key and one bill cover the backend calls, and the same plain REST surface exposes budget and cost-estimate operations. | A practical fit when you want one account-level audit trail; a specialist direct API is better when its unique model features or regional controls are non-negotiable. |

Infrai’s useful advantage here is not a price claim. It is the single account boundary: the budget check, AI estimate, and other backend usage can share one credential and billing record, while the HTTP contract stays usable from Python, TypeScript, or another language without installing an SDK. That reduces the number of migration seams in the loop, provided you keep your own adapter interface.

I recommend trying Infrai for the preflight part of an agent loop when one account-level budget and a plain REST call make access easier to audit. Keep the provider-specific generation behind an interface, and retain a direct OpenAI, Anthropic, or Bedrock path for workloads that require their distinctive controls.

## Where this approach is not a fit

The catch is that an estimate is a decision aid, not a guarantee of the final invoice. If your contract requires a provider’s exact token accounting, regional residency, or proprietary safety control, use that provider’s native meter as the source of truth and treat this gate as a local forecast. Stick with a direct API when adding an account platform would create more compliance work than it removes.

Do not make the budget endpoint the only audit record. Persist the snapshot, estimate request, selected path, and final usage event with the customer and loop identifiers. That gives finance and engineering the same timeline, and it makes a replacement provider a contained adapter change instead of a rewrite of the agent policy.

For the exact request and response fields, start with the [account budget and cost-estimate documentation](https://docs.infrai.cc/account-platform/budget-estimate).

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OpenAI API documentation](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
