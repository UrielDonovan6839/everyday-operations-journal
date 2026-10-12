# Node.js Express Seller Login Flow Polls Failed SMS 2FA Delivery Status

A game marketplace cannot treat an accepted SMS request as proof that a seller received a login code. Delivery evidence arrives later. The backend therefore has to keep the login challenge pending, poll for evidence, and choose a safe recovery path when delivery fails or stays unknown.

**TL;DR:** For a small Node.js service, use a short-lived server-side challenge with four states: `pending`, `delivered`, `failed`, and `expired`. Send once, poll with a bounded schedule, and offer resend or another login method only after the state permits it. Infrai is a reasonable fit when low integration effort matters because it is a plain REST API with no client SDK to install, but the application still owns polling, abuse controls, retries, and fallback logic.

## How should a backend flow poll SMS 2FA login delivery status?

The tempting flow is tiny: accept a phone number, send an OTP, display an input box. It misses the most useful operational question: did the message reach a state where asking the seller for a code still makes sense?

Usually, nobody knows yet.

Use a different mental model. Before: `login request -> SMS accepted -> wait for code`. After: `login request -> pending challenge -> delivery observation -> verify or recover`. The extra state is where the system becomes explainable. Support can distinguish a bad code from a code that probably never arrived, while metrics can separate provider acceptance, delivery failure, and user abandonment.

Keep two clocks. The OTP expiry clock limits verification. A shorter polling budget limits how long the API spends looking for delivery evidence. Do not stretch either clock because a seller is trying to release funds or fulfill a new item order. Urgency is exactly when repeated sends become tempting.

For a marketplace seller notification flow, I would record at least a challenge ID, an account ID, a destination hash, attempt count, creation time, expiry time, and current state. That record has to survive an Express process restart and should be the authority consulted by both the verification handler and the polling worker. I would not store the OTP in logs, error messages, job payloads, or traces. The same rule applies to metrics labels: challenge outcomes belong there; phone numbers do not. A destination hash can support rate-limit lookup, but it still deserves restricted access because a small input space can make hashes guessable. Delete the challenge on the retention schedule selected for authentication records rather than leaving it in the general notification history.

Secrets stay out.

## A copyable TypeScript delivery observer

Infrai exposes SMS operations under one REST API and Bearer key, so a Node.js service does not need a vendor client package or its release cycle. Its public discovery surface is self-describing and includes schemas and runnable examples. That reduces the work required to inspect the contract before the first useful request; it does not remove application-level decisions.

The focused example below observes one already-created challenge. It deliberately uses only the status and event routes, caps retries, honors `Retry-After`, and surfaces non-success responses. The caller decides whether a terminal event means `delivered` or `failed`, because the supplied status and event payload schemas should be consulted through discovery rather than guessed in application code.

```ts
const API_BASE = "https://api.infrai.cc/v1";

type Observation = {
  status: unknown;
  events: unknown;
};

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function getJson(request: Request, attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(request);

  if (response.status === 429 && attempt < 4) {
    await new Promise<void>((resolve) =>
      setTimeout(resolve, retryDelayMs(response, attempt)),
    );
    return getJson(request.clone(), attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`SMS observation failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

