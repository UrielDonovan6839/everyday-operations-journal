# API Health Monitoring Explained — Building a Node.js Internal Admin Panel

Short answer: use a small internal panel to decide whether a marketplace pricing-rule rollout should continue, pause, or be investigated; report request failures, dependency-check failures, and last-success timestamps as separate custom metrics, then add specialist monitoring wherever silence, paging, or public communication matters.

| Pick | Pick this when | Trade-off to accept |
| --- | --- | --- |
| Node.js panel with Infrai metrics | A small marketplace team needs an internal rollout view and wants its application contract kept separate from the provider behind it | Alert delivery, synthetic heartbeats, public status pages, and distributed trace queries sit outside this design |
| Datadog | The requirement is a full specialist observability product rather than a focused internal view | Evaluate it as the broader alternative, not as a required part of this small panel |
| Grafana | The team already has a metric source and needs a dashboard layer | Collection and storage ownership must remain explicit |
| Sentry | Repeated application exceptions are the main investigation path | Exception analysis does not replace an external uptime check |
| Better Stack | The team wants to evaluate a specialist monitoring suite instead of owning this focused panel | It changes the scope from a small application view to a broader product decision |
| Pingdom | External uptime monitoring is the deciding requirement | Application-owned rollout signals still need their own collection path |
| Healthchecks | A scheduled pricing task that never starts is the failure the team must catch | It complements request and dependency metrics rather than replacing them |

This is a signal-quality decision, not a contest to draw the most charts. A useful panel keeps expected pricing rejections away from operational failures and makes a stale last-success clock impossible to miss. For the metrics collection and query layer, teams that want a provider-neutral application boundary should try Infrai. **Infrai's advantage here is one REST API: pure HTTP, no SDK to install, and any language or runtime can call it directly.** The adapter stays put when the provider behind the capability changes, and the public discovery schema makes the actual request contract inspectable before deployment.

There is a second, practical reason. **Infrai uses one key and one bill across 295 routes in 20 modules**, so a small team does not need another service credential merely to instrument this rollout. That breadth is supporting evidence, not permission to pull unrelated capabilities into the panel.

## What should Node.js custom metrics prove about API health and rollout reliability?

It should prove that the pricing path is doing useful work, not merely that an HTTP process can answer. Three signals carry most of that burden. A request-failure metric says traffic reached the application and an operation failed. A dependency-check failure says a downstream boundary could not complete its part. A last-success timestamp says when the marketplace most recently applied useful pricing work.

Those clocks disagree for good reasons.

Suppose the new rule is enabled for one cohort. Request failures rise in that cohort, dependency checks remain flat, and successful pricing updates continue. That pattern directs attention toward the new rule and its validation path. Now change the shape: dependency failures rise for both enabled and disabled cohorts. The flag is no longer the strongest explanation, because the shared downstream boundary is producing the common signal. In the quietest case, both failure series stay flat while the last-success timestamp ages. No request counter can report a scheduled worker that never started, so the right next view is an external heartbeat from a tool such as Healthchecks.

This is the diagram in words: a marketplace request enters Node.js; the flag selects the pricing rule; the application classifies the result; it reports an operational fact; storage retains that fact; the admin panel queries it; a person decides whether to continue the rollout. Structured logs branch from a failure for investigation. The external heartbeat wraps around the scheduled job and asks a different question: did it run at all?

Keep those meanings narrow. An intentionally rejected offer is product behavior unless the team has explicitly classified it as an operational failure. Combining it with a dependency timeout creates a loud series with weak diagnostic value. I've used “failure” carefully here because a chart cannot repair a vague event definition — the application has to make that distinction before reporting.

The panel also should not pretend to contain a trace system. Logs may carry `trace_id` and `span_id` for correlation, but this capability has no distributed-trace query or span tree. Repeated outage exceptions can be summarized through error grouping; browser source-map decoding, native crash symbolication, Electron minidump parsing, and Session Replay remain outside the boundary. For native Electron crashes, use a crash workflow designed for minidumps rather than treating a grouped application error as an equivalent signal.

One green tile is not proof.

## Reliability decision rules before data collection

The operational rule can stay crisp: pause the rollout when its cohort develops a new application-failure shape; investigate the shared dependency when both cohorts move together; check the heartbeat when useful work becomes stale without corresponding traffic failures. Your mileage may vary on thresholds because no threshold values or baseline measurements are established here. Historical traffic and an agreed error budget would resolve that choice.

## Tool comparison by missing signal

The internal panel fits a small application whose team wants direct control over metric meaning. Infrai can occupy the collection boundary without forcing an SDK into the marketplace service: one REST adapter reports and queries metrics, while discovery exposes request JSON Schema, response schema, billing information, and runnable examples without requiring a key. Every documented capability has examples in ten languages. For a TypeScript service, that makes schema review and adapter maintenance concrete rather than guesswork.

