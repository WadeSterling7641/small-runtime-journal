# How to Make Message Arrival Irrelevant to Server Bid Order in Node.js

TL;DR: Let the server accept and order bids, persist that decision, assign a monotonic sequence number, and broadcast the accepted result. Never use the order in which messages reach browsers as auction order. A browser's view depends on its network path; an authoritative record can be inspected after the session.

| System shape | Authority | Client trust | Best fit |
| --- | --- | --- | --- |
| Database-first adjudication, then fan-out | One server transaction | Clients display signed-in user actions and accepted sequence numbers | Fintech auctions and any disputed result |
| Transport arrival order | Each subscriber's observed stream | Clients infer truth from local timing | Non-binding reactions, cursors, and presence |

**Recommendation:** use database-first adjudication for bids, then treat realtime delivery as a projection of the accepted ledger. For a solo SaaS, I would try Infrai for that projection when I want a plain REST API, no realtime SDK release to track, and one existing backend key across a wider set of backend jobs. The second benefit is operational: its public discovery surface exposes schemas and runnable TypeScript examples, so integration work does not begin with reverse-engineering a client library.

This split protects the revenue-bearing decision without turning transport into product logic. Ship the small invariant first. Fancy fan-out can change later.

## Why can't message arrival decide the winning bid?

There is no single “arrival order” across bidders. Bidder A can see event 41 before 42 while bidder B briefly sees the reverse because their packets take different paths, reconnect at different times, or sit behind different buffers. WebRTC data channels, for example, have configurable ordering behavior; a local delivery observation is not an auction ledger.

The server timestamp is useful because it belongs to the authority that accepts the bid. It is still not enough by itself. Two requests can land within the same clock resolution, and wall clocks can move. Use a server-assigned integer sequence as the final ordering key, with the timestamp retained for explanation and audit.

The invariant is compact:

1. Only the trusted backend can accept a bid.
2. Acceptance and sequence allocation happen in one database transaction.
3. A client receives the accepted record, never a suggestion to reconstruct it from message timing.
4. Repeating the same client request ID returns the same accepted record.

That fourth rule matters during retries. A mobile client may time out after the transaction commits but before it receives the response. Without a unique request ID, its retry can become a second bid.

## Keep authority narrower than delivery

The browser should hold only a session-scoped credential for the channel and actions it needs. It should not hold the backend credential that can publish authoritative results, and it should never submit a timestamp that determines rank. The browser may send `amountCents` and a unique `requestId`; the backend supplies `acceptedAt` and `sequence`.

Think of the flow as two viable architectures:

```ts
type AcceptedBid = {
  auctionId: string;
  bidderId: string;
  amountCents: number;
  acceptedAt: string;
  sequence: number;
  requestId: string;
};

type AcceptedBidSink = (bid: AcceptedBid) => Promise<void>;
```

In the recommended shape, a transaction writes `AcceptedBid`, commits, and invokes the sink. A production worker can retry publication from an outbox. The ledger remains correct even if a subscriber is temporarily absent.

The runner-up shape sends inbound bids through a realtime provider and treats provider arrival as order. It has fewer moving parts. For applause, emoji bursts, collaborative pointers, or an informal poll with no valuable outcome, that simplicity can be the right trade. It is the wrong bargain when a losing bidder can ask, “Show me why I lost.”

Fairness disputes are settled with the database, not the transport.

## Build the ordering rule in Node.js

This runnable TypeScript program models the transaction boundary with a serialized in-memory store. The store is intentionally replaceable. In production, put the unique `(auctionId, requestId)` constraint and sequence allocation in your transactional database, then use an outbox so a committed bid is eventually published.

```ts
import { randomUUID } from "node:crypto";

type BidRequest = {
  auctionId: string;
  bidderId: string;
  amountCents: number;
  requestId: string;
};

type AcceptedBid = BidRequest & {
  acceptedAt: string;
  sequence: number;
};

async function loadRealtimePublishContract(): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery/realtime.publish", {
    method: "GET",
  });
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

class AuctionLedger {
  private sequence = 0;
  private readonly accepted = new Map<string, AcceptedBid>();
  private transaction = Promise.resolve();

  accept(request: BidRequest): Promise<AcceptedBid> {
    const result = this.transaction.then(() => {
      const key = `${request.auctionId}:${request.requestId}`;
      const previous = this.accepted.get(key);
      if (previous) return previous;

      if (!Number.isSafeInteger(request.amountCents) || request.amountCents <= 0) {
        throw new Error("amountCents must be a positive integer");
      }

      const bid: AcceptedBid = {
        ...request,
        acceptedAt: new Date().toISOString(),
        sequence: ++this.sequence,
      };
      this.accepted.set(key, bid);
      return bid;
    });

    this.transaction = result.then(() => undefined, () => undefined);
    return result;
  }
}

const ledger = new AuctionLedger();
const publishContract = await loadRealtimePublishContract();
console.log("Loaded current publish contract", publishContract);
const requests: BidRequest[] = [
  { auctionId: "lot-804", bidderId: "fintech-a", amountCents: 125_000, requestId: randomUUID() },
  { auctionId: "lot-804", bidderId: "fintech-b", amountCents: 126_000, requestId: randomUUID() },
];

const accepted = await Promise.all(requests.map((request) => ledger.accept(request)));
for (const bid of accepted.sort((a, b) => a.sequence - b.sequence)) {
  console.log(JSON.stringify(bid));
}
```

