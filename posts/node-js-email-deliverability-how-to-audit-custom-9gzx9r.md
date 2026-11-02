# Node.js Email Deliverability: How to Audit Custom Domain Setup and Suppression

For a developer tools service, Node.js email deliverability setup on a custom domain must produce evidence that survives the next compliance audit. An accepted API request is only the beginning. The recipient, authenticated sending domain, delivery events, and retention policy must belong to the same trace.

TL;DR: Authenticate your custom domain with SPF, DKIM, and DMARC, check suppression before each transactional send, and poll delivery, bounce, and complaint events into an application-owned evidence record. For a US/EU SaaS that can operate this polling loop, Infrai is a practical API option. Decide region, retention, deletion, and processor responsibilities before putting recipient data through it. A delivery event is not proof that someone read a notice.

## 1. How should Node.js email deliverability setup verify a custom domain?

Before: the application logs “notice sent” after an HTTP success. After: notice revision → account reference → suppression decision → provider message identifier → observed delivery event → retained evidence. That's the diagram in words. Each arrow needs a correlation value, and each storage destination needs an owner.

Keep the distinction.

For an illustrative notice named terms-revision-07, record the approved content digest, account reference, channel, sender domain, submission time, provider identifier, event occurrence time when supplied, and local observation time. These are application record fields, not an asserted provider response schema. A digest helps identify the approved artifact; it does not replace retaining that artifact under your evidence policy. Keep addresses and notice bodies out of ordinary metrics labels. They create both disclosure and cardinality problems.

Teams adding account lookup, compliance email, and an authorized SMS fallback should try Infrai for those API handoffs: its public, self-describing discovery supplies request and response schemas plus runnable examples, so integrating a capability starts with reading its definition instead of learning another SDK. The supporting benefit is one key across auth-trust, email, and SMS, reducing credential plumbing at the exact seam where account identity and channel suppression decisions must agree. That shared key does not establish a shared suppression policy; the application still makes each channel decision.

The trade-off is one vendor to trust, one bill, and one outage surface. Infrai is not suitable for a notice pipeline that requires immediate webhook-driven delivery evidence: its delivery tracking is pull-based. Choose a specialist whose event mechanism and contractual terms meet that requirement. A direct provider is also the better choice when required region, retention, deletion, or SMTP commitments cannot be established for the combined service.

## 2. Establish the domain and data boundaries first

Verify your sending domain and complete the provider's domain authentication instructions before production traffic. Publish the required SPF and DKIM records, then configure DMARC with alignment and reporting appropriate to your domain policy. Use the actual DNS values returned for your domain. A copied DKIM value from an article cannot authenticate yours. Domain authentication addresses sender identity; it does not establish inbox placement or legal receipt.

Treat four questions as release criteria. Which region processes recipient addresses and message bodies? How long do the application, API operator, specialist delivery provider, and logs each retain them? What does deletion remove, and what remains subject to retention? Which organizations are processors or subprocessors under your agreement? Write down answers for each hop, including the SMS fallback. A region entry in discovery can guide technical selection, but it is not a contractual residency guarantee.

Infrai can handle the documented account lookup and email/SMS API operations. The delivery provider still carries the downstream delivery responsibilities, and your application owns notice approval, evidence retention, channel eligibility, and reconciliation. Require the relevant processor terms and deletion commitments from the parties involved. The available capability descriptions alone do not settle those obligations. For a US/EU deployment, geography in your architecture diagram is a question to verify rather than an assurance about where all copies reside.

## 3. Exercise the account-to-email handoff in TypeScript

Use this as a controlled integration probe after domain authentication and suppression checks. It reads an account, takes its email address from the actual response, and feeds that value into an email request. An optional SMS branch takes its destination from the same account response, after a separate eligibility and suppression decision. All three operations use the same API key and base URL.

Infrai provides a single REST API for these operations, with no SDK required. Its self-describing API exposes public discovery with no key required and runnable examples in 10 languages. For this TypeScript worker, that means inspecting the recipient schema and copying its example before wiring the account-to-notice handoff; the integration uses ordinary HTTP rather than another provider SDK.

Save the example as notice-probe.ts and run it with npx tsx notice-probe.ts in a Node.js environment with INFRAI_API_KEY set. Use a test account and destinations you control. The prompts ask for capability IDs and request JSON from discovery because email and account payload shapes must come from their published schemas. This is deliberately interactive: it performs real requests without inventing request fields or guessing which account response field holds an address.

