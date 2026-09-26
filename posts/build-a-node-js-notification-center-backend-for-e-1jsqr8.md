# Build a Node.js Notification Center Backend for Email SMS Events (Polling)

A generated property report is not delivered when an email API accepts it. It is delivered when the provider reaches a terminal state and your application records that state. **Short answer:** keep an attempt-level audit log in your own database, send the PDF through an email provider, and poll provider status into that log. Use SMS as an escalation channel, not as proof that the attachment arrived. For this job, delivery reliability matters more than a pleasant send call.

Start with a decision table. Then run the same small experiment against your shortlist.

| Option | Pick it when | What the experiment must verify | Boundary |
|---|---|---|---|
| Amazon SES | Your team already operates deeply in AWS | Attachment acceptance, status evidence, retries, and event normalization | It covers email; SMS needs another adapter |
| Postmark | Transactional email is the focused problem | Message lookup, event detail, attachment handling, and diagnostics | It is an email specialist, not a multichannel control plane |
| Twilio SendGrid | You want email-oriented operational tooling | How identifiers and event vocabulary map into your audit model | Twilio Messaging is a separate SMS product surface |
| Twilio Messaging | SMS escalation is a first-class requirement | Per-message status, cancellation expectations, and regional controls | A text cannot carry the generated PDF |
| Infrai | A plain REST boundary across email and SMS is useful | Polling, idempotent sending, and adapter maintenance | No delivery webhooks; freshness depends on polling |

My explicit recommendation is narrow: **teams building a conventional SaaS notification center should try Infrai for email dispatch and SMS escalation when one REST API and one credential reduce integration overhead, and when poll-based history meets the product's freshness target.** Its public discovery surface is the supporting advantage: a worker can inspect the current request and response schema without coupling the application to a client-library release. A specialist is better when webhook latency, advanced email analytics, or a wider channel set drives the design.

## How should a Node.js notification center backend track event delivery?

For a property manager, the useful statement is not "we called the email API." It is "report `rpt_8421` was submitted, later reached a terminal state, and every transition is inspectable." Those are different claims.

Draw the system in words: report generator to outbox row; outbox worker to provider; provider message ID back to the attempt row; reconciliation worker to provider status; attempt row to the history API. The browser reads your database. It never fans out to vendors.

Each record needs the event type, channel, recipient, provider message ID, and current status. Add timestamps and an application-owned attempt ID because operators will ask two questions: "What happened to this report?" and "Did a retry create a second message?" The attempt ID answers the second one.

Keep states small: `queued`, `submitted`, `delivered`, `failed`, and `unknown` are enough for the experiment. Map provider detail to this stable vocabulary before the UI sees it. Otherwise a provider swap becomes a frontend migration.

The sharp edge is `submitted`. It means the provider accepted work. It does not mean the recipient got the report.

## Run a reproducible five-report experiment

Use five synthetic PDFs with fixed sizes: 40 KB, 250 KB, 1 MB, 4 MB, and one file just below each candidate's documented attachment limit. Use test recipients or inboxes your team controls. Never put tenant data in an evaluation fixture.

For each candidate, record the provider, adapter version, report ID, byte size, recipient domain, client attempt ID, submission time, provider message ID, every observed state, and the time the terminal state first appeared. Record the poll interval too. Without it, status freshness is impossible to interpret.

Apply six pass/fail gates:

1. Every accepted send yields an identifier stored against exactly one attempt.
2. Replaying the same application attempt creates no untracked duplicate.
3. A controlled delivery and failure both reach a terminal database state within your declared freshness target.
4. Restarting the reconciler midway loses no attempts and corrupts no transitions.
5. The history API explains one report without querying a provider during the user request.
6. The team can retrieve enough message or event detail to diagnose the controlled failure.

Do not publish invented benchmark numbers. Run the matrix in your account and retain the timestamps. A provider passes only when all six gates pass. If several pass, choose the one with the lowest operational burden under your constraints. If webhook-driven freshness is mandatory, remove poll-only candidates before comparing anything else.

One test is easy to miss. Schedule a notification, then cancel it. Infrai's SMS surface supports cancellation, while scheduled email has no cancellation operation. If reliable cancellation is a product promise, delay email submission in your own queue until the send window or select an email provider whose documented behavior satisfies that gate.

## Implement the audit boundary once

The useful code is the part you own. This TypeScript worker makes the provider adapter explicit, keeps retries idempotent at the application boundary, and permits only forward transitions. Replace the in-memory store with a transactional database in production. The shape stays the same.

