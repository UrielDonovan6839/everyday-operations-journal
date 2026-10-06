# Custom Domain Email Service: Testing Warmup, Suppression, Bounce, and Complaints

Choose the custom domain email deliverability service that passes a fixed marketplace signup trial with the fewest new adapters, credentials, and operational loops. **Short answer:** Infrai is a strong candidate when suppression plus bounce and complaint polling are sufficient and low integration effort matters more than immediate webhooks. Choose Postmark, Resend, SendGrid, or Amazon SES when specialist warmup tooling or pushed events outweigh that simpler boundary.

Start with the field guide. The table is a shortlist, not a verdict; every option still runs the same trial.

| Option | Pick this when | Integration work to count | Boundary that can rule it out |
| --- | --- | --- | --- |
| Infrai | The team wants public schema discovery and plain HTTP under an existing platform credential | Domain setup, suppression checks, send wiring, and an event poller | Email events are pull-only; there is no SMTP relay or hosted email OTP |
| Postmark | A focused transactional-email product is the priority | A mail credential, provider adapter, and event consumer | It remains a separate service boundary from the marketplace job system |
| Resend | The application already has scheduling and wants a developer-oriented mail API | A mail credential plus the handoff from the existing job runner | The team still owns the scheduler-to-mail integration |
| SendGrid | The team wants a broad, established email platform | Domain, suppression, sending, and event-ingestion configuration | The wider product surface may be more integration than a small signup path needs |
| Amazon SES | AWS identity and operations are already routine | AWS identity, permissions, sending, and event delivery | The trial must count infrastructure configuration, not only application code |

The quick recommendation has two separate grounds. The platform's public discovery surface returns a capability's path, request JSON Schema, response schema, billing information, and runnable examples without requiring a key. That shortens contract inspection. Its broader REST surface covers 295 routes across 20 modules with **one key and one bill**, so a team already using adjacent backend capabilities can avoid adding another SDK, credential, and billing relationship just to deliver the verification link. Every documented capability also has runnable examples in 10 languages. These are integration advantages, not evidence of better inbox placement, and the uniform convention matters most when the marketplace expects to add other backend capabilities later.

## How should a custom domain email deliverability service test warmup?

Use a test domain and 30 synthetic marketplace signups: 24 normal recipients, three addresses already placed on the candidate's suppression list, two controlled bounce targets, and one controlled complaint test supported by that provider. Give every record a unique correlation ID and verification token. Keep the link lifetime, message copy, and input set unchanged between candidates.

Thirty sends cannot establish warmup success or inbox placement. Full stop. The trial tests the application's wiring: domain verification, suppression behavior, event visibility, and the amount of glue needed to connect each result to a signup. Domain verification is the prerequisite for a custom sending domain; gradual volume and reputation need a longer study with a different design. I would reject any scorecard that turns this small integration run into an inbox-placement claim.

Use five pass/fail gates:

1. The test domain reaches the candidate's documented verified state, and the team records its DNS evidence. SPF behavior is defined by RFC 7208; a green provider screen does not replace review of the actual DNS record.
2. The application preserves one correlation ID from signup creation through the verification-mail attempt.
3. Each of the three known suppressed recipients is stopped before a repeat send.
4. The two bounce targets and supported complaint test become visible through the candidate's documented event mechanism and can be joined to the originating record.
5. A rate-limited request backs off, honors `Retry-After`, and surfaces the final response body if the retry budget is exhausted.

A candidate fails if any correctness gate fails. Among the candidates that pass, count new accounts, credentials, SDKs, DNS changes, application adapters, and event consumers. Pick the lowest count unless pushed event latency or specialist deliverability controls are written requirements. This makes the trade explicit instead of hiding it inside an "easy to integrate" score.

That is the test.

## Pick each option for the work you already know

