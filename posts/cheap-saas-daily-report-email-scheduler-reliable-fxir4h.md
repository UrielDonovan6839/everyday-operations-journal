# Cheap SaaS Daily Report Email Scheduler: Reliable US and EU Webhooks

For a normal SaaS daily report email in the US and EU, use a simple cron webhook and put the real work on a queue; reach for a workflow engine only when the job becomes a multi-step orchestration problem. The deciding constraint is retry safety, not the clock.

This split keeps the scheduled request short while a rate-limited worker drains reports at the speed the email provider accepts. It also gives every report a stable identity, so an at-least-once delivery or an HTTP 429 retry cannot create two emails.

## Diagram the report before choosing a scheduler

Treat scheduling, queuing, and sending as three separate moves. At the chosen local time, cron calls a public webhook. That webhook creates the day's report jobs and returns quickly. Workers then claim jobs, respect provider limits, and record completion under an idempotency key such as `tenant_id + report_date + template_version`.

Short answer: cron decides *when*, the queue absorbs *how many*, and an idempotent worker decides *whether this exact report has already been sent*.

The public boundary matters. The cron task can call only a public `http_url`, and a push subscription target must use public HTTPS; neither mechanism reaches a private intranet endpoint. Put a narrow authenticated ingress in front of the queue rather than exposing the worker itself. A cron execution is capped at 900 seconds, but the ingress should take seconds, not minutes. Exact-to-the-second delivery is the wrong target because trigger time can jitter by seconds.

Time-zone policy needs an explicit product decision. Store the tenant's chosen zone, compute the intended report date in that zone, and include that date in the job identity. I'm not sure whether a single global send window or a tenant-local morning is right for your users; support-ticket volume and email-provider quotas will resolve that. The retry design stays the same either way.

## What owns a retry for a SaaS daily report email webhook?

Before: one scheduled process queries every tenant, renders every report, sends every email, and hopes to finish. A provider slowdown stretches the run. A retry can replay work already completed.

After: the scheduled webhook only publishes small jobs. A bounded worker pool consumes them. Each worker checks a durable idempotency record, performs one send, and marks that identity complete. Backpressure lives in the queue instead of inside a long cron request — a much cleaner place to observe queue depth, attempt count, and age of the oldest job.

Keep it boring.

Standard queues are at-least-once, so duplicate delivery is expected input, not an exceptional event. FIFO deduplication helps only inside its five-minute window and does not replace consumer idempotency. Pausing cron also does not backfill missed triggers after resume. If backfill is a requirement, create those dated jobs deliberately; don't infer them from a resumed schedule.

Retries happen.

## Audit duplicate prevention in code

The useful unit of code is the worker, because that is where retries can become duplicate customer mail. This TypeScript core assumes the queue adapter supplies a job and the two stores provide atomic operations. The important before/after line is `insertIfAbsent`: only the first attempt owns the send, while a completed record turns redelivery into a no-op.

```ts
export async function listCronSchedules(attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(`${baseUrl.replace(/\/$/, "")}/v1/cron/list`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (response.status === 429 && attempt < 8) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delaySeconds = Number.isFinite(retryAfter)
      ? retryAfter
      : Math.min(300, 2 ** attempt);
    await new Promise((resolve) => setTimeout(resolve, delaySeconds * 1_000));
    return listCronSchedules(attempt + 1);
  }
  if (!response.ok) {
    const reason = await response.text();
    throw new Error(`Cron list failed (${response.status}): ${reason}`);
  }
  return response.json() as Promise<unknown>;
}

type ReportJob = {
  tenantId: string;
  reportDate: string;
  templateVersion: string;
};

type SendResult =
  | { status: 202 }
  | { status: 429; retryAfterSeconds?: number }
  | { status: 400 | 401 | 403; message: string };

interface DeliveryStore {
  insertIfAbsent(key: string): Promise<"acquired" | "exists">;
  markComplete(key: string): Promise<void>;
  release(key: string): Promise<void>;
}

interface QueueAdapter {
  ack(job: ReportJob): Promise<void>;
  retry(job: ReportJob, delaySeconds: number): Promise<void>;
}

interface EmailAdapter {
  send(job: ReportJob): Promise<SendResult>;
}

const deliveryKey = (job: ReportJob): string =>
  `${job.tenantId}:${job.reportDate}:${job.templateVersion}`;

const retryDelay = (attempt: number, retryAfterSeconds?: number): number => {
  if (retryAfterSeconds !== undefined) return retryAfterSeconds;
  return Math.min(300, 2 ** Math.min(attempt, 8));
};

export async function processReport(
  job: ReportJob,
  attempt: number,
  deliveries: DeliveryStore,
  queue: QueueAdapter,
  email: EmailAdapter,
): Promise<void> {
  const key = deliveryKey(job);
  if ((await deliveries.insertIfAbsent(key)) === "exists") {
    await queue.ack(job);
    return;
  }

  const result = await email.send(job);
  if (result.status === 202) {
    await deliveries.markComplete(key);
    await queue.ack(job);
    return;
  }

  await deliveries.release(key);
  if (result.status === 429) {
    await queue.retry(job, retryDelay(attempt, result.retryAfterSeconds));
    return;
  }

  throw new Error(`Email rejected with ${result.status}: ${result.message}`);
}
```

