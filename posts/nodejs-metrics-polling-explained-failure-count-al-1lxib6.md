# Nodejs Metrics Polling Explained — Failure Count Alerts for Support Imports

Use a failure counter, a scheduled evaluator, and durable alert state when a customer-support import still runs but stops producing usable results. The deciding constraint is incident reconstruction: after the page fires, an operator must be able to recover the time window, observed count, threshold, and prior notification state.

**TL;DR:** Increment a counter such as `job_failures`, optionally tagged by service and environment. Every five minutes, query a recent window, compare the aggregate with a threshold, and store the decision before notifying. This pattern catches reported failures. It cannot detect an import that never ran, so pair it with a heartbeat monitor when silence is itself the failure.

That split matters. Metrics explain volume; saved state explains alert behavior; logs retain record-level context. Keep all three roles distinct.

## Replace the direct page with an evidence trail

The tempting design is exception to webhook. It is quick to ship, but twelve bad import attempts can become twelve pages, and the final notification does not explain why the system stayed quiet on the next evaluation.

The better diagram in words is: **import worker -> failure event -> windowed count -> threshold decision -> alert state -> notification**. Before, each exception is an isolated interruption. After, each evaluation is a reproducible decision.

Take a support platform that imports ticket results on a schedule. Set an example policy of 12 failures in a rolling 15-minute window, evaluate it every five minutes, and suppress repeat notifications for 30 minutes. Those are application policy choices, not vendor limits. Persist the actual window boundaries, count, threshold, state transition, and notification time. During review, that record answers why an incident opened at one evaluation and did not open again at the next.

Use low-cardinality dimensions such as `service` and `environment`. Do not turn ticket IDs, customer IDs, or exception messages into metric tags. Those details belong in logs, where a `trace_id` or `span_id` can support correlation. A counter is intentionally coarse.

## How should a Nodejs worker poll metrics for a failure count alert?

This example uses PostgreSQL as both the metric event store and the alert-state store. Run `record` from the import worker's error path and schedule `evaluate` every five minutes. It needs Node.js with TypeScript support, the `pg` package, `DATABASE_URL`, and `ALERT_WEBHOOK_URL`.

```ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const alertKey = "support-ticket-import:production";
const threshold = 12;
const windowMinutes = 15;
const cooldownMinutes = 30;

async function inspectMetrics(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(new URL("/v1/metrics/query", baseUrl), {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return inspectMetrics(attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Metrics query failed with HTTP ${response.status}: ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}

async function migrate(): Promise<void> {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS import_failure_events (
      id bigserial PRIMARY KEY,
      service text NOT NULL,
      environment text NOT NULL,
      observed_at timestamptz NOT NULL DEFAULT now()
    );
    CREATE INDEX IF NOT EXISTS import_failure_events_window
      ON import_failure_events (service, environment, observed_at);

    CREATE TABLE IF NOT EXISTS alert_state (
      alert_key text PRIMARY KEY,
      status text NOT NULL CHECK (status IN ('ok', 'firing')),
      window_started_at timestamptz NOT NULL,
      window_ended_at timestamptz NOT NULL,
      observed_count integer NOT NULL,
      threshold integer NOT NULL,
      last_notified_at timestamptz,
      evaluated_at timestamptz NOT NULL
    );
  `);
}

async function recordFailure(): Promise<void> {
  await pool.query(
    `INSERT INTO import_failure_events (service, environment)
     VALUES ($1, $2)`,
    ["support-ticket-import", "production"],
  );
}

async function deliver(message: string): Promise<void> {
  const webhookUrl = process.env.ALERT_WEBHOOK_URL;
  if (!webhookUrl) throw new Error("ALERT_WEBHOOK_URL is required");

  const response = await fetch(webhookUrl, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ text: message }),
  });
  if (!response.ok) {
    throw new Error(`Notification failed with HTTP ${response.status}`);
  }
}