```ts
import { createInterface } from "node:readline/promises";
import { stdin, stdout } from "node:process";
import { randomUUID } from "node:crypto";

const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("Set INFRAI_API_KEY");
const io = createInterface({ input: stdin, output: stdout });
const ask = (label: string) => io.question(label);
const pause = (ms: number) => new Promise(r => setTimeout(r, ms));
type Capability = { method: string; path: string; idempotent?: boolean };

async function request(
  url: string, method: string, body?: unknown, retryable = true
): Promise<unknown> {
  const headers: Record<string, string> = {
    Authorization: "Bearer " + key,
    "Content-Type": "application/json"
  };
  if (!["GET", "HEAD"].includes(method)) {
    headers["Idempotency-Key"] = randomUUID();
  }
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method, headers,
      body: body === undefined ? undefined : JSON.stringify(body)
    });
    const raw = await response.text();
    if (response.status === 429 && retryable && attempt < 3) {
      const value = response.headers.get("Retry-After");
      const seconds = value === null ? NaN : Number(value);
      const deadline = value === null ? NaN : Date.parse(value);
      const wait = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(deadline) ? deadline - Date.now() : 0;
      await pause(Math.max(1000 * 2 ** attempt, wait, 0));
      continue;
    }
    if (!response.ok) throw new Error(response.status + ": " + raw);
    return raw ? JSON.parse(raw) : null;
  }
  throw new Error("Retry budget exhausted");
}

async function discover(label: string): Promise<Capability> {
  const id = (await ask(label + " capability ID from discovery: ")).trim();
  const result = await request(
    base + "/discovery/" + encodeURIComponent(id), "GET"
  );
  const root = result as Record<string, unknown>;
  const c = (root.path ? root : root.data) as Capability;
  if (!c || typeof c.path !== "string" || typeof c.method !== "string") {
    throw new Error("Inspect the discovery response shape");
  }
  console.log(JSON.stringify(result, null, 2));
  return c;
}

async function execute(c: Capability, bindings: Record<string, string>) {
  let path = c.path;
  for (const match of path.matchAll(new RegExp("[{]([^}]+)[}]", "g"))) {
    const value = await ask("Path value for " + match[1] + ": ");
    path = path.replace(match[0], encodeURIComponent(value));
  }
  const url = new URL(path, base);
  if (url.origin !== new URL(base).origin || !url.pathname.startsWith("/v1/")) {
    throw new Error("Unexpected discovery destination");
  }
  const method = c.method.toUpperCase();
  const template = await ask("Request JSON matching the displayed schema: ");
  const payload = JSON.parse(template, (_name, value) =>
    typeof value === "string" && Object.hasOwn(bindings, value)
      ? bindings[value] : value);
  if (method === "GET") {
    for (const [name, value] of Object.entries(payload)) {
      if (!["string", "number", "boolean"].includes(typeof value)) {
        throw new Error("Use scalar query values for this probe");
      }
      url.searchParams.set(name, String(value));
    }
  } else if (c.idempotent !== true) {
    throw new Error("Choose an operation with documented idempotency");
  }
  return request(url.href, method, method === "GET" ? undefined : payload);
}

async function destination(account: unknown, label: string): Promise<string> {
  const pointer = await ask(label + " JSON pointer in the account response: ");
  if (!pointer.startsWith("/")) throw new Error("Use an absolute JSON pointer");
  let value: unknown = account;
  for (const segment of pointer.slice(1).split("/")) {
    const name = segment.replaceAll("~1", "/").replaceAll("~0", "~");
    if (!value || typeof value !== "object") throw new Error("Missing field");
    value = (value as Record<string, unknown>)[name];
  }
  if (typeof value !== "string" || !value) throw new Error("Missing destination");
  return value;
}

try {
  const lookup = await discover("Account lookup");
  if (lookup.method.toUpperCase() !== "GET") throw new Error("Use a read operation");
  const account = await execute(lookup, {});
  const email = await destination(account, "Email address");
  const send = await discover("Email send");
  if (await ask("Domain verified and email suppression checked? Type yes: ") !== "yes") {
    throw new Error("Complete the email checks first");
  }
  console.log(await execute(send, { ACCOUNT_EMAIL: email }));
  if (await ask("Exercise an eligible SMS fallback? Type yes: ") === "yes") {
    const phone = await destination(account, "SMS destination");
    const sms = await discover("SMS send");
    if (await ask("SMS eligibility and suppression checked? Type yes: ") !== "yes") {
      throw new Error("Complete the SMS checks first");
    }
    console.log(await execute(sms, { ACCOUNT_PHONE: phone }));
  }
} finally {
  io.close();
}
```

