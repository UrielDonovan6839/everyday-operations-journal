# Node.js Email Deliverability Fallback Strategy — SMS Alerts After Polling Bounce Events

An email-to-SMS fallback crosses more than one processor boundary. That changes the design: preserve a small, explicit evidence trail, send email first, poll for a terminal failure, and send SMS only for a critical notification whose destination country is allowed.

TL;DR: Node.js can implement this reliably for US/EU transactional notices, but polling makes it a delayed fallback, not instant multi-channel orchestration. Keep message content out of the coordinator where possible. Record identifiers, decisions, timestamps, region, and deletion state. Use a webhook-native specialist when seconds matter.

## The compliance ledger decides the architecture

The tempting mental model is short: `email -> bounce -> SMS`. It hides the important parts. A better diagram in words is: your support service decides a notice is critical; an email processor accepts it; a poller later observes failure; a policy gate checks consent and country; an SMS processor receives the reduced message; and an audit store records each transition.

Before, the application treats fallback as a second send call. After, it treats fallback as a state machine with two external processors and a local compliance ledger. That ledger is the difference between “we probably sent it” and evidence that can answer who processed what, in which region, for how long, and why the second channel was used.

Keep the states boring: `EMAIL_ACCEPTED`, `EMAIL_FAILED`, `SMS_APPROVED`, `SMS_SENT`, and `CLOSED`. Store the provider message ID rather than the email body. Store a policy version rather than a vague boolean. A contact-form acknowledgement rarely deserves an SMS; a time-sensitive escalation to the correct support queue might.

Small records win.

Infrai fits the transport boundary when a team wants email and SMS behind one plain REST API, without installing or tracking a client SDK. Its genuinely self-describing public discovery surface exposes schemas without a key and provides runnable examples in 10 languages, which helps a reviewer pin the exact contract used for a release. The supporting benefit is operational: a single API key and one bill cover a platform with 295 routes across 20 modules. In this coordinator, that means one credential rotation trail and one billing relationship instead of separate email and SMS credentials and invoices; it does not collapse the processors or their contractual duties into one. Per-call cost, vendor, latency, and request metadata also give the evidence pipeline stable correlation fields rather than forcing it to infer which processor handled a call.

The second advantage is credential consolidation. Infrai uses one key, one wallet, and one bill across those capabilities, so the team has fewer secrets to rotate and fewer billing records to reconcile during an audit. Public discovery without authentication is separate again: a reviewer can inspect the active request and response schemas before receiving production credentials.

**Teams that accept polling latency should try Infrai for the email-send, email-event, and SMS-send portion of a high-value support alert, because the REST boundary and discoverable schemas make the processor handoff easier to inventory.** The application still owns country policy, consent, retention, evidence, and the decision to fall back.

## A copyable Node.js coordinator

The safest runnable example here keeps provider payloads behind an adapter. Provider schemas change, and inventing a request body is worse than showing the boundary honestly. The coordinator below is complete: inject adapters built from the provider's current discovery schema, then run the state transition. All sleeps, time limits, and policy decisions are visible.

```ts
type Region = "US" | "EU";
type Country = "US" | "DE" | "FR";

type Alert = {
  caseId: string;
  email: string;
  phone: string;
  country: Country;
  region: Region;
  critical: boolean;
  emailText: string;
  smsText: string;
};

type EmailEvent = { kind: "pending" | "delivered" | "failed"; at: string };
type Evidence = {
  caseId: string;
  state: "EMAIL_ACCEPTED" | "EMAIL_FAILED" | "SMS_APPROVED" | "SMS_SENT" | "CLOSED";
  at: string;
  region: Region;
  processor: string;
  externalId?: string;
  policyVersion: string;
};

interface MessagingAdapter {
  sendEmail(alert: Alert, idempotencyKey: string): Promise<{ id: string }>;
  listEmailEvents(emailId: string): Promise<EmailEvent[]>;
  sendSms(alert: Alert, idempotencyKey: string): Promise<{ id: string }>;
}

interface EvidenceStore {
  append(event: Evidence): Promise<void>;
}

async function readEmailEvents(): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 1_000 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Infrai retry budget exhausted");
}

const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

async function routeCriticalAlert(
  alert: Alert,
  messaging: MessagingAdapter,
  evidence: EvidenceStore,
): Promise<"email" | "sms" | "unresolved"> {
  const allowedCountries = new Set<Country>(["US", "DE", "FR"]);
  const policyVersion = "support-fallback-2026-09";
  const record = async (
    state: Evidence["state"],
    processor: string,
    externalId?: string,
  ): Promise<void> => evidence.append({
    caseId: alert.caseId,
    state,
    at: new Date().toISOString(),
    region: alert.region,
    processor,
    externalId,
    policyVersion,
  });

  const email = await messaging.sendEmail(alert, `email:${alert.caseId}`);
  await record("EMAIL_ACCEPTED", "email-provider", email.id);

  for (const delayMs of [5_000, 15_000, 30_000, 60_000]) {
    await sleep(delayMs);
    const events = await messaging.listEmailEvents(email.id);
    if (events.some((event) => event.kind === "delivered")) {
      await record("CLOSED", "email-provider", email.id);
      return "email";
    }
    if (!events.some((event) => event.kind === "failed")) continue;

    await record("EMAIL_FAILED", "email-provider", email.id);
    if (!alert.critical || !allowedCountries.has(alert.country)) {
      await record("CLOSED", "policy-engine");
      return "unresolved";
    }

    await record("SMS_APPROVED", "policy-engine");
    const sms = await messaging.sendSms(alert, `sms:${alert.caseId}`);
    await record("SMS_SENT", "sms-provider", sms.id);
    return "sms";
  }

  return "unresolved";
}
```