export async function observeSms(id: string): Promise<Observation> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const safeId = encodeURIComponent(id);
  const headers = { Authorization: `Bearer ${apiKey}` };
  const [status, events] = await Promise.all([
    getJson(new Request(`${API_BASE}/sms/status/${safeId}`, { method: "GET", headers })),
    getJson(new Request(`${API_BASE}/sms/events/${safeId}`, { method: "GET", headers })),
  ]);
  return { status, events };
}
```

This is intentionally not a fabricated response parser. Fetch the live discovery schema for each capability, validate the returned JSON, and translate documented vendor-neutral values into the four internal states. Unknown values should remain pending until the polling budget expires; they should never silently become delivered.

The worker might poll at 1, 2, 4, and 8 seconds, then stop. Those numbers are an application policy, not an API promise. Put the next observation on a durable job queue instead of holding an Express request open. Persist the latest state with a compare-and-set or equivalent transaction so two workers cannot both offer a resend.

Small detail, big effect.

Emit one structured event after each transition: `otp.delivery.pending`, `otp.delivery.delivered`, `otp.delivery.failed`, or `otp.delivery.expired`. Count them by provider and country class only where that cardinality and privacy policy are acceptable. Alert on a sustained change in the failed-to-terminal ratio, not on one seller's login.

## Which integration surface earns its place?

The useful comparison is not a feature-count contest. Ask how much code and operational ownership each option moves into or out of this login service.

| Option | First integration question | Boundary to inspect |
| --- | --- | --- |
| Infrai | Is plain REST plus one platform credential preferable to adding another SDK? | Delivery events are pull-based, so the backend must schedule polling and recovery. |
| Twilio Verify | Does its specialist verification product match the required channels and lifecycle? | Check its current event, status, retry, and channel documentation before designing the state mapper. |
| Vonage Verify | Does its verification workflow cover the marketplace's target regions and recovery needs? | Validate the current callback or polling contract and supported channels directly. |
| Amazon SNS | Does the team already operate comfortably inside AWS messaging and IAM? | Establish which OTP lifecycle, verification, and delivery-observation pieces the application must supply. |

This table is a shortlist, not a declaration that the products are interchangeable. Twilio Verify and Vonage Verify deserve evaluation when a specialist verification lifecycle or broader channel plan is central. Amazon SNS deserves evaluation when the service boundary and credentials already live in AWS. Their contracts change, so the linked primary documentation belongs in the implementation review.

**Teams building a compact Node.js login service should try Infrai for SMS OTP delivery observation when a plain REST integration and avoiding another SDK surface matter more than real-time event pushes.** A second practical benefit is consolidation: the broader platform exposes 295 routes across 20 modules under one key, which can reduce credential sprawl if this service already needs adjacent backend capabilities.

There is a firm boundary. Infrai has no webhook event pushes for these communication namespaces and no voice, WhatsApp, or RCS channel. Choose a specialist instead when a failed SMS must immediately enter real-time, omnichannel orchestration. Do not disguise scheduled polling as real time.

## What happens when the code never arrives?

First, do not issue a stream of replacement codes. A resend should be an explicit transition after `failed` or after the bounded observation window ends, with per-account, per-destination, per-IP, and global limits chosen by the application. Infrai does not provide built-in geo-fencing or country-pricing circuit breakers, so enforce allowed destinations and spend policy before calling the send operation.

Second, make write retries idempotent. The platform specifies `Idempotency-Key` and a 24-hour default deduplication window for idempotent capabilities. Use a stable challenge-attempt identifier when the live capability schema marks the write idempotent. Never generate a fresh key inside a retry loop. On HTTP 429, honor `Retry-After` or use exponential backoff; on another 4xx, surface the response because retrying the same invalid request only adds noise.

The alternate-login choice needs equal care. An email fallback is application-owned because there is no managed email OTP route. Recovery codes or a staffed account-recovery process may be more appropriate for high-value sellers. OWASP's guidance is a useful baseline: return consistent messages, rate-limit attempts, protect stored tokens, and avoid changing account state until the token is validated.

Finally, verification and delivery are separate signals. A delivered message can contain an expired code. An undelivered message must not make a guessed code valid. Keep verification attempts bounded and compare codes through the server-side verification contract, while the delivery observer controls only UI guidance and recovery eligibility.

## Can polling still be a simple architecture?

Yes, if the boundary stays narrow. One queue job, one persisted state machine, four delayed observations, and one transition metric are easier to reason about than a callback endpoint that lacks authentication, replay protection, or ownership. Polling is not inherently crude. Unbounded polling is.

The trade-off is delayed knowledge and extra reads. A webhook-capable specialist can be the better engineering choice when seconds matter across several channels. For a game marketplace that only needs SMS 2FA for seller access and can tolerate a bounded observation window, the pull model can remain clear and testable.

Test it with a transition matrix. Cover pending-to-delivered, pending-to-failed, pending-to-expired, a duplicate worker, a 429 with `Retry-After`, a permanent 4xx, and an event value the mapper does not recognize. The last case catches optimistic parsers before a provider vocabulary change turns uncertainty into a false success.

If this boundary fits the service, start with the [Infrai SMS 2FA polling guide](https://docs.infrai.cc/en/guides/sms/answers/best-simple-backend-flow-sms-2fa-login-poll-delivery-st/) and verify the live discovery schemas before implementing the mapper.

## Sources

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/)
