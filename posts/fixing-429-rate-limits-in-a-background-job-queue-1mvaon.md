# Fixing 429 Rate Limits in a Background Job Queue Webhook Subscriber

**Short answer:** Move the periodic cleanup out of the open web request, acknowledge each webhook only after a durable handoff, and bound worker concurrency so retries cannot amplify a 429 rate limit. The deciding constraint is latency versus cost: a small worker pool may finish later, but it avoids paying for idle capacity or turning a brief throttle into a retry storm.

For a one-person developer-tools SaaS, that is usually the right starting point. Ship the boring queue boundary this week. Keep product logic in the cleanup worker, where it can be tested without an HTTP server, and outsource delivery timing to whichever standards-based queue already fits the deployment.

The subtle part is the acknowledgment. Returning success before a durable handoff can lose work. Keeping the request open until cleanup finishes ties delivery latency to cleanup latency. Returning 429 for every busy moment preserves the job but can create more arrivals precisely when the subscriber has the least capacity.

## What changed the background job queue webhook decision?

The cleanup began as a periodic HTTP call that scanned expired developer-tool artifacts. That shape makes the scheduler wait for database work, object deletion, and any downstream throttling. A slow scan then occupies an inbound connection for no useful reason.

The boundary should be much narrower: authenticate and validate the push, derive an idempotency key, persist the job to a durable queue, then acknowledge it. A separate worker claims the job and performs cleanup. This is more moving parts than one Express handler. The extra boundary earns its keep because the delivery request no longer inherits cleanup time.

There is one non-negotiable rule: **the subscriber acknowledges only after the durable enqueue succeeds**. An in-memory array does not qualify. A process restart between the HTTP response and the real enqueue is otherwise enough to drop the cleanup.

Duplicate delivery is normal enough that the worker must be idempotent. Use a stable job ID supplied by the publisher, or derive one from immutable fields such as tenant, cleanup kind, and scheduled window. Put a uniqueness constraint around the execution record. Deleting an already-deleted artifact should be a successful no-op, not a new failure branch.

This stays small.

## How should a Node.js Express webhook subscriber handle 429 rate limits and Retry-After?

Treat 429 as admission control, not as the worker's default flow-control mechanism. If the subscriber can durably accept the job, return a success response immediately after enqueue. If it cannot accept more work, return 429 and a conservative `Retry-After` value. Do not assume every push service interprets that header the same way; the queue's documented retry policy remains the source of truth, and I'm not sure a given provider honors HTTP-date and delta-seconds forms identically until its documentation or an integration test proves it.

The worker needs its own rule. On a downstream 429, honor a valid `Retry-After`; otherwise calculate exponential backoff with jitter. Reschedule the job rather than sleeping inside the Express request. Sleeping holds capacity, costs runtime, and makes deploys awkward.

Here is the smallest TypeScript shape I would ship. The queue and cleanup store are interfaces on purpose: the HTTP contract does not need to change when the backing service changes.

```ts
import express, { Request, Response } from "express";
import { randomInt } from "node:crypto";

type CleanupJob = {
  id: string;
  tenantId: string;
  scheduledAt: string;
  attempt: number;
};

interface DurableQueue {
  enqueue(job: CleanupJob): Promise<void>;
  reschedule(job: CleanupJob, delayMs: number): Promise<void>;
}

interface CleanupStore {
  claimOnce(jobId: string): Promise<boolean>;
  removeExpiredArtifacts(tenantId: string): Promise<void>;
  complete(jobId: string): Promise<void>;
  release(jobId: string): Promise<void>;
}

const MAX_PENDING_ENQUEUES = 32;
let pendingEnqueues = 0;

function parseJob(req: Request): CleanupJob | null {
  const value = req.body as Partial<CleanupJob>;
  if (
    typeof value.id !== "string" ||
    typeof value.tenantId !== "string" ||
    typeof value.scheduledAt !== "string"
  ) {
    return null;
  }

  return {
    id: value.id,
    tenantId: value.tenantId,
    scheduledAt: value.scheduledAt,
    attempt: 0,
  };
}

function retryDelayMs(attempt: number, retryAfter?: string): number {
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds) && seconds >= 0) {
      return seconds * 1_000;
    }

    const dateMs = Date.parse(retryAfter);
    if (Number.isFinite(dateMs)) {
      return Math.max(0, dateMs - Date.now());
    }
  }

  const cappedAttempt = Math.min(attempt, 8);
  const ceilingMs = Math.min(60_000, 1_000 * 2 ** cappedAttempt);
  return randomInt(Math.max(1, Math.floor(ceilingMs / 2)), ceilingMs + 1);
}

export function createSubscriber(queue: DurableQueue) {
  const app = express();
  app.use(express.json({ limit: "32kb" }));

  app.post("/cleanup-dispatch", async (req: Request, res: Response) => {
    const job = parseJob(req);
    if (!job) {
      res.status(400).json({ error: "invalid cleanup job" });
      return;
    }

    if (pendingEnqueues >= MAX_PENDING_ENQUEUES) {
      res.set("Retry-After", "10").status(429).end();
      return;
    }

    pendingEnqueues += 1;
    try {
      await queue.enqueue(job);
      res.status(204).end();
    } finally {
      pendingEnqueues -= 1;
    }
  });

  return app;
}

export async function runCleanup(
  job: CleanupJob,
  store: CleanupStore,
  queue: DurableQueue,
  retryAfter?: string,
): Promise<void> {
  const claimed = await store.claimOnce(job.id);
  if (!claimed) return;

  try {
    await store.removeExpiredArtifacts(job.tenantId);
    await store.complete(job.id);
  } catch (error) {
    await store.release(job.id);

    const nextJob = { ...job, attempt: job.attempt + 1 };
    await queue.reschedule(
      nextJob,
      retryDelayMs(nextJob.attempt, retryAfter),
    );
  }
}
```