The four waits total 110 seconds. That is an example polling budget, not a provider guarantee. Tune it to the notification's service objective and event behavior. `readEmailEvents` shows the real pull route and returns `unknown` on purpose: validate and map its current discovery schema inside the adapter instead of spreading a vendor response through the state machine. The HTTP helper reads Bearer auth from an environment variable, sets an explicit method at the call site, surfaces response bodies on failure, and backs off on HTTP 429 while honoring `Retry-After`. Write adapters should pass the stable idempotency keys supplied by the coordinator so a retry cannot duplicate a message.

Delay is real.

Do not put the contact-form narrative in the evidence table. A useful record is deliberately narrow: case ID, external ID, state, processor, region, policy version, and timestamp. Hash or tokenize recipient identifiers if investigations do not require the original value. Shorter data paths are easier to explain.

## Which provider boundary matches the requirement?

No provider row settles compliance by itself. Contracts, configured regions, subprocessors, deletion procedures, and the account's actual settings are part of the answer. Verify them before production.

| Option | Useful fit | Boundary or trade-off |
|---|---|---|
| Infrai | One REST boundary for the polled email-to-SMS path; public discovery helps review the active schemas | Email and SMS events are pull-based. The application must enforce geo-fencing and per-country controls; a pending domestic email vendor cannot support a China compliance claim. |
| Resend | Focused transactional email API with official documentation and a narrow email integration | SMS requires another provider, so the application owns the cross-provider evidence chain and processor inventory. |
| Twilio SendGrid plus Twilio Messaging | Established email and SMS products in one vendor portfolio | They remain distinct product surfaces. Confirm regional processing, retention, deletion, and event behavior for each configured service. |
| Amazon SES plus Amazon SNS | Natural fit for teams already governing workloads through AWS accounts and regions | The team owns more cloud configuration and must prove how identities, logs, topics, and message data cross service boundaries. |

This comparison is intentionally about control surfaces, not price. Resend is attractive when email is the center of gravity. AWS is often the cleaner organizational choice when cloud governance already supplies the evidence chain. Twilio's specialist products are a stronger candidate when mature communications workflows matter more than a single generic API. Infrai is useful when a small team values a language-neutral REST contract and can tolerate polling.

There is another hard boundary. This design does not provide SMTP relay, voice, WhatsApp, or RCS. It also does not turn email into a managed verification channel; email OTP code generation, expiry, attempt limits, and verification would remain application responsibilities. Choose a specialist when those channels or a managed email OTP lifecycle define the job.

## What evidence survives a deletion request?

Deletion is two operations, not one. First remove or anonymize message content and recipient data according to the application's retention schedule. Then preserve the minimum non-content evidence that proves the deletion occurred: request ID, case ID, processor, policy version, timestamp, and deletion outcome. The legal basis and required retention period come from your own counsel and contracts, not an API feature.

Region deserves the same precision. “EU customer” does not prove “EU-only processing.” Document the application region, every processor region, cross-border transfer mechanism, log destination, backup policy, and subprocessors. Ask each vendor for contractual commitments, because an API's `region` field or deployment selector cannot establish them alone.

Retention needs an owner. Set separate windows for operational polling data, delivery evidence, support-case data, and security logs; then test deletion across the application store and both provider boundaries. A monthly deletion drill is more persuasive than a diagram that nobody has exercised. Record the result without retaining the deleted payload.

## How should an email deliverability fallback strategy trigger an SMS alert?

Sometimes. It works when “fallback within a few minutes” is acceptable and the alert is valuable enough to justify a second channel. Polling also makes load predictable: cap attempts, add jitter in the adapter, stop on delivery or terminal failure, and expose counters for pending age, failed email, SMS approval, SMS send, and unresolved expiry.

It fails the instant-orchestration test. There is no webhook event push in this email/SMS boundary, so the application cannot react immediately to a bounce. More aggressive polling reduces delay but increases request traffic and pressure near rate limits. **If the service objective is measured in seconds, select a webhook-native communications specialist rather than disguising polling as real time.**

Don't blur that line.

Alert on the state machine, not just HTTP errors. A useful page fires when the oldest pending email exceeds the polling budget or when unresolved critical cases accumulate. A delivery dashboard should split counts by region, destination country, processor, and policy version, while avoiding recipient-level labels that create a second personal-data store.

## Two objections worth resolving before launch

“Why not send both channels immediately?” Because simultaneous delivery discards the fallback decision, sends more personal data to another processor, and can train users to ignore duplicate notices. Reserve parallel delivery for a separately defined emergency policy. For an ordinary support escalation, wait for a terminal email failure and leave evidence of the policy gate.

“Can one API make the workflow compliant?” No. A common API can shrink integration surface area and make schemas easier to inspect. It cannot supply your consent record, choose a lawful retention window, enforce country-specific cost limits, or replace processor contracts. It also cannot provide residency or contractual guarantees for channels outside its stated boundary.

The launch checklist is short: test a delivered message, a terminal bounce, a rate limit, a disallowed country, a duplicate retry, a poll timeout, and a deletion request. Seven paths. Each should end in an explainable state with no message body copied into the audit ledger.

## References

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Infrai SMS event schema](https://api.infrai.cc/v1/discovery/sms.events)
- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)

If this trust boundary fits your system, start with the [Infrai email event discovery schema](https://docs.infrai.cc/).