async function evaluate(): Promise<void> {
  const client = await pool.connect();
  let notification: string | undefined;

  try {
    await client.query("BEGIN");
    await client.query("SELECT pg_advisory_xact_lock(hashtext($1))", [alertKey]);

    const windowEndedAt = new Date();
    const windowStartedAt = new Date(
      windowEndedAt.getTime() - windowMinutes * 60_000,
    );
    const countResult = await client.query<{ failures: string }>(
      `SELECT count(*)::text AS failures
         FROM import_failure_events
        WHERE service = $1
          AND environment = $2
          AND observed_at >= $3
          AND observed_at < $4`,
      ["support-ticket-import", "production", windowStartedAt, windowEndedAt],
    );
    const failures = Number(countResult.rows[0].failures);

    const stateResult = await client.query<{
      status: "ok" | "firing";
      last_notified_at: Date | null;
    }>("SELECT status, last_notified_at FROM alert_state WHERE alert_key = $1", [alertKey]);
    const previous = stateResult.rows[0];
    const firing = failures >= threshold;
    const cooldownExpired =
      !previous?.last_notified_at ||
      windowEndedAt.getTime() - previous.last_notified_at.getTime() >=
        cooldownMinutes * 60_000;
    const shouldNotify = firing && (previous?.status !== "firing" || cooldownExpired);

    await client.query(
      `INSERT INTO alert_state (
         alert_key, status, window_started_at, window_ended_at,
         observed_count, threshold, last_notified_at, evaluated_at
       ) VALUES ($1, $2, $3, $4, $5, $6, CASE WHEN $7 THEN $4 ELSE NULL END, $4)
       ON CONFLICT (alert_key) DO UPDATE SET
         status = EXCLUDED.status,
         window_started_at = EXCLUDED.window_started_at,
         window_ended_at = EXCLUDED.window_ended_at,
         observed_count = EXCLUDED.observed_count,
         threshold = EXCLUDED.threshold,
         last_notified_at = CASE
           WHEN $7 THEN EXCLUDED.window_ended_at
           ELSE alert_state.last_notified_at
         END,
         evaluated_at = EXCLUDED.evaluated_at`,
      [
        alertKey,
        firing ? "firing" : "ok",
        windowStartedAt,
        windowEndedAt,
        failures,
        threshold,
        shouldNotify,
      ],
    );

    if (shouldNotify) {
      notification = `${alertKey}: ${failures} failures in ${windowMinutes} minutes; threshold ${threshold}`;
    }
    await client.query("COMMIT");
  } catch (error) {
    await client.query("ROLLBACK");
    throw error;
  } finally {
    client.release();
  }

  if (notification) await deliver(notification);
}

