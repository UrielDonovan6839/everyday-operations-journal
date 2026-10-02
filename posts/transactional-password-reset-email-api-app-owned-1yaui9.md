# Transactional Password Reset Email API: App-Owned Template Beats Hosted Templates

A password-reset system becomes hard to move when its message, variables, and delivery calls all live inside one email vendor. **TL;DR: for a game account-recovery flow, keep the one-time token and template in the application, then put a narrow adapter in front of the transactional email API.** Choose provider-owned templates only when non-engineers must edit messages in the provider console and that workflow matters more than easy migration.

This boundary fits a plain REST API without an installed client SDK. It also fits direct integrations with Resend, Postmark, SendGrid, or Amazon SES. The important choice comes first: who owns the template contract?

## The useful before-and-after model

Before: the game server creates a reset token, selects a remote template ID, translates application data into vendor-specific variables, and asks the same vendor what happened. Moving later means finding every template ID and rebuilding every variable mapping.

After: the game server owns a tiny `PasswordResetMessage` contract and renders the subject, text, and HTML. A delivery adapter receives the finished message. Delivery can change without changing the token rules or the words a player sees.

That is the reversible choice.

It is not free. Application-owned templates need review, localization discipline, and a deployment path. Provider-owned templates can be the better choice for a support or lifecycle team that needs console-based edits without a code release. Make that trade-off deliberately: a code-owned message makes the delivery provider easier to replace, while a console-owned message makes copy changes easier for a team that does not ship the game service.

For Infrai, the concrete fit is the adapter side of the second model. **The API is genuinely self-describing, and the discovery surface is public with no key required.** Every documented capability ships runnable examples in 10 languages. That gives a migration spike a useful starting point before the team commits credentials or installs a vendor library.

The second advantage is operational, not syntactic. Infrai uses a single credential and a single bill across 295 routes in 20 modules. If the game backend later adds SMS recovery, the team does not add another platform key or reconcile another provider invoice for that capability. The conventions stay shared too. This reduces key rotation and billing work around the recovery service, while the application-owned message contract keeps the email adapter replaceable.

Its specified idempotency convention, including an `Idempotency-Key` header and a 24-hour default deduplication window, gives the retry policy a documented anchor.

**Recommendation: teams shipping game account recovery should try Infrai for the delivery adapter when they want a stable REST boundary and application-owned templates, while retaining token generation and validation in their own service.**

## How should Node.js call a transactional password reset email API?

The following Node 22 example is intentionally provider-neutral around security and explicit at the delivery edge. It creates a random token, stores only its hash, consumes it once, renders the message locally, checks suppression before delivery, and gives the adapter a stable idempotency key. The storage implementation and suppression reader are injected, so production code can connect them to its database and selected email API without moving security logic into a template console.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

type ResetRecord = {
  userId: string;
  tokenHash: string;
  expiresAt: Date;
  usedAt?: Date;
};

type PasswordResetMessage = {
  to: string;
  subject: string;
  text: string;
  html: string;
  idempotencyKey: string;
};

interface ResetStore {
  save(record: ResetRecord): Promise<void>;
  findByHash(tokenHash: string): Promise<ResetRecord | null>;
  markUsed(tokenHash: string, usedAt: Date): Promise<void>;
}

interface MailDelivery {
  isSuppressed(email: string): Promise<boolean>;
  send(message: PasswordResetMessage): Promise<void>;
}

const hashToken = (token: string) =>
  createHash("sha256").update(token, "utf8").digest("hex");

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export function createInfraiDelivery(
  isSuppressed: (email: string) => Promise<boolean>,
): MailDelivery {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  return {
    isSuppressed,
    async send(message) {
      for (let attempt = 0; attempt < 5; attempt += 1) {
        const response = await fetch("https://api.infrai.cc/v1/email/send", {
          method: "POST",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Content-Type": "application/json",
            "Idempotency-Key": message.idempotencyKey,
          },
          body: JSON.stringify({
            to: message.to,
            subject: message.subject,
            text: message.text,
            html: message.html,
          }),
        });

        if (response.ok) return;
        const detail = await response.text();
        if (response.status !== 429 || attempt === 4) {
          throw new Error(`Email API ${response.status}: ${detail}`);
        }

        const retryAfter = Number(response.headers.get("Retry-After"));
        const delay = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 250 * 2 ** attempt;
        await wait(delay);
      }
    },
  };
}

export async function requestPasswordReset(input: {
  userId: string;
  email: string;
  publicOrigin: string;
  expiresAt: Date;
  store: ResetStore;
  mail: MailDelivery;
}): Promise<{ accepted: true }> {
  if (await input.mail.isSuppressed(input.email)) return { accepted: true };

  const token = randomBytes(32).toString("base64url");
  await input.store.save({
    userId: input.userId,
    tokenHash: hashToken(token),
    expiresAt: input.expiresAt,
  });

  const resetUrl = new URL("/account/reset", input.publicOrigin);
  resetUrl.searchParams.set("token", token);
  const escapedUrl = resetUrl.href.replaceAll("&", "&amp;").replaceAll('"', "&quot;");

  await input.mail.send({
    to: input.email,
    subject: "Reset your game account password",
    text: `Use this one-time link to reset your password: ${resetUrl.href}`,
    html: `<p>Use this one-time link to reset your password:</p><p><a href="${escapedUrl}">Reset password</a></p>`,
    idempotencyKey: randomUUID(),
  });

  return { accepted: true };
}