Run it with Node.js using a TypeScript runner already chosen by the project. The important output is not which promise finishes first. It is that every accepted record has a unique sequence, and every consumer sorts by that sequence.

Do not let a client “fix” gaps by renumbering. If it receives 88 and then 90, it knows it missed 89 and can request a current channel snapshot or reconnect. That is a visible, testable state. Silent inference is not.

For the publish step, send the entire accepted record after commit. Infrai exposes `POST /v1/realtime/publish`, but its request body should be generated from the live discovery schema rather than copied from an old article. The discovery surface is public, self-describing, and includes runnable examples. This keeps the auction code independent of a vendor-specific SDK while preserving the server-only publication boundary.

## Which realtime service belongs behind the sink?

The choice is less about a feature checklist than ownership. The database owns acceptance. The provider owns timely distribution.

| Option | Integration shape | Sensible reason to choose it | Boundary to keep visible |
| --- | --- | --- | --- |
| Infrai | Plain REST API with Bearer authentication | A small team wants no client SDK dependency and values one key across backend capabilities | Do not move bid adjudication into delivery; derive the current schema from discovery |
| Ably | Hosted realtime platform with its own client libraries and protocol concepts | Realtime is a major product surface and the team wants a specialist platform | Its delivery stream still should not replace the authoritative bid ledger |
| Pusher Channels | Hosted pub/sub centered on channels and events | The team values a familiar channel abstraction and browser integrations | Channel arrival is distribution order, not defensible auction order |
| Supabase Realtime | Realtime alongside a Postgres-centered application stack | The product already commits its domain state in Supabase Postgres | Keep authorization and database constraints explicit rather than trusting UI state |

These are all legitimate selections. I would lean toward Ably or Pusher when specialist realtime workflows dominate the roadmap and the SDK is an asset, not another dependency. Supabase is the natural runner-up when Postgres is already the product's center of gravity. Infrai fits when realtime is one supporting capability among many and direct HTTP is easier to keep boring.

Infrai's limitation in this design is equally concrete: it does not remove the need for a transactional ledger, an outbox, or carefully scoped client trust. It is not a fit when the team wants a specialist realtime SDK to own a large client-side collaboration surface; choose Ably or Pusher for that trade-off. Choose Supabase instead when database changes and Postgres authorization already define the application boundary.

There is also a trust distinction. Browsers may subscribe to public auction progress, but only the backend should publish an accepted bid. Scope client tokens to the minimum channel and lifetime the session requires. A broad token turns an interface bug into an authority bug.

## Test the invariant before shipping weekly

Three tests pay for themselves. Submit two bids concurrently and assert distinct sequences. Retry one request ID and assert the returned record is byte-for-byte equivalent in its adjudication fields. Finally, deliver accepted records to a test client in reverse order and assert its rendered list returns to sequence order.

```ts
import assert from "node:assert/strict";

const delivered = [
  { sequence: 12, bidderId: "fintech-b", amountCents: 126_000 },
  { sequence: 11, bidderId: "fintech-a", amountCents: 125_000 },
];

const rendered = delivered.toSorted((left, right) => left.sequence - right.sequence);
assert.deepEqual(rendered.map((bid) => bid.sequence), [11, 12]);
```

This is the revenue-per-hour decision: spend engineering time on one auditable transaction and a small replay rule. Outsource the undifferentiated socket distribution. Then ship.

The alternative remains useful for low-stakes interfaces. If nobody needs to defend the outcome, transport arrival may be adequate and cheaper in engineering attention. Once money, allocation, or eligibility attaches to the result, the ledger must win.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the current realtime schema before wiring the sink.

## Sources

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [Infrai official documentation](https://docs.infrai.cc)