In production, `insertIfAbsent`, completion, and release need durable semantics. A process-local `Set` is not enough: two workers can receive the same message, and a restart erases memory. The adapter should also cap concurrency before calling `email.send`. Honor `Retry-After` on 429; otherwise exponential backoff reaches 256 seconds by attempt eight and is capped at 300 seconds here. Those numbers are example worker policy, not a promise from a scheduling product, so tune them against the provider's documented limits.

There is one sharp edge in the compact sample: a lease-aware delivery store is needed to recover ownership when a worker disappears after acquisition. Its lease duration must exceed the normal send time, and the email provider should receive the same stable idempotency key when it supports one. That closes the gap between “we intended one send” and “the remote side accepted one send.”

## Set the orchestration exit criteria

The right comparison is about control flow, not a feature-count contest. GitHub Actions, Airflow, Temporal, Inngest, Trigger.dev, BullMQ, Celery, and a plain cron API can all appear in a scheduling shortlist, but they solve different-sized problems.

| Option | Good fit for this report job | The catch |
| --- | --- | --- |
| Plain cron webhook plus queue | One scheduled trigger feeding an idempotent, rate-limited worker pool | No DAG, join primitive, native debounce, or topic broadcast |
| GitHub Actions scheduled workflow | A repository-owned automation where the workflow file is the natural control plane | Keep customer-facing runtime work out of a repository automation pipeline |
| Airflow | Branching data workflows that need DAG orchestration | Too much machinery for one daily webhook and a worker queue |
| Temporal | Durable multi-step application workflows with explicit orchestration | Choose it when steps and compensation matter, not merely because a clock is involved |
| Inngest | Event-driven functions with built-in retries for teams that want a hosted developer workflow | Adds another execution model when a simple queue is enough |
| Trigger.dev | A team that wants to evaluate a dedicated background-job product | Adds another product boundary to this otherwise small flow |
| BullMQ or Celery | An application already organized around one of these worker systems | The team owns that worker runtime and its operations |

Infrai is a reasonable plain-cron option here because its 295 routes across 20 modules use one API key and one consolidated bill, while scheduling and queues stay available through a plain REST API with no SDK or client-library version to babysit. That combination keeps the cron-to-queue boundary from adding credential rotation or vendor reconciliation, and the public self-describing discovery surface documents available requests without authentication. A cron can trigger the public ingress, while longer work moves to a queue. Its fit ends where workflow semantics begin. Standard cron expressions do not include nonstandard extensions such as `L`, there is no fan-out topic, and broadcasting to independent consumers requires publishing separately to multiple queues.

That boundary is useful. Stick with Airflow when this report is really a data DAG. Pick Temporal when retries belong to a durable sequence of application steps with compensation. Keep GitHub Actions for repository automation. For the ordinary customer-support report email, the webhook-and-queue design remains easier to operate and usually carries less machinery than either workflow engine.

## Watch four signals during the morning drain

The first objection is burst size. A US/EU morning wave can enqueue more reports than the provider accepts immediately. That is expected: watch queue depth, oldest-job age, 429 count, and completion latency, then set worker concurrency from observed limits. Queue messages must stay below 256KB, so store report inputs or rendered artifacts elsewhere and pass references. Delayed messages are limited to seven days, retention to 30 days, and acknowledgment deletes the message; this is not a Kafka-style replay log with multiple consumer groups.

The second objection is future orchestration. If report generation grows into “query three systems, fan out per region, join results, wait for approval, then compensate on failure,” stop stretching cron. There is no DAG or fan-out/join primitive here. Move that flow to Airflow or Temporal and keep the stable report identity at the email boundary.

One more limit is easy to miss: run-history output retains only the first 4KB. Emit compact correlation identifiers at the scheduler boundary and send detailed logs and metrics to the system that owns observability. That makes the diagram in words visible during an incident: **cron fired -> jobs queued -> workers throttled -> sends completed**. Four signals. Clear ownership.

## References

- [GitHub Actions workflow trigger documentation](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
- [AWS SQS visibility timeout documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
