# Node.js Webhook Retries: Rate Limits and Recovery for Property Reservations

Short answer: retry a failed reservation webhook from a durable queue, acknowledge the queue message only after the retry decision is persisted, and enforce the destination's rate limit before every attempt. Keep the reservation's hold-window expiry separate from delivery retries. One clock decides when the reservation expires; another decides when delivery is attempted again.

For a property-management system, that separation is the operational recovery rule. A delayed task can expire a reservation after 15 minutes, for example, while its webhook may need several hours of controlled retry. If the process restarts, the database and queue should still explain what happened: `pending`, `sent`, `retrying`, or `dead_lettered`.

## How should a queue retry a failed webhook delivery?

Start with the failure, not with a fashionable queue feature. This small decision table keeps recovery behavior visible.

| Failure signal | Retry decision | Operational reason |
| --- | --- | --- |
| Timeout or connection failure | Retry with exponential backoff and jitter | The receiver may be unavailable, and synchronized retries create a second outage. |
| HTTP 429 | Honor `Retry-After` when present, then apply a capped delay | The receiver has explicitly asked for slower traffic. |
| HTTP 400 or a validated schema error | Do not retry automatically | Repeating a permanent input error only creates noise. |
| HTTP 500 or 503 | Retry within a bounded window | The receiver may recover, but the sender needs a terminal state. |

Pick a simple delayed queue when the retry schedule is short and a small worker fleet can own the state. Pick a database-backed schedule when operators need SQL queries, manual replay, or a clear audit trail across deployments. Pick a separate rate-limiter service when many workers share one destination quota and local counters cannot coordinate safely. Each option can work. The choice is about recovery ownership.

One rule is non-negotiable.

Keep it boring.

## How should a Node.js worker make the retry decision?

Use one durable record per delivery. It should contain the reservation identifier, an idempotency key, attempt count, next-attempt time, last status, and the final error classification. The worker claims due records, checks the limiter, sends the webhook, and commits the outcome.

The important ordering is easy to get wrong. Marking a queue message complete before recording the outcome loses the delivery if the process dies in between. Recording a retry and then refusing the queue acknowledgement can create a duplicate. Design for at-least-once processing and make the receiver safe to call more than once with the same idempotency key.

Here is the core decision loop. The queue adapter is deliberately generic; `ack` and `retryLater` are the only queue-specific edges.

```ts
type DeliveryResult =
  | { kind: "sent"; status: number }
  | { kind: "retry"; status?: number; retryAfterMs?: number; reason: string }
  | { kind: "dead"; status?: number; reason: string };

const MAX_ATTEMPTS = 8;
const MAX_DELAY_MS = 6 * 60 * 60 * 1000;

function backoffMs(attempt: number): number {
  const base = Math.min(MAX_DELAY_MS, 1_000 * 2 ** attempt);
  const jitter = Math.floor(Math.random() * 1_000);
  return Math.min(MAX_DELAY_MS, base + jitter);
}

async function handleDelivery(message: { deliveryId: string; attempt: number }) {
  const delivery = await store.claim(message.deliveryId);
  if (!delivery || delivery.state === "sent" || delivery.state === "dead") {
    await queue.ack(message);
    return;
  }

  await limiter.waitForSlot(delivery.destination);
  const result = await sendWebhook({
    url: delivery.url,
    body: delivery.body,
    headers: { "Idempotency-Key": delivery.idempotencyKey }
  });

  if (result.kind === "sent") {
    await store.markSent(delivery.id, result.status);
    await queue.ack(message);
    return;
  }

  if (result.kind === "dead" || delivery.attempt + 1 >= MAX_ATTEMPTS) {
    await store.markDead(delivery.id, result.reason, result.status);
    await queue.ack(message);
    return;
  }

  const delay = result.retryAfterMs ?? backoffMs(delivery.attempt);
  await store.scheduleRetry(delivery.id, Date.now() + delay, result.reason);
  await queue.ack(message);
}
```

The example caps both attempts and delay. Those are policy values, not laws. The useful part is the transaction boundary: persist `sent`, `dead`, or the next attempt before acknowledging the queue message. In a real implementation, `claim` also needs a lease or compare-and-set guard so two workers do not send the same delivery concurrently.

## What does rate limiting change in production?

A retry policy without a limiter is an outage amplifier. Suppose 200 reservations expire together after a busy check-in window. If every worker retries at the same two-second interval, the receiver sees a burst even when the average traffic is acceptable. Exponential backoff reduces repeated pressure; jitter prevents workers from sharing the same timetable.

Use a token bucket or another explicit quota model, and decide where its state lives. A per-process limiter is easy to deploy but does not represent a fleet-wide limit. A shared limiter gives a stronger guarantee but adds a dependency that must be monitored. For a single destination, I would rather have a conservative shared budget and visible queue lag than a clever local estimate that silently violates the contract. The trade-off is deliberate: coordination improves correctness across workers, while a local counter reduces moving parts and can be adequate for one process.

Treat `429` as a scheduling input. Parse `Retry-After` carefully, bound an unreasonable value, and retain the reason in the delivery record. Never turn every non-2xx response into a retry: authentication failures, malformed payloads, and rejected reservation identifiers need an operator or application fix.

## What should operators recover first?

The dashboard needs three views: due deliveries, retry age, and terminal failures. A count of successful HTTP responses is not enough. Alert on the age of the oldest due delivery and on the rate of dead-letter transitions. Those signals answer the operational question: can the system catch up before the next reservation batch arrives?

Keep the reservation expiry workflow independent. If the webhook receiver is down, the reservation should still become expired according to the hold window. The delivery record can then report that state change later. A webhook is an integration consequence of the business event, not the clock that defines the event.

Test the awkward boundaries: a worker crash after the HTTP response but before `markSent`, a restart with an expired lease, a 429 with and without `Retry-After`, and two workers claiming one delivery. The expected result is at-least-once execution with idempotent effects, bounded retries, and an audit trail that makes manual replay safe.

The operational limitations are real. No retry loop can repair a permanently invalid payload. No local limiter can know a quota held by another service. And a queue cannot tell you whether a receiver applied a request before the network connection failed. For that last ambiguity, the idempotency key, receiver-side deduplication, and operator-visible status are the practical contract.

## References

- https://www.rabbitmq.com/docs/confirms
- https://nodejs.org/api/timers.html
- https://www.rfc-editor.org/rfc/rfc9110.html#name-retry-after
- https://opentelemetry.io/docs/specs/semconv/