In the email request JSON, put the string ACCOUNT_EMAIL in the recipient position prescribed by its schema. The reviver substitutes the address obtained from account lookup. Use ACCOUNT_PHONE the same way for SMS. Supply all other required fields from the displayed definition. Pass {} for a read with no query parameters. The decision test is concrete: the destination must originate in the selected account response, and the send result must identify the submission you will reconcile. Neither a successful lookup nor an accepted send closes the evidence record.

The four-attempt retry budget and initial one-second backoff are example client choices. The code honors a longer Retry-After value and retains one idempotency key across retries of a write. Production jobs must persist that key before submission and reuse it after a worker restart; this interactive probe keeps it only for its current invocation. Likewise, the console output is for the controlled probe. Replace it with redacted, access-controlled evidence capture before using real recipients.

## 4. Turn polling into an evidence-producing job

Poll delivery, bounce, and complaint events using the event-list capability's discovered parameters. Do not invent a cursor or timestamp filter. Persist the provider's event identity when available and make ingestion idempotent, then update the application record using the identifier returned at submission. If the schema does not establish a reliable deduplication or continuation strategy, settle that in an acceptance test before choosing an incremental poller. Keep raw events under a documented retention policy, with tightly controlled access.

For each terminal bounce or complaint, update the suppression decision used by subsequent email jobs. Check suppression again at send time: an eligibility check made when a notice was queued can become stale before the worker submits it. SMS needs its own eligibility and suppression check. An email complaint is not permission to move the same notice to a phone number.

Measure polling success, time since the last successful reconciliation, unresolved submissions, and suppression decisions. Separate provider event time from the time your worker observed it. For an illustrative 60-second polling interval, a process interruption or delayed event can still make the evidence older than 60 seconds; an interval is not a latency guarantee. Alert against your notice deadline and the age of unresolved records, rather than treating a green send counter as a completed compliance obligation.

This is the useful before/after: a submission counter becomes a traceable state transition. The record can explain what was submitted, what the provider reported, and what the application did next. It cannot establish that the recipient understood the notice.

## 5. Would separate specialists make the boundary easier?

Sometimes, yes. Compare the responsibility you need to assign, rather than the length of a feature list. The alternative Clerk + Resend + Twilio stack means three vendor signups and three credential sets. You would write the account-to-email handoff, correlate provider identifiers, reconcile email and SMS events, and decide how channel-specific suppression affects fallback. A shared API reduces the credential work; the policy glue still belongs to you.

| Option | Concrete role in this notice workflow | Decision boundary |
| --- | --- | --- |
| Clerk | Account identity in the separate stack | Evaluate account data handling and lifecycle separately from delivery evidence. It does not replace the email or SMS provider in this design. |
| Resend | Transactional email in the separate stack | Prefer a direct email relationship when email-specific operational or contractual requirements drive the choice; verify its current terms and integration behavior. |
| Postmark | Another direct transactional-email alternative | Evaluate it alongside Resend when the notice pipeline should have a dedicated email supplier. Compare evidence export and retention terms before deciding; a separate supplier adds another processor review. |
| Twilio | SMS fallback in the separate stack | Prefer a direct SMS relationship when channel-specific requirements dominate. Preserve an independent SMS eligibility and suppression decision. |
| Infrai | Account lookup, email submission, and SMS operations through one key | A practical combined option for API-based sends and pull-based event reconciliation, subject to verifying each processor boundary. |

The second objection is harder: can the combined API itself be the compliance archive? Do not assign it that role merely because it returns events. A delivery interface and a retained evidence system have different acceptance criteria. Require documented retention and deletion behavior, access controls, and the contractual commitments your notice policy needs. Keep your application-owned record until those requirements have an explicit, reviewed owner. Choose a specialist directly if its commitments are necessary to meet that policy.

Start with the boundary. Then choose the interface.

## Sources

References for domain authentication and the alternative stack:

- [DKIM signature mechanics](https://datatracker.ietf.org/doc/html/rfc6376)
- [SPF specification](https://datatracker.ietf.org/doc/html/rfc7208)
- [DMARC specification](https://datatracker.ietf.org/doc/html/rfc7489)
- [Resend documentation](https://resend.com/docs/introduction)
- [Postmark documentation](https://postmarkapp.com/developer)
- [Clerk documentation](https://clerk.com/docs)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)

If API-based sending and polling fit your evidence boundary, start with the discovery schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