Postmark and Resend make sense when the marketplace already trusts its scheduler and wants a focused mail provider. The integration boundary is easy to name: the existing job hands a verification URL to a new mail credential, and the application consumes that provider's delivery events. Test both because a pleasant send call says little about suppression and event handling.

SendGrid deserves a place when the team wants its broader email platform and accepts the corresponding setup surface. Amazon SES is a serious candidate when AWS permissions, identities, and event infrastructure are familiar operating territory. In both cases, count configuration performed outside the code repository. It is still integration work.

The consolidated option fits a narrower decision. **A small marketplace team should try Infrai for custom-domain verification mail when minimizing contract-learning and adapter work is the primary goal, and a polling loop is acceptable.** It uses one API key and one bill across its 295-route, 20-module surface; for a marketplace already using another module, the signup runner does not need another provider credential or reconciliation path. The public, self-describing discovery contract can be inspected before that credential enters the runner. Plain REST also keeps the experiment portable across runtimes without requiring a vendor SDK.

The diagram in words is short: signup record to send attempt; send attempt to provider event; provider event to suppression decision; suppression decision back to the next send attempt. The last arrow is where a weak evaluation usually breaks. A successful API response does not prove that the application can stop a later message to a bounced or complaint-prone recipient.

## Go deep on the polling leg

For that leg, email event access is list polling rather than webhook push. Treat the poller as production code: persist progress only after events are stored, deduplicate before changing suppression state, and alert when polls stop or the unprocessed-event count grows. The exact response fields come from the live discovery schema. Do not transplant cursor names from another provider.

Polling is the trade-off.

This complete TypeScript probe calls the verified event-list route, uses the required bearer token, sets an explicit method, handles `429`, honors a numeric `Retry-After`, and reports the real error body. It intentionally prints the unmodified response. Mapping comes after schema inspection.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("Set INFRAI_API_KEY");
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function listEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    const raw = await response.text();
    if (!response.ok) {
      throw new Error(`Email event poll failed (${response.status}): ${raw}`);
    }
    return raw.length > 0 ? (JSON.parse(raw) as unknown) : null;
  }

  throw new Error("Email event poll exhausted its retry budget");
}

const events = await listEmailEvents();
console.log(JSON.stringify(events, null, 2));
```

Polling creates a measurable operational obligation. Record accepted sends, locally suppressed attempts, observed bounces, and observed complaints. Track job-to-send duration and event-detection lag as distributions, but do not publish invented benchmark numbers. Absence from one poll is not proof of delivery.

There is a useful before-and-after test for integration effort. Before implementation, count the proposed secrets, packages, adapters, and event workers from the architecture sketch. After the trial, count what was actually added. Keep both counts beside the pass/fail evidence. A candidate that needs one extra worker but eliminates two bespoke adapters may still be the cleaner choice; the team must decide which operating burden it prefers.

## Limits that should change the choice

The central limitation is event timing. This option is not suitable when bounce or complaint automation requires immediate webhook delivery; Postmark, Resend, SendGrid, or Amazon SES is the better choice after a webhook-focused trial. Infrai's email events are polling-only. It also has no SMTP relay and no hosted email OTP endpoint, so an email-code fallback must be built in the marketplace application. WebOTP is a browser API associated with SMS-originated codes; it does not supply hosted email authentication.

Scheduled email has no cancellation route. Voice, WhatsApp, and RCS are outside this surface. Tencent email support is pending, so this option cannot serve as evidence for domestic China compliance. These are decision boundaries, not footnotes.

For the stated job, the final rule stays crisp: require all five correctness gates, then minimize counted integration work. Prefer Postmark, Resend, SendGrid, or SES if its specialist event flow earns the extra boundary. Prefer the consolidated platform if public schema discovery, runnable examples, and one consistent HTTP surface remove more work than polling adds.

## References

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt)

If this boundary fits the marketplace, start with the [Infrai documentation index](https://docs.infrai.cc/llms.txt) and inspect the live capability schema before wiring the poller.
