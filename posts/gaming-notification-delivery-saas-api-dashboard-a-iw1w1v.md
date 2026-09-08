# Gaming Notification Delivery: SaaS API Dashboard and Cron Failure Alerts

Short answer: for a gaming notification service, use separate custom metrics for delivery attempts, delivery failures, completed cron runs, and the age of the last run; poll those metrics with stateful rules, then keep enough identifiers in logs to reconstruct one failed delivery without putting player data into metric labels.

The best simple design is deliberately boring. A dashboard shows whether the system is drifting. An alert says that a defined condition changed. Logs explain which notification failed and where. Those are three jobs, not three views of one magic health number.

There is one hard limit: a custom query poll is appropriate for a small set of delay-tolerant alerts, but it isn't a paging system. If notification delivery has formal escalation, complex routing, or a strict response target, use a dedicated alerting path rather than asking one cron process to impersonate an on-call platform.

## Govern the incident evidence before it disappears

Start with the investigation you expect to perform. A player earns a reward, the game asks the notification service to send a message, and the provider-facing worker records an outcome. During an incident, the useful questions arrive in order: Did delivery traffic exist? What fraction failed? Did the scheduled retry job run? Which stage rejected one specific message?

Run that sequence as a tabletop before choosing a dashboard. At 03:17, an alert reports that delivery failures crossed the configured policy. The responder first checks attempts and learns that traffic did not disappear. The failure ratio rose while the successful retry timestamp remained fresh, so the cron worker is present and the problem is active rejection rather than scheduler silence. A bounded outcome code narrows the affected stage. The responder then takes one pseudonymous delivery key from a structured log result and follows acceptance, queueing, provider submission, and retry in timestamp order. No single chart proves the cause; together, the counters select the branch of the investigation, the completion signal rules out a missing worker, and the logs supply event-level evidence. If any one of those records is absent, the timeline stops at that boundary. That is the design test — not the color of the graph.

The alert is only a pointer.

## How should a SaaS API dashboard query custom metrics for failed cron jobs?

That sequence yields four measurements:

| Measurement | What it establishes | What it cannot establish |
| --- | --- | --- |
| `notification_delivery_attempts_total` | The denominator for delivery volume | Whether any attempt succeeded |
| `notification_delivery_failures_total` | Explicit failed outcomes | Whether a worker ran at all |
| `notification_retry_runs_total` | Completed retry-job runs | Whether the most recent run was on time |
| `notification_retry_last_success_seconds` | Recency of the last successful run | Why an individual delivery failed |

Use monotonically increasing counters for attempts, failures, and completed runs. The poll query calculates increases over the same window, then derives `failures / attempts` only when the denominator clears a volume floor. Without that floor, one failure from two attempts produces a dramatic percentage that may describe a quiet test shard rather than a broad delivery incident. The floor and threshold belong to service policy; there is no universal correct pair.

The recency gauge covers a different failure mode. If the retry scheduler never invokes its worker, a failure counter stays flat. Zero reported failures could mean clean execution or total silence. A successful-run timestamp lets the rule compare current time with the last observed completion, so absence becomes visible.

Tiny distinction. Huge payoff.

Here is the before/after mental model. Before: one red error-rate panel mixes player traffic, provider responses, and retry behavior, while a stopped cron job looks healthy because it emits nothing. After: the dashboard reads left to right as a diagram in words: traffic entered, delivery outcomes changed, retry runs completed, and the last successful run remained recent. Each panel answers one question, and each alert links to a next investigative move.

## Control identity and retention in reconstruction data

An error-rate alert is the starting gun. Incident reconstruction needs a stable, pseudonymous join key that follows the delivery through acceptance, queueing, provider submission, and retry. Put that key in structured logs, not in metric dimensions. A dashboard should take an operator from the alert window to the relevant log query; the log events then show the ordered stages for that delivery.

Don't put `player_id`, message text, email addresses, device tokens, or other user-specific values into metric labels. Besides multiplying time series, those values widen the places where sensitive data can travel. OWASP's logging guidance says logs should exclude or appropriately mask data such as access tokens, passwords, sensitive personal data, and connection strings. Metrics deserve the same design review even though their labels feel smaller than log records.

Consider a concrete, hypothetical window. The service records 2,400 attempts and 144 failures, so the observed error rate is 6%. The configured policy happens to require at least 1,000 attempts and a 5% failure rate before firing. Those numbers are examples, not recommended defaults. What matters is the reasoning: the denominator prevents low-volume noise, the ratio catches a broad shift, and the delivery key lets an engineer inspect representative failures without making every player a metric series.

A useful event contains a timestamp, the pseudonymous delivery key, the processing stage, a bounded outcome code, and the retry attempt number. Keep outcome codes enumerable, such as `provider_rejected` or `payload_invalid`, so both queries and dashboards remain understandable. Preserve the raw sensitive payload somewhere only if the system genuinely requires it and its access, retention, and deletion controls have been designed. A convenient debug field is not automatic permission to collect data.

This is where the architecture earns its keep. Suppose the failure-rate alert fires but retry-run recency remains normal. That points toward active processing with bad outcomes. Suppose delivery attempts fall to zero while upstream game events remain normal. That points toward the handoff before the worker. Suppose the retry completion timestamp becomes stale while failure counts remain flat. That is scheduler silence, not evidence of success. The same four panels produce three distinct hypotheses before anyone opens a log search.