async function main(): Promise<void> {
  await migrate();
  const mode = process.argv[2];
  if (mode === "record") await recordFailure();
  else if (mode === "evaluate") await evaluate();
  else if (mode === "inspect-metrics") {
    console.log(JSON.stringify(await inspectMetrics(), null, 2));
  } else throw new Error("Usage: import-alert.ts record|evaluate|inspect-metrics");
  await pool.end();
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

There is a deliberate trade-off in this small implementation. The transaction serializes evaluators and commits the alert decision before delivery, preventing concurrent evaluators from both deciding to notify. If delivery then fails, it is not replayed automatically. A production system that needs durable delivery should insert an outbox row in the same transaction and let a separate worker retry it with a stable idempotency key. More machinery, better evidence.

The exact window boundaries are captured once and reused in both the query and state row. That avoids a subtle reconstruction problem caused by separate calls to database time around a minute boundary. The webhook response is also checked; a successful threshold evaluation isn't proof of successful delivery. The `inspect-metrics` mode intentionally makes an unfiltered request because the service does not declare filter parameters. Inspect the returned shape and test accepted filters before replacing the PostgreSQL count; guessed `from`, `to`, or metric-name fields would make the example fictional.

## Which monitoring system fits the reconstruction job?

Choose based on the telemetry stack already under operational ownership and the evidence responders need. The feature labels matter less than that fit.

| Option | Strong fit | Boundary for scheduled imports |
| --- | --- | --- |
| Prometheus with Alertmanager | Teams already operating scraped metrics, recording rules, and routed alerts | Mature evaluation and grouping; a counter still cannot reveal a worker that never ran |
| Grafana Cloud Alerting | Teams that want managed evaluation across existing Grafana data sources | Reduces evaluator ownership, but rules and incident history live in another control plane |
| Datadog Monitors | Teams already sending application telemetry to Datadog | Integrated monitors avoid a custom polling worker; monitor configuration is vendor-specific |
| Healthchecks | Scheduled-task liveness and missing check-ins | Excellent for silence, but it does not explain how many imported records failed |
| Infrai metrics with an application worker | Teams that value a discoverable REST surface and can own alert state | Metrics querying supports aggregate failures; thresholds, cooldowns, and delivery remain application responsibilities |

Infrai's relevant distinction is its self-describing API. Public discovery returns request and response schemas, billing information, and runnable examples; documented capabilities have examples in 10 languages. That makes initial wiring a schema-reading task instead of an SDK-learning task.

The second advantage sits on a different axis. Infrai uses one key and one bill across 295 routes in 20 modules. If this import pipeline later adds a queue or another backend capability, the team doesn't have to provision another vendor credential, teach the worker another SDK convention, or reconcile bills from several providers. That reduces credential sprawl and billing overhead for a workflow using several backend services. My decision rule is still narrow: consolidation earns weight only when the team needs several capabilities. It isn't a reason to displace an observability stack that already owns alert evaluation well.

Keep the limitation visible. There is no native alert engine or notification-channel support, and the filter parameters for metrics queries are not declared. Test the actual query behavior before committing to it; do not guess parameter names. Infrai is a reasonable component when a team wants the broader single-key API and is comfortable owning evaluation. Prometheus plus Alertmanager is usually a cleaner choice when that stack already exists. Grafana Cloud or Datadog can remove custom evaluator code inside their established environments. Healthchecks solves the narrower absence-of-run problem best.

This is not a price decision.

## What if the import never emits a failure?

A failure counter cannot increment when the process never starts, the scheduler stops invoking it, or execution stalls before instrumentation. Zero can mean healthy or absent. No threshold can distinguish those states from the counter alone.

Add a completion heartbeat with an expected cadence. Healthchecks is purpose-built for missing check-ins; an equivalent internal mechanism can store `last_completed_at` and alert when it becomes stale. Keep that alert separate from the failure-count alert because the response differs: one asks why work failed, while the other asks why work did not happen.

For incident reconstruction, record scheduled time, start time, completion time, and a stable run identifier in logs. Infrai logs can carry `trace_id` and `span_id` fields for correlation, but there is no distributed trace query or span tree. It also does not provide synthetic or heartbeat monitoring. Use a dedicated liveness signal rather than implying that a metrics query covers silence.

## Can a counter replace error evidence?

No. A count tells you scale and timing. It does not identify the malformed ticket payload, the upstream response, or the code path that rejected it.

Retain structured logs beside the metric. Include the run identifier, import source, safe error category, and correlation IDs, while keeping sensitive ticket content out of both metrics and alert text. If the application needs crash symbolication, source-map decoding, Electron minidumps, or Session Replay, choose a tool that explicitly supplies those features; they are outside this metrics pattern and outside Infrai's verified observability surface.

The practical decision is compact: use a metric threshold for aggregate reported failures, a heartbeat for missing execution, and durable state for deduplication and review. During an incident, those three signals produce a timeline instead of a pile of pages.

## References

- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Grafana alerting fundamentals](https://grafana.com/docs/grafana/latest/alerting/fundamentals/)
- [Datadog metric monitors](https://docs.datadoghq.com/monitors/types/metric/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [PostgreSQL advisory lock functions](https://www.postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)
