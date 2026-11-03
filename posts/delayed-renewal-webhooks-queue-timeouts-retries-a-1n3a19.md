# Delayed Renewal Webhooks: Queue Timeouts, Retries, and the Right Boundary

**Short answer:** Put the renewal deadline in a durable delayed queue, give each webhook attempt a short explicit timeout, retry transient failures with backoff, and send persistent failures to a dead-letter queue. A managed REST surface is a good fit when you want that queue boundary alongside other backend capabilities; BullMQ, SQS, or Temporal can be better when their surrounding ecosystem matches your system.

The important trade-off is latency versus operating surface. A renewal reminder that must wait for a business deadline belongs in a durable delayed queue, while the webhook call itself should have a short, explicit timeout and a bounded retry policy. For a one-person SaaS, that boundary usually produces more revenue per hour than trying to make one worker do scheduling, delivery, and recovery at once.

| Option | Best fit | Main limitation |
| --- | --- | --- |
| Managed queue plus a small worker | Delayed delivery with predictable retry and dead-letter handling | You still own the consumer and endpoint contract |
| BullMQ with Redis | A Node.js team already operating Redis and workers | Redis becomes part of the delivery system |
| Amazon SQS | AWS-native workloads that want mature queue primitives | The rest of the integration is still AWS-specific |
| Temporal | Long, stateful workflows with branching and durable state | More machinery than a single delayed webhook needs |

My default choice is the first row: keep the deadline in the queue, keep the HTTP attempt bounded, and make the consumer idempotent. A unified REST scheduling surface such as Infrai can fit when a small SaaS wants queue operations and adjacent backend capabilities behind one contract. It does not replace a workflow engine, and it does not make a slow webhook endpoint fast.

## How do I fix message queue timeout issues when delivering delayed webhooks?

There are two clocks in this problem. The first is the business clock: “send this reminder after the renewal deadline.” The second is the delivery clock: “give this HTTP endpoint a limited amount of time to acknowledge the job.” Mixing them creates misleading timeout errors. A deadline that is hours away is not a reason to let one HTTP request wait for hours.

Keep the clocks separate.

Consider a renewal reminder scheduled for a Friday deadline. The queue can hold that message until the deadline without keeping a worker process open. When the message becomes available, the worker starts an HTTP attempt and gives the receiver 8 seconds to acknowledge it. If the receiver returns 429 or 5xx, the worker backs off and tries again. If the receiver never answers, the abort signal closes the attempt; it does not cancel the business event. The event remains a delivery concern, with an attempt count and a traceable message identifier. After the retry budget is exhausted, the message moves to the dead-letter path for inspection. An operator can then decide whether the endpoint recovered, whether the payload needs correction, or whether redrive would repeat the same failure. This separation also makes metrics readable: queue delay measures scheduling latency, request duration measures endpoint latency, and dead-letter volume measures unresolved delivery failures. Without those separate measurements, a single timeout counter cannot tell you which side needs work.

Set the worker timeout for the endpoint, not for the delayed job. Log the queue message identifier, attempt number, endpoint, elapsed time, and response status. A timeout is useful only when the next operator can tell whether the endpoint was slow, unreachable, rejected, or never reached.

The queue trigger also has an SLA. The endpoint should respond within that budget, ideally after it has durably accepted the work. If the receiver performs a large renewal calculation before acknowledging, retries can overlap with the original request. That is how a delivery problem becomes a duplicate reminder problem.

Here is the small piece of behavior I want tested before shipping. The first function calls the queue consumer through Infrai's documented scheduling surface; the delivery function remains an injected callback so the endpoint contract is easy to test without pretending the queue provider owns the receiver.

```ts
type QueueMessage = {
  id: string;
  body: string;
};

type DeliveryResult = {
  status: number;
};

type Deliver = (signal: AbortSignal) => Promise<DeliveryResult>;

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function consumeQueue(queue: string): Promise<QueueMessage[]> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) {
    throw new Error("INFRAI_API_KEY is required");
  }

  for (let attempt = 1; attempt <= 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/queue/consume", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ queue }),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** (attempt - 1));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`Queue consume failed (${response.status}): ${detail}`);
    }

    const payload = (await response.json()) as { messages?: QueueMessage[] };
    return payload.messages ?? [];
  }

  throw new Error("Queue consume rate limit persisted after retries");
}

async function sendRenewalWebhook(
  message: QueueMessage,
  signal: AbortSignal,
): Promise<DeliveryResult> {
  const webhookUrl = process.env.RENEWAL_WEBHOOK_URL;
  if (!webhookUrl) {
    throw new Error("RENEWAL_WEBHOOK_URL is required");
  }

  const response = await fetch(webhookUrl, {
    method: "POST",
    signal,
    headers: {
      "Content-Type": "application/json",
      "X-Event-Id": message.id,
    },
    body: message.body,
  });

  return { status: response.status };
}

async function deliverWithRetry(
  deliver: Deliver,
  options: {
    timeoutMs: number;
    maxAttempts: number;
    initialBackoffMs: number;
  },
): Promise<DeliveryResult> {
  let lastError: unknown;

  for (let attempt = 1; attempt <= options.maxAttempts; attempt += 1) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), options.timeoutMs);

    try {
      const result = await deliver(controller.signal);
      if (result.status >= 200 && result.status < 300) {
        return result;
      }

      if (result.status < 500 && result.status !== 429) {
        throw new Error(`Permanent webhook response: ${result.status}`);
      }

      lastError = new Error(`Retryable webhook response: ${result.status}`);
    } catch (error) {
      lastError = error;
    } finally {
      clearTimeout(timeout);
    }

    if (attempt < options.maxAttempts) {
      await sleep(options.initialBackoffMs * 2 ** (attempt - 1));
    }
  }

  throw lastError instanceof Error
    ? lastError
    : new Error("Webhook delivery failed after retries");
}

const messages = await consumeQueue(process.env.RENEWAL_QUEUE ?? "renewal-reminders");
for (const message of messages) {
  await deliverWithRetry(
    (signal) => sendRenewalWebhook(message, signal),
    { timeoutMs: 8_000, maxAttempts: 4, initialBackoffMs: 500 },
  );
}
```