I'm not sure what window will fit every game's traffic shape, because launches, regional peaks, and overnight troughs change the evidence available to a ratio. Resolve that uncertainty with historical distributions and replay tests: evaluate candidate rules against known normal windows and deliberately injected delivery failures, then document why the selected volume floor and duration are acceptable. Guessing a neat round threshold is easier. It is also harder to defend at 03:17.

## Keep the rule interface portable during a backend migration

Keep collection outside the rule. Any standards-based metrics backend or internal aggregation service can normalize one query window into the small input below. The evaluator has no network route, vendor response shape, or notification transport baked into it, which makes the decision logic straightforward to test. During a backend migration, run the old and new adapters against the same stored windows and compare their normalized `DeliveryWindow` values before switching alert delivery. The rule and its tests stay fixed; only collection changes. If the two adapters disagree, the team has a data-contract problem to resolve before it has an alert-policy problem.

```ts
type DeliveryWindow = {
  attempts: number;
  failures: number;
  retryRuns: number;
  secondsSinceRetrySuccess: number;
};

type AlertPolicy = {
  minimumAttempts: number;
  maximumFailureRate: number;
  minimumRetryRuns: number;
  maximumRetrySuccessAgeSeconds: number;
};

type AlertSignal = {
  kind: "delivery_failure_rate" | "missing_retry_run";
  summary: string;
};

function requireCount(name: string, value: number): void {
  if (!Number.isInteger(value) || value < 0) {
    throw new Error(`${name} must be a non-negative integer`);
  }
}

function evaluateDeliveryWindow(
  sample: DeliveryWindow,
  policy: AlertPolicy,
): AlertSignal[] {
  requireCount("attempts", sample.attempts);
  requireCount("failures", sample.failures);
  requireCount("retryRuns", sample.retryRuns);
  requireCount(
    "secondsSinceRetrySuccess",
    sample.secondsSinceRetrySuccess,
  );

  if (sample.failures > sample.attempts) {
    throw new Error("failures cannot exceed attempts");
  }

  const signals: AlertSignal[] = [];
  const failureRate = sample.attempts === 0
    ? 0
    : sample.failures / sample.attempts;

  if (
    sample.attempts >= policy.minimumAttempts &&
    failureRate >= policy.maximumFailureRate
  ) {
    signals.push({
      kind: "delivery_failure_rate",
      summary: `Delivery failure rate is ${(
        failureRate * 100
      ).toFixed(1)}%`,
    });
  }

  if (
    sample.retryRuns < policy.minimumRetryRuns ||
    sample.secondsSinceRetrySuccess >=
      policy.maximumRetrySuccessAgeSeconds
  ) {
    signals.push({
      kind: "missing_retry_run",
      summary: "Retry execution is outside its expected policy",
    });
  }

  return signals;
}

const signals = evaluateDeliveryWindow(
  {
    attempts: 2400,
    failures: 144,
    retryRuns: 1,
    secondsSinceRetrySuccess: 180,
  },
  {
    minimumAttempts: 1000,
    maximumFailureRate: 0.05,
    minimumRetryRuns: 1,
    maximumRetrySuccessAgeSeconds: 900,
  },
);

for (const signal of signals) {
  process.stdout.write(`${JSON.stringify(signal)}\n`);
}
```

The example returns one delivery failure-rate signal. Its retry run remains inside the illustrative policy. Change `retryRuns` to `0` and `secondsSinceRetrySuccess` to `901`, and the missing-run signal appears too. Crisp inputs make crisp tests.

The polling process still needs state. Notify when a condition changes from healthy to firing, not on every query. Record the active condition, the first observed time, the latest observed value, and the recovery transition. Otherwise a five-minute poll can repeat the same alert twelve times in an hour, training the team to mute the channel. Also monitor the poller's own successful completion from an independent path. A worker cannot reliably report its own disappearance.

Test the whole chain before deployment: fixture data enters the evaluator, a transition creates exactly one notification, a continuing failure creates no duplicate unless policy calls for reminders, and recovery creates one clear resolution. Then exercise the missing-data case. Query failure, empty traffic, and scheduler silence are different states; don't collapse all three into a zero.

## Test the polling chain as a reliability boundary

Sometimes, within a narrow boundary. A custom poll is suitable when there are only a few transparent rules, the query interval is an acceptable detection delay, one team owns the code, and notification delivery can remain simple. It keeps the policy readable and makes local tests cheap.

The catch is operational ownership. The team now owns scheduling, query validation, durable alert state, deduplication, notification delivery, recovery messages, access control, retention, and monitoring of the poller. The dashboard alone owns none of those responsibilities. It is a view.

Use a dedicated alerting system when missed delivery has a formal paging consequence, when schedules and escalation routes vary by team, when acknowledgment state matters, or when the notification path must be operated separately from the application. Likewise, a pure metrics design is not suitable when reconstruction requires request-level causality across many services; use an appropriate tracing and structured-logging design for that investigation. Stick with a code-owned poll only while its intentionally small boundary remains true.

The second objection is cost: doesn't another query every minute create needless load? Your mileage may vary. Measure query duration, scanned series, retention, and poll frequency in the actual backend, then choose the coarsest interval that still meets the detection target. Bounded labels and pre-aggregated counters usually make the question easier to reason about, but the answer must come from the system's measurements rather than a universal claim.

For the gaming notification service, the decision rule is concise: use counters to detect reported delivery failures, a last-success signal to detect missing cron work, and pseudonymous structured logs to reconstruct an individual path. Move beyond the custom poll when response coordination becomes the harder problem. That's the boundary.

## References

- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