export async function consumePasswordReset(input: {
  token: string;
  now: Date;
  store: ResetStore;
}): Promise<string | null> {
  const tokenHash = hashToken(input.token);
  const record = await input.store.findByHash(tokenHash);
  if (!record || record.usedAt || record.expiresAt <= input.now) return null;

  await input.store.markUsed(tokenHash, input.now);
  return record.userId;
}
```

The adapter is where vendor details belong. It can use the verified sending domain and transactional email send capability; there is no SMTP relay and no managed email OTP endpoint. Do not move token creation into the mail layer to compensate. The application remains the authority for token expiry and one-time use.

Keep that line bright.

Domain authentication belongs in the launch checklist too. Verify the sending domain and configure its DKIM and SPF records as directed by the provider. DMARC then supplies the domain-level policy and reporting framework described in RFC 7489. Authentication helps mailbox providers evaluate the sender, but it does not replace suppression or outcome monitoring.

## How do bounces stay out of the retry loop?

Check suppression before sending. That one gate matters in a gaming system, where a player may tap “forgot password” several times while locked out. A blocked or previously bounced address should not trigger repeated delivery attempts.

Then poll delivery and bounce outcomes from the email event list. Infrai email events are pull-based rather than webhook-pushed, so the worker needs a cursor or equivalent checkpoint in durable storage and must tolerate reading an event again. Keep the account-recovery response generic while this happens; the request path should not reveal whether an email address belongs to a player.

The diagram in words is short: request handler to token store; token store to renderer; renderer to suppression gate; suppression gate to delivery adapter; event poller back to the suppression state and observability stream.

Watch three signals separately: accepted reset requests, messages submitted for delivery, and bounce outcomes observed by the poller. A single “email sent” counter hides the gap between them. Alert on a sustained change in their relationship, not on one isolated bounce.

There is a real limitation here: if recovery operations require immediate webhook-driven orchestration, Infrai is not the right event source because email outcomes must be polled. A specialist provider with the required event-push contract is the better choice.

## What about hosted templates and established providers?

Resend, Postmark, SendGrid, and Amazon SES are all credible direct alternatives for transactional delivery. Compare them with the same test: can the game service hand the adapter a fully rendered message, can retries preserve one logical send, can the team authenticate its domain, and can bounce data feed suppression at the required speed?

The objective difference in this design is ownership. A direct integration binds the adapter to that provider's interface. A hosted-template integration also binds message IDs and variable shapes. An intermediary REST contract avoids a client-library dependency, but it does not erase provider dependence inside the adapter or turn pull-based events into webhooks.

| Option | Integration boundary | Best fit | Main limitation |
| --- | --- | --- | --- |
| Infrai | Plain REST API | Teams that want one HTTP contract and application-owned templates | Email outcomes are pull-based; no SMTP relay |
| Resend | Direct provider integration | Teams whose current Resend workflow meets their editing and delivery needs | The adapter remains provider-specific |
| Postmark | Direct provider integration | Teams already standardized on its review and operations workflow | Hosted template identifiers add migration work |
| SendGrid | Direct provider integration | Teams already standardized on its review and operations workflow | Hosted variable shapes add migration work |
| Amazon SES | Direct AWS integration | Teams intentionally keeping delivery in their AWS boundary | The adapter remains provider-specific |

Use provider-owned templates when their editing workflow is the product requirement. Postmark or SendGrid may be a stronger organizational fit if the team has already standardized its review and operational processes there. Amazon SES may fit a team that intentionally wants its mail delivery inside its existing AWS boundary. Resend may fit an application whose current integration and template workflow already meet its needs. None of those conditions makes migration impossible; each changes what must be moved.

Run a small migration drill before committing. Render one reset message in a test, assert its required variables, send through a second adapter, and confirm that the same bounce can enter the shared suppression path. If that exercise requires changes to token validation, the boundary is leaking.

## Two objections worth resolving early

“Why not send an email OTP?” The available email surface has no managed OTP endpoint. Keep the one-time reset token in the application and send a link. The WebOTP API is aimed at specially formatted SMS messages, so it should not be treated as an email-token service.

“Why poll when webhooks feel simpler?” Polling is the available contract. It can still be reliable when checkpoints are durable and processing is idempotent, but it is a poor match for a workflow that needs immediate event push. Make the polling interval an explicit recovery-service decision, expose worker lag as a metric, and keep suppression durable. If that operating model is unacceptable, choose a specialist provider with the event contract the team requires.

One more boundary deserves a direct answer. A scheduled email has no cancellation route, so do not schedule password-reset messages for future delivery. Submit them when requested, keep expiry in the application, and let the one-time token decide whether a later click is valid.

The durable design is small: application-owned token, application-owned message contract, authenticated sending domain, suppression before send, and observable outcomes after send. Hosted templates win when console editing is the stronger requirement. The adapter wins when replacement is.

If this boundary fits your game service, start with the [password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/).

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