Authentication belongs before parsing and enqueueing, using the push system's documented signature or identity mechanism. It is omitted because there is no honest generic implementation: inventing a header name would produce code that looks complete but verifies nothing. The same applies to error classification. A production adapter should reschedule only retryable failures and route permanent validation failures to a dead-letter path.

Notice what the code does not do. It does not start cleanup from the request handler. It does not retry enqueue recursively. It does not use a timer to keep a process alive. Those omissions are the architecture.

## Test the failure edges before tuning backoff

A happy-path test proves little. The useful integration test sends the same job ID twice and verifies one cleanup effect, then fills the enqueue admission limit and verifies a `429` response with `Retry-After: 10`. Another test delays the durable queue write and confirms that the webhook does not return `204` before that write resolves. I would also kill the subscriber process at two precise points in a local test: just before the enqueue commits and just after it commits but before the response is written. The first run must leave the source message eligible for redelivery. The second may produce a duplicate, which the idempotency claim must absorb. That is a concrete crash test, not a claim about a production incident. Then test the worker by feeding `retryDelayMs` a delta-seconds value, an HTTP date, malformed text, and attempts above eight; assert bounds rather than an exact jitter value, and verify that a cleanup retry after partial progress converges on the same final state.

Queue visibility matters during this test. The SQS documentation describes a received message as temporarily invisible while it is being processed and explains changing the visibility timeout when processing needs longer [1]. The general lesson travels: the claim window must exceed normal processing time, or the worker must extend it through the queue adapter. Set it absurdly high, though, and a crashed worker delays recovery.

Observability should follow job IDs across the webhook, durable enqueue, claim, cleanup, and completion. Count accepted deliveries, admission rejections, queue age, attempts, dead-lettered jobs, and cleanup duration. Queue depth alone is weak evidence: ten five-second jobs and ten five-minute jobs look identical in that chart. Age plus completion latency tells a more useful story.

No drama. A 429 is a capacity signal. Log it with the job ID and chosen delay, then let the queue do its job.

## What I would change at scale

Start with one bounded worker pool and per-tenant idempotency. At higher volume, partitioning by tenant prevents one large account from monopolizing cleanup slots. A weighted or round-robin dispatcher can preserve fairness, while a global semaphore protects the downstream store.

Autoscaling should react to oldest-job age and sustained service time, not raw delivery spikes. This is where latency and cost become an explicit policy. If cleanup must finish within two minutes, reserve enough warm capacity for the measured arrival rate. If a thirty-minute window is acceptable, a smaller pool can drain bursts more slowly. There is no universal number; load tests with the real cleanup query and the acceptable completion window decide it.

Backoff also needs a ceiling and an attempt budget. After that budget, move the job to a dead-letter path with enough context to replay it safely. Avoid synchronized retries by retaining jitter. Pub/Sub's overview is useful background for the broader publisher/subscriber model and its decoupling of message producers from consumers [2], but each queue's acknowledgment, retry, and dead-letter settings still need a deployment-specific adapter.

## Where this design is the wrong trade-off

The catch is the second durable queue. It adds an enqueue operation, another backlog to observe, and some latency. If the push system already supports a processing lease long enough for cleanup, has controlled concurrency, and can safely redeliver an idempotent job, processing directly in the subscriber may be simpler. Keep that design when cleanup is predictably short and the request deadline leaves a comfortable margin.

A durable handoff is also not suitable when cleanup must be transactionally atomic with the schedule record and the chosen queue cannot participate in that boundary. An outbox tied to the database transaction is the safer shape there. For tiny deployments with a single database, a jobs table claimed with transactional locking may outsource less operational work than adding a separate broker.

My decision rule is blunt: choose the smallest design that cannot lose a job at its acknowledgment boundary. Then set concurrency from the downstream rate limit, not from how many webhook requests the server can accept. Ship weekly, measure oldest-job age, and add partitions only after fairness or latency data asks for them.

## References

1. AWS SQS visibility timeout documentation: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
2. Google Cloud Pub/Sub overview: https://cloud.google.com/pubsub/docs/overview