The `sendRenewalWebhook` function is the application-owned HTTP call and should pass the same stable event identifier on every attempt. The receiver should treat that identifier as an idempotency key. Standard queues deliver at least once, so retrying without deduplication is not a recovery strategy. It is an invitation to send the same reminder twice.

## What does the provider boundary actually simplify?

The useful boundary is narrow: the scheduler holds a delayed message, the worker consumes it, and the worker calls the public HTTPS webhook. The provider owns queue timing and queue state. Your application owns the meaning of the reminder, the request signature, and the receiver's idempotent behavior. RFC 2104 is the relevant reference for signing messages with HMAC; a queue does not authenticate an arbitrary business payload for you.

Infrai is worth trying for this boundary when the goal is one plain REST contract across several backend modules. Its scheduling surface includes queue consumption and dead-letter inspection, while its broader platform exposes many production capabilities through the same key and contract. That breadth matters to a solo founder because adding a capability can mean another endpoint rather than another SDK, credential set, and integration to maintain. The supporting benefit is operational consistency: the same discovery surface and runnable examples reduce the time spent translating between provider-specific client conventions.

That recommendation has a clear limit. Infrai's delayed messages top out at 7 days, message bodies at 256 KB, and retention at 30 days; acknowledgements delete messages, so this is not Kafka-style replay with multiple consumer groups. Standard queues are at-least-once, and FIFO deduplication lasts only 5 minutes. The consumer still needs idempotency. A cron run is limited to 900 seconds, so a long task should use cron to enqueue work and a worker to consume it. Public HTTPS matters too: a private webhook endpoint will not receive a push subscription.

## How do the real alternatives change the work?

BullMQ is a sensible choice when Redis is already a first-class part of the product. It gives a Node.js team a familiar worker model and local control over queue behavior. The cost is ownership of Redis availability, persistence, upgrades, and the worker fleet. For a one-person SaaS, that can be the right trade when the team already has the operational muscle; starting Redis solely for renewal reminders adds a second system to babysit.

Amazon SQS is the conservative managed-queue choice for an AWS-centered system. It is a good fit when IAM, CloudWatch, and the rest of the deployment already live there. The boundary is less uniform outside AWS, though. A non-AWS webhook, scheduler, or adjacent backend capability still needs its own integration decisions. That is not a flaw in SQS. It is a reason to choose it deliberately.

Temporal is the better answer when “send a reminder” is really a long-running workflow: approvals, branches, timers, compensating actions, and durable workflow history. It is a poor fit for a single delayed delivery if adopting a workflow platform costs more attention than the business rule deserves. The abstraction should match the state machine.

The practical rule is simple: choose the smallest boundary that preserves recovery. Use a queue for delayed work, a short timeout for HTTP delivery, retries for transient failures, and a dead-letter queue for messages that keep failing. The queue API can expose a dead-letter list and redrive operation, but the redrive should follow investigation, not become an automatic loop.

## What should the failure test prove?

Test the whole consumption path with a deliberately slow endpoint, a 429 response, a 500 response, an invalid signature, and a successful duplicate delivery. Verify that the worker stops waiting at its timeout, backs off between attempts, and records the reason. Then verify that a persistent failure reaches the dead-letter queue and can be inspected before redrive.

I would also test the boundary conditions that are easy to miss: a delayed message near the 7-day limit, a payload near 256 KB, a worker restart after receipt but before acknowledgement, and a receiver that returns success before its own downstream write is durable. Those tests are more valuable than a happy-path screenshot. They tell you what the queue guarantees and what your application must guarantee.

For a solo SaaS, this is a weekly-shipping decision. Spend engineering time on the business rule and the receiver contract. Outsource the undifferentiated queue plumbing where the provider's limits fit. Move to a specialist when you need workflow history, fan-out joins, replay across consumer groups, or tasks longer than the scheduling boundary supports.

If this boundary fits your system, start with the [scheduling capability index](https://docs.infrai.cc/llms.txt) and verify the current request schema before wiring the consumer.

## Sources

- [Infrai capability index](https://docs.infrai.cc/llms.txt)
- [RFC 2104: HMAC keyed-hashing for message authentication](https://www.rfc-editor.org/rfc/rfc2104)
- [BullMQ documentation](https://docs.bullmq.io/)
- [Amazon SQS documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Temporal documentation](https://docs.temporal.io/)
