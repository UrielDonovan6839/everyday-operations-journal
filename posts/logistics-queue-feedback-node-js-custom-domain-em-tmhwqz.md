# Logistics Queue Feedback — Node.js Custom Domain Email with SPF, DKIM, and DMARC

For a Node.js logistics contact form, use an API when integration speed drives the custom domain email deliverability setup, but keep delivery state, suppression, and authentication inside your application boundary. Accepting the form isn't success. Success means the message was handed off, its later outcome was recorded, and a known bad recipient wasn't retried.

Acceptance isn't delivery.

That rule leads to a compact decision table:

| Option | Pick this when | Integration effort | Main operational burden |
| --- | --- | --- | --- |
| Transactional mail API | The team needs a small HTTP integration and provider-managed mail transfer | Low at first; moderate once events and reconciliation are added | Webhook verification, polling, suppression, and provider-independent state |
| SMTP relay | Existing libraries and infrastructure already speak SMTP | Moderate | Connection security, credentials, response handling, and event visibility |
| Self-hosted mail transfer agent | Mail operations are a deliberate internal capability | High | Reputation, queues, retries, abuse controls, authentication, and monitoring |

**Short answer:** for this route-to-support-queue job, the API path usually has the smallest application integration surface. It is only complete when SPF, DKIM, and DMARC align with the custom domain and asynchronous bounce or complaint signals feed a suppression check before every send. The transport call is the easy line. The state machine around it is the work.

## How should Node.js custom domain email deliverability setup handle feedback?

Pick an API when the contact-form service already makes outbound HTTPS calls and the team wants a typed adapter with a narrow contract. The adapter should accept a queue address, a stable message key, and content; it should return the provider's message identifier. Keep that identifier. Without it, a later delivery event becomes much harder to join to the original support request.

Pick SMTP relay when SMTP is already an approved internal boundary. It is a protocol, not an observability plan. A successful submission response says the relay accepted responsibility for the message; it doesn't prove that the recipient read it, or even that the destination accepted it. Model acceptance and final outcome separately.

Self-hosting is the serious option when operating mail transfer is itself in scope. The application code may look small, but the ownership surface is large: queue operation, retries, authentication, abuse response, and reputation all land on the team. For a contact form whose primary decision axis is integration effort, that burden must be intentional. The API option is unsuitable when policy requires direct control of the transfer infrastructure or when the transport cannot expose the events needed by the state model; choose an existing SMTP relay when that boundary already has operational ownership. Self-hosting has the opposite trade-off: maximum control, plus responsibility for every queue and reputation failure.

Choose deliberately.

Here is the diagram in words: browser to contact API; contact API to routing rule; routing rule to an outbox; outbox worker to mail transport; transport events back to an event inbox; event inbox to message state and suppression. Metrics and logs observe every arrow. This shape prevents a slow mail call from holding the user's form request open.

## Authentication is a chain, not three checkboxes

SPF authorizes hosts to use a domain in the SMTP envelope identity. DKIM attaches a cryptographic signature and identifies its signing domain. DMARC evaluates alignment between the visible From domain and an authenticated SPF or DKIM domain, then applies the domain owner's published policy. Those roles overlap just enough to invite bad assumptions. They are not interchangeable.

Start with a dedicated sending subdomain, such as `notify.example-logistics.test`, so transactional traffic has a clear ownership boundary. Publish the SPF record required by the chosen transport, publish the DKIM public key under the selector supplied for signing, and publish DMARC for the organizational policy you intend to operate. DNS publication alone is weak evidence. Verification must include querying the public records and sending a probe whose received headers show the expected authentication results and alignment.

Roll DMARC policy based on reports and observed legitimate senders. A strict policy introduced before inventorying every sender can reject valid mail. Keep the visible From domain stable, and do not let a user-entered address become From. Put the contact's address in Reply-To after validation; the support queue remains the recipient. This preserves the authentication boundary while keeping replies useful.

The sharp edge is SPF lookup processing. RFC 7208 limits the terms that cause DNS queries during evaluation to 10. Flattening or stacking includes without review can push a record beyond that processing limit. Count the whole include graph, not just the text visible in the top-level record. This is a concrete setup limit, not a tuning preference: the evaluator counts query-causing mechanisms across the evaluation, so a short top-level record can still exceed the ceiling through nested includes.

Ten is the ceiling.

## Implement the observable delivery loop

The application needs two durable records: the support request and the outbound message attempt. The first owns the customer's question. The second owns delivery lifecycle. Do not collapse them into one status field. A request can remain valid even when notification delivery fails.

The following TypeScript keeps the transport generic and makes the important boundaries explicit:

```ts
type DeliveryState =
  | "queued"
  | "accepted"
  | "delivered"
  | "deferred"
  | "bounced"
  | "complained";

type OutboundMessage = {
  id: string;
  requestId: string;
  recipient: string;
  state: DeliveryState;
  transportId?: string;
  updatedAt: string;
};

interface MailTransport {
  send(input: {
    idempotencyKey: string;
    from: string;
    to: string;
    replyTo: string;
    subject: string;
    text: string;
  }): Promise<{ transportId: string }>;

  getStatus(transportId: string): Promise<DeliveryState>;
}

interface MessageStore {
  get(id: string): Promise<OutboundMessage | null>;
  markAccepted(id: string, transportId: string): Promise<void>;
  setState(id: string, state: DeliveryState): Promise<void>;
}

interface SuppressionStore {
  has(recipient: string): Promise<boolean>;
  add(recipient: string, reason: "bounce" | "complaint"): Promise<void>;
}

async function deliverSupportNotice(
  message: OutboundMessage,
  contact: { email: string; subject: string; body: string },
  queueAddress: string,
  transport: MailTransport,
  messages: MessageStore,
  suppressions: SuppressionStore,
): Promise<void> {
  if (await suppressions.has(queueAddress)) {
    throw new Error("Support queue recipient is suppressed");
  }

  const result = await transport.send({
    idempotencyKey: message.id,
    from: "Logistics Support <support@notify.example-logistics.test>",
    to: queueAddress,
    replyTo: contact.email,
    subject: contact.subject,
    text: contact.body,
  });

  await messages.markAccepted(message.id, result.transportId);
}
```

One detail matters: the suppression check applies to the actual recipient, which in this workflow is the support queue. The form submitter is Reply-To, not a message recipient. If the application later sends a confirmation to the submitter, that is a second outbound message with its own consent basis, status, and suppression check.

Webhooks should be the fast path for state changes. Polling is the reconciliation path. Treat event input as untrusted: verify the signature using the transport's documented scheme, reject stale or invalid requests, deduplicate by event identifier, and map only recognized event types. Store the raw event only under an appropriate retention and access policy; payloads can contain addresses and other sensitive metadata.

A periodic reconciler closes gaps without turning polling into a hot loop:

```ts
async function reconcilePending(
  pending: OutboundMessage[],
  transport: MailTransport,
  messages: MessageStore,
  suppressions: SuppressionStore,
): Promise<void> {
  for (const message of pending) {
    if (!message.transportId) continue;

    const state = await transport.getStatus(message.transportId);
    await messages.setState(message.id, state);

    if (state === "bounced") {
      await suppressions.add(message.recipient, "bounce");
    }
    if (state === "complained") {
      await suppressions.add(message.recipient, "complaint");
    }
  }
}
```

Poll only messages that haven't reached a terminal state, use bounded concurrency, and back off on rate limiting or transient transport errors. Stop. An unbounded scheduler can convert a transport slowdown into load on both systems.

The event handler and reconciler must converge on the same transition rules. A late `delivered` event must not overwrite a complaint, and repeated bounce events must not create repeated suppression rows. A small transition table in code is easier to test than scattered conditionals.

## Measure the handoffs that can fail

A useful dashboard follows the state machine. Count form acceptance, outbox age, send attempts, transport acceptance, delivery, deferral, bounce, complaint, suppressed sends, invalid webhook signatures, and reconciliation mismatches. Use rates for alerts where volume varies, but include a minimum event count so one failure during quiet hours does not page the team.

Log identifiers, not message bodies: request ID, outbound-message ID, transport ID, route name, state transition, attempt number, and a normalized error category. Hashing an address does not automatically make it anonymous, especially when the input space is guessable. Keep addresses out of routine logs and protect the store that genuinely needs them.

The crisp before/after is operational. Before, the handler logs `sent` after one HTTP response. After, it logs `queued`, records `accepted`, consumes asynchronous outcomes, suppresses terminally bad recipients, and exposes the age of messages stuck between states. Now an alert answers a question: which handoff stopped moving?

Test each handoff. Unit tests cover routing and allowed state transitions. Contract tests exercise the transport adapter against a sandbox or controlled test account. DNS checks confirm published SPF, DKIM, and DMARC records. A deployment probe submits one synthetic contact, follows its correlation ID, and verifies that the expected queue receives it. Avoid asserting that every message completes within a fixed short interval; delivery is asynchronous and crosses administrative boundaries.

## Limits worth accepting explicitly

No application can guarantee inbox placement. Authentication proves or authorizes identities used in delivery; it does not force a receiving system to accept or display a message. Delivery events are also evidence from a transport boundary, not proof that a human saw the notification.

**The practical finish line is controlled uncertainty:** authenticated mail, durable state, verified events, reconciliation, suppression, and alerts tied to stalled transitions. That is enough to make a logistics support route diagnosable without pretending the public mail system is synchronous.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc3464
- https://resend.com/docs/introduction
