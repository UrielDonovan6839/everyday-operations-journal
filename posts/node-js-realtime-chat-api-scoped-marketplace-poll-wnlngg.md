# Node.js Realtime Chat API: Scoped Marketplace Poll Channels With Durable History

Use one realtime channel per live marketplace poll, issue a narrowly scoped client token, and keep every durable vote and poll transition in your own database. The deciding constraint is client trust: a browser may receive live results, but it must not become the authority for history or vote acceptance.

**Short answer:** backfill the current poll from Postgres when someone joins, then subscribe that client to live updates. Treat the channel as transport. Treat your tables as truth.

That split also keeps the Node.js application understandable. After a reload, the API reconstructs the session from durable rows. During the session, the channel carries the newest accepted state quickly. If the live connection is replaced, history remains intact.

## The before-and-after model

The tempting design is short: browser publishes a vote, every subscriber updates its count, and the team assumes the channel can answer late joiners. It cannot satisfy the stated durability rule. Anything a user should see after reload belongs in tables.

The corrected path is longer, but each component gets one job. This is an explicit trade-off: a little more application plumbing buys a durable authority that is independent of a live connection.

1. The browser sends a vote to the trusted Node.js application.
2. The application validates the poll, voter, choice, and authorization.
3. Postgres records the accepted vote and updates the durable poll state.
4. Only after that write succeeds, the application publishes the resulting state to the poll's channel.
5. A joining browser fetches a database backfill, then subscribes for newer events.

Picture the boundary in words: browser to Node.js to Postgres, then Node.js to channel to browsers. The arrow into Postgres comes before fan-out. That ordering is the useful part.

Database first.

One channel per poll gives a natural scope for presence and tokens. A token for `poll_8f31` should not imply access to `poll_93ac`, and a viewer token should not silently acquire the authority to submit a vote. Keep authorization decisions on the server.

## A copyable Node.js publish path

This TypeScript example commits a vote first and publishes a compact state event second. Infrai fits a team that wants one REST API and one key across backend modules, with no realtime server SDK to install. It is a poor fit when the team wants a provider-specific client library to own most connection behavior or already relies on another platform's database-coupled authorization; test Ably, Pusher Channels, PubNub, or Supabase Realtime in those cases.

The sample deliberately exposes one application route and calls one realtime route. `recordVote` is the database transaction boundary supplied by your repository layer. The event identifier comes from the committed row, making a repeated publish recognizable to consumers.

```ts
import express from "express";

type PollState = {
  eventId: string;
  pollId: string;
  revision: number;
  totals: Record<string, number>;
};

declare function recordVote(input: {
  pollId: string;
  voterId: string;
  choiceId: string;
}): Promise<PollState>;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const realtimeBaseUrl = process.env.REALTIME_BASE_URL;
if (!realtimeBaseUrl) throw new Error("REALTIME_BASE_URL is required");

const app = express();
app.use(express.json());

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function publish(state: PollState): Promise<void> {
  const url = `${realtimeBaseUrl}/v1/realtime/publish`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": state.eventId,
      },
      body: JSON.stringify({
        channel: `poll_${state.pollId}`,
        event: "poll.state.changed",
        data: state,
      }),
    });

    if (response.ok) return;

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Realtime publish failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delayMs);
  }
}

app.post("/polls/:pollId/votes", async (request, response) => {
  try {
    const voterId = String(request.header("x-user-id") ?? "");
    const choiceId = String(request.body.choiceId ?? "");
    if (!voterId || !choiceId) {
      response.status(400).json({ error: "voter and choice are required" });
      return;
    }

    const state = await recordVote({
      pollId: request.params.pollId,
      voterId,
      choiceId,
    });
    await publish(state);
    response.status(202).json(state);
  } catch (error) {
    response.status(500).json({
      error: error instanceof Error ? error.message : "vote failed",
    });
  }
});

app.listen(3000);
```

The retry is bounded to 4 attempts. A `429` honors `Retry-After` when present and otherwise starts at 250 milliseconds and backs off exponentially. The idempotency key is stable because it is attached to the committed event, not generated inside the retry loop. A subtle trap is generating that key inside `publish`: each retry then looks new, defeating the protection precisely when the network is uncertain.