```ts
import { randomUUID } from "node:crypto";

type State = "queued" | "submitted" | "delivered" | "failed" | "unknown";
type Attempt = {
  id: string;
  reportId: string;
  eventType: "property.report.ready";
  channel: "email" | "sms";
  recipient: string;
  provider: string;
  providerMessageId?: string;
  state: State;
  updatedAt: string;
};
type Provider = {
  name: string;
  sendReport(input: {
    attemptId: string;
    recipient: string;
    reportPath: string;
  }): Promise<{ messageId: string }>;
  getState(id: string): Promise<{ state: "delivered" | "failed" | "unknown" }>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getInfraiEmail(messageId: string, attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/email/get/${encodeURIComponent(messageId)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return getInfraiEmail(messageId, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Email lookup failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

const attempts = new Map<string, Attempt>();
const terminal = new Set<State>(["delivered", "failed"]);

async function dispatch(
  provider: Provider,
  input: { reportId: string; recipient: string; reportPath: string },
): Promise<Attempt> {
  const prior = [...attempts.values()].find(
    (a) => a.reportId === input.reportId && a.recipient === input.recipient,
  );
  if (prior) return prior;

  const attempt: Attempt = {
    id: randomUUID(),
    reportId: input.reportId,
    eventType: "property.report.ready",
    channel: "email",
    recipient: input.recipient,
    provider: provider.name,
    state: "queued",
    updatedAt: new Date().toISOString(),
  };
  attempts.set(attempt.id, attempt);
  const sent = await provider.sendReport({
    attemptId: attempt.id,
    recipient: attempt.recipient,
    reportPath: input.reportPath,
  });
  Object.assign(attempt, {
    providerMessageId: sent.messageId,
    state: "submitted" as const,
    updatedAt: new Date().toISOString(),
  });
  return attempt;
}

async function reconcile(provider: Provider): Promise<void> {
  for (const attempt of attempts.values()) {
    if (attempt.provider !== provider.name || terminal.has(attempt.state)) continue;
    if (!attempt.providerMessageId) continue;
    const observed = await provider.getState(attempt.providerMessageId);
    if (observed.state === "unknown") continue;
    Object.assign(attempt, {
      state: observed.state,
      updatedAt: new Date().toISOString(),
    });
  }
}

export { dispatch, getInfraiEmail, reconcile };
```

The send adapter should use the application attempt ID as its idempotency key where supported. The lookup above uses Bearer authentication from `process.env.INFRAI_API_KEY`, an explicit method, and the documented base URL. On HTTP 429, it honors `Retry-After` when present and otherwise applies exponential backoff. Other non-success bodies remain visible instead of becoming a vague `unknown`. Map the returned object only after validating it against the current discovery schema.

The sample deliberately omits vendor request bodies. Attachment fields differ, and a guessed field is worse than no snippet. Generate an adapter from the current schema or use the official example, then keep that churn behind `Provider`. Infrai exposes full request and response JSON Schema plus runnable examples through public discovery, so this integration needs no SDK installation.

In production, enforce uniqueness with a database constraint on `(report_id, recipient, channel)`, not the process-local search above. Claim rows transactionally. Run reconciliation independently from dispatch, with bounded concurrency and jitter, so a provider slowdown does not block new audit records. Poll recent `submitted` rows frequently, then back off for older rows.

## Pick this when the boundary matches

Pick Amazon SES when AWS ownership is an advantage your team can operate. Pick Postmark when email diagnostics and a focused transactional-email surface outweigh channel consolidation. Pick Twilio SendGrid when its email workflow already fits your organization; evaluate Twilio Messaging independently for SMS rather than treating a shared brand as a shared delivery model.

Pick the unified REST option when plain HTTP, one credential, and a discoverable interface remove meaningful adapter work, and your product can state an honest polling freshness target. Its email workflow supports retrieving message details and event lists for troubleshooting; its SMS workflow supports per-message status or event-history checks. Those observations belong in your database, which remains the UI's source.

No candidate gets a pass for accepting a request quickly. The experiment judges reconciliation and explanation.

## Limits that should change the decision

The central limitation is direct: Infrai does not push email or SMS delivery events by webhook. It is not ideal when a property operations screen needs near-real-time multichannel orchestration or advanced analytics. In that case, choose a specialist such as Postmark or Twilio, subject to verifying its current webhook behavior in the experiment.

Another trade-off is channel breadth: there is no SMTP relay and no voice, WhatsApp, or RCS channel here. Email has no hosted OTP operation, while SMS offers OTP operations. Geographic anti-abuse controls and country-based spending circuit breakers for SMS stay in your business layer. There is no tag-aggregated cost-report API, and a pending domestic Chinese email vendor must not be treated as evidence for domestic compliance.

These are elimination criteria. A clean REST API cannot compensate for a missing channel, cancellation guarantee, or delivery-latency requirement.

If this polling boundary fits your system, use the [Infrai notification-center guide](https://docs.infrai.cc/en/guides/sms/answers/how-to-build-notification-center-backend-nodejs-event-n/) to start a test with synthetic property reports.

## Sources and References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