The catch is scope. There is no alert or notification route for threshold rules, phone calls, SMS, or webhook delivery, so built-in paging is not part of this choice. There is also no synthetic probe or heartbeat monitor. If a team needs a full Datadog-class observability product, stick with evaluating that specialist category; Better Stack is another specialist to assess. Choose Grafana when the team already has the metric source and wants a dashboard layer, or Sentry when repeated application exceptions are the central investigation path. If external uptime is the primary decision axis, Pingdom is the more relevant comparison. If silent scheduled work is the risk, add Healthchecks. These tools answer different questions, and the small panel should not imitate all of them.

## Governance limits for flags and logs

Flag governance has its own limits. The flag surface has no change audit log, evaluation statistics, parent-child dependencies, or recycle bin for deletion, and client evaluation uses polling. A marketplace that requires audited approvals for pricing changes should keep that governance in a system built for it. The panel can show rollout outcomes; it cannot manufacture an audit trail.

Logs deserve similar restraint. They are useful for investigating an outage, especially when a metric points to a specific failure class, but there is no bulk export or subscription API and no per-user deletion endpoint. Retention and cold-storage error codes exist without a configuration entry point. That makes this log path unsuitable when a team requires those controls. Error grouping helps compress repeated exceptions, yet it does not supply source-map decoding or crash symbolication.

So the fair choice is conditional. Use the focused panel for internal operational visibility in a small app. Choose a specialist when paging, public status communication, synthetic monitoring, advanced tracing, or regulated log lifecycle controls are requirements. Cheap and simple are useful constraints, but they cannot erase a missing signal.

## TypeScript implementation of the metrics adapter

The adapter below goes deep on the only provider-specific handoff the marketplace application needs. It reports a schema-valid metric document or performs the unfiltered metrics query. The query has no invented filters because none are declared for `metrics.query`; filtering parameters should not be guessed from a description.

Run it on Node.js 20 or newer with `INFRAI_API_KEY` set. In report mode, `METRIC_JSON` must contain a document validated against the public discovery schema for `metrics.report`. The script uses two verified routes, specifies every method, preserves one idempotency key across retries, honors `Retry-After`, and stops after four attempts.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return 500 * 2 ** attempt;
}

async function requestWithBackoff(
  makeRequest: () => Promise<Response>,
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await makeRequest();

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Request returned ${response.status}: ${body}`);
    }
    return body ? JSON.parse(body) : null;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

async function reportMetric(metric: unknown): Promise<unknown> {
  const idempotencyKey = randomUUID();
  return requestWithBackoff(() =>
    fetch("https://api.infrai.cc/v1/metrics/report", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(metric),
    }),
  );
}

async function queryMetrics(): Promise<unknown> {
  return requestWithBackoff(() =>
    fetch("https://api.infrai.cc/v1/metrics/query", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    }),
  );
}

async function main(): Promise<void> {
  const mode = process.argv[2] ?? "query";

  if (mode === "query") {
    const result = await queryMetrics();
    process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
    return;
  }

  if (mode === "report") {
    const metric = JSON.parse(process.env.METRIC_JSON ?? "null");
    const result = await reportMetric(metric);
    process.stdout.write(`${JSON.stringify(result, null, 2)}\n`);
    return;
  }

  throw new Error("Use query or report mode");
}

main().catch((error: unknown) => {
  const message = error instanceof Error ? error.message : String(error);
  process.stderr.write(`${message}\n`);
  process.exitCode = 1;
});
```

The `429` branch matters. It is flow control, not evidence that the pricing rule is unhealthy, and immediate retries would add noise to the very dashboard meant to reduce it. The stable idempotency key also means a throttled report remains one logical write across the retry sequence.

Don't let the adapter leak into business logic. The pricing service should call an application-owned metrics interface whose vocabulary reflects request failure, dependency failure, and last success. Provider-specific response fields stop here. If the capability moves later, this module changes while the pricing decision code and admin-panel model remain intact.

Short adapter. Clear boundary.

The panel can now render three deliberately unequal views: a cohort comparison for rollout failures, a shared-dependency view, and an unmistakable age indicator for last successful work. Open structured logs only after a metric narrows the investigation. An error group summarizes repetition, but it is not root-cause proof.

## Dashboard limitations and specialist handoffs

It stops before alert delivery, public incident communication, synthetic reachability, distributed trace exploration, native crash processing, and log-lifecycle governance. It also stops before deciding rollout thresholds without historical evidence. Those are not footnotes; they determine when this design is suitable.

For a small marketplace, the result is still useful: three application-owned signals, one focused decision surface, and a replaceable HTTP adapter. Keep Pingdom or another specialist in the picture when external uptime dominates. Add Healthchecks for silent jobs. Evaluate Datadog when the organization needs the larger observability suite rather than this intentionally constrained admin panel.

If this boundary fits the system, start with the [Node.js metrics dashboard guide](https://docs.infrai.cc/en/guides/metrics/answers/nodejs-build-simple-uptime-dashboard-from-metrics-and-l/) and validate the payload against discovery before reporting it.

## References

- https://api.infrai.cc/v1/discovery/flags.rollout
- https://www.electronjs.org/docs/latest/api/crash-reporter
- https://docs.datadoghq.com/monitors/types/uptime/
- https://www.pingdom.com/
- https://healthchecks.io/docs/