There is still an operational choice here: if publishing fails after the database commit, a production design should persist an outbox record in the same transaction and let a worker publish it. That is a database pattern, not a claim about the realtime provider. It closes the gap between durable acceptance and fan-out without letting the browser retry the vote blindly.

## How should a SaaS team choose a realtime chat API?

Return a snapshot and a revision from the application's backfill endpoint, then start the subscription. While those two actions overlap, buffer incoming events. Apply only events whose revision is newer than the snapshot, in order, and discard duplicate event IDs.

This is a small protocol with a large payoff. A late subscriber sees the database snapshot. A connected subscriber sees fresh changes. A reconnecting subscriber can repeat the same join procedure instead of asking the channel to impersonate a database.

Token scope should match that protocol. Issue a short-lived token for one poll channel and the minimum allowed action. Do not put a general service key in browser code. Presence can say who is connected to the poll, but it should not decide who voted or whether a vote counted; those are durable application facts.

Watch three signals separately: database acceptance failures, publish failures, and client revision lag. Combining them into one “realtime error” counter hides the actual repair path. Alerting should follow ownership: database alerts protect truth, publish alerts protect freshness, and lag alerts protect the audience experience.

## Comparing the provider choices fairly

Ably, Pusher Channels, PubNub, and Supabase Realtime are all real options worth testing alongside a plain REST provider. Do not select among them from a feature-logo grid. Run the same poll join, reconnect, and token-scope test against each candidate's current documentation and client model.

| Option | What to verify for this poll | Best fit for this decision |
| --- | --- | --- |
| Ably | Whether its token and channel rules express one-poll viewer access cleanly | Teams willing to adopt its documented realtime model and libraries |
| Pusher Channels | Whether private-channel authorization maps cleanly to poll membership | Teams that prefer its channel-oriented integration model |
| PubNub | Whether access management matches the required publish and subscribe split | Teams evaluating a broader pub/sub platform surface |
| Supabase Realtime | Whether its Postgres-centered model matches the existing data boundary | Teams already treating Postgres as the application center |
| Plain REST realtime API | Whether scoped token issuance and browser subscription meet the trust boundary | Node.js services that want server-side publishing without another server SDK |

These are evaluation prompts, not claims that the products are interchangeable. Their authorization concepts, connection libraries, and database coupling differ; verify the current behavior in each vendor's primary documentation before committing. The useful bake-off has 4 checks: a late join gets the right backfill, a token cannot cross poll channels, a duplicate event does not double-apply, and a reconnect converges on the database revision. Run those checks with two identities and two poll channels; a happy-path demo with one browser cannot expose a cross-channel token mistake.

Choose based on the hardest boundary. If the team already centers authorization and change delivery on Supabase and Postgres, that coupling may be an advantage. If the application wants a channel-specific managed service and its client model fits, Ably, Pusher Channels, or PubNub may reduce integration work. If avoiding a server SDK matters, a plain REST surface is appealing, but the browser connection and scoped-token flow still need direct validation.

## What does this design deliberately give up?

It gives up the fiction that one realtime primitive should also be the ledger. You operate a database schema, a backfill endpoint, revision handling, and possibly an outbox worker. That is more application code, more monitoring, and another schema to migrate. For an ephemeral typing indicator or a disposable audience reaction, this design may be excessive; a provider-managed transient stream can be enough when nothing must survive reload. For accepted marketplace votes, the extra machinery has a clear purpose because settlement, moderation, or a later dispute may need the durable record.

The return is a clean failure boundary. Poll history remains queryable after a reload, authorization stays in the trusted service, and changing the fan-out provider does not require migrating the record of accepted votes. For a live marketplace session, that trade is usually the right one because correctness outlives the connection.

It also avoids using presence as business state. Presence is transient by nature. “Connected now” can improve the live interface; “voted successfully” must survive disconnection and reload.

Freshness can wait. Truth cannot.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [PubNub access manager](https://www.pubnub.com/docs/general/security/access-control)
- [Supabase Realtime authorization](https://supabase.com/docs/guides/realtime/authorization)
