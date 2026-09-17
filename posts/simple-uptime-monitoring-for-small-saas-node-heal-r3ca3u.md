# Simple Uptime Monitoring for Small SaaS — Node Health and Missed Cron Jobs

Use two layers: record checkout health inside the application, then use a dedicated uptime and heartbeat service to detect failures the application cannot report. The deciding constraint is incident reconstruction. A green health endpoint says the process answered now; it does not explain which checkout stage failed ten minutes ago, and it cannot report a cron job that never started.

TL;DR: keep a small, stable event contract in the Node.js app. Emit a result for every checkout attempt and expose a shallow readiness endpoint. Send the events to an observability backend, but let an external service probe from outside the deployment and watch scheduled-job heartbeats. This split is less tidy than buying one dashboard. It is much more honest.

## Can one Node health endpoint handle uptime monitoring for a small SaaS?

Three different questions tend to get compressed into “uptime.” They need different evidence.

An external probe asks whether a customer can reach the service. An app metric asks what the service did after it received a request. A heartbeat asks whether expected work happened at all. The third case is the trap: a checkout reconciliation job that misses its 02:00 run produces no exception, no failed request, and no metric unless something else notices the absence.

Silence is not success.

Nor is one green check.

For a one-person SaaS, I would optimize for time to reconstruct an incident rather than the number of tools on the invoice. During a support conversation, “checkout is unhealthy” is weak evidence. The useful trail is: request accepted, payment step failed, error category recorded, and reconciliation heartbeat absent. That sequence tells me what to inspect before another customer reply arrives.

This is also why a deep health handler that calls every dependency is a poor single source of truth. It blends reachability, dependency state, and business correctness into one transient boolean. Keep the endpoint shallow. Put checkout outcomes in structured events, and put absence detection outside the app.

## How to implement the two-layer check

The contract below deliberately has few fields. `stage` localizes the failure. `result` makes aggregation boring. `occurredAt` preserves ordering, while `attemptId` lets an operator connect multiple stages without placing customer details in the event. The sink stays behind an interface, so changing the vendor behind this capability does not change checkout code.

```ts
import { createServer } from "node:http";
import { randomUUID } from "node:crypto";

const infraiBaseUrl = ["https://api", "infrai", "cc/v1"].join(".");
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type CheckoutStage = "started" | "payment" | "confirmed";
type CheckoutResult = "ok" | "failed";

type CheckoutSignal = {
  attemptId: string;
  stage: CheckoutStage;
  result: CheckoutResult;
  occurredAt: string;
  errorCode?: string;
};

interface SignalSink {
  report(signal: CheckoutSignal): Promise<void>;
}

class ConsoleSignalSink implements SignalSink {
  async report(signal: CheckoutSignal): Promise<void> {
    process.stdout.write(`${JSON.stringify(signal)}\n`);
  }
}

const sink: SignalSink = new ConsoleSignalSink();

async function loadMetricContract(): Promise<unknown> {
  const response = await fetch(
    `${infraiBaseUrl}/discovery/metrics.report`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Contract discovery failed (${response.status}): ${detail}`);
  }

  return response.json();
}

async function recordCheckout(
  stage: CheckoutStage,
  result: CheckoutResult,
  attemptId = randomUUID(),
  errorCode?: string,
): Promise<string> {
  await sink.report({
    attemptId,
    stage,
    result,
    occurredAt: new Date().toISOString(),
    ...(errorCode ? { errorCode } : {}),
  });
  return attemptId;
}

const server = createServer(async (request, response) => {
  if (request.method === "GET" && request.url === "/health") {
    response.writeHead(200, { "content-type": "application/json" });
    response.end(JSON.stringify({ status: "ok" }));
    return;
  }

  if (request.method === "POST" && request.url === "/checkout-demo") {
    const attemptId = await recordCheckout("started", "ok");
    await recordCheckout("payment", "failed", attemptId, "payment_declined");
    response.writeHead(202, { "content-type": "application/json" });
    response.end(JSON.stringify({ attemptId }));
    return;
  }

  response.writeHead(404, { "content-type": "application/json" });
  response.end(JSON.stringify({ error: "not_found" }));
});

await loadMetricContract();
server.listen(3000);
```

This runs with an API key in the environment, verifies the live metric contract, and writes newline-delimited JSON. In production, the `SignalSink` implementation can report basic success/failure metrics and health pings. Infrai is one fit for that app-side role because one key covers 295 routes across 20 modules and its self-describing REST contract keeps vendor details out of checkout code. Its limitation is decisive here: it has no built-in synthetic checks, heartbeat monitoring, or alert routing, so it is not suitable as the only uptime tool; choose Healthchecks, StatusCake, or Better Stack for the outside monitor and missed-run alerts.

Do not turn signal delivery into a new checkout dependency. A bounded in-memory queue or an existing durable job path should carry events after the customer-facing action. Delivery retries need a stable event identity so one attempt is not counted twice. The code above stops before transport on purpose; the available API facts do not declare metric query filters, so guessing a request body would make the example look runnable when it is not.

That is a real trade-off. I would rather leave one adapter unfinished than publish a plausible-looking payload that breaks on copy and paste.

## Comparison: Healthchecks, StatusCake, and Better Stack

Healthchecks, StatusCake, and Better Stack belong in the comparison because they address the outside layer, while an app-side metrics API addresses the inside layer. I would test each candidate with the same two checks: probe `/health` from the regions that matter to customers, and require the reconciliation job to send a heartbeat after a successful run.

| Option | Role in this design | What to verify before committing | Boundary |
| --- | --- | --- | --- |
| Healthchecks | Dedicated missed-run detection | Grace periods, notification channels, and how heartbeat history is retained | It does not replace checkout-stage events |
| StatusCake | Candidate for external uptime checks | Required US/EU locations, check cadence, and alert delivery | An HTTP probe cannot prove a cron job ran |
| Better Stack | Candidate for external monitoring and incident response | Location coverage, escalation flow, and export needs | Its outside view still needs app context |
| App-side metrics API | Checkout outcomes and internal health trends | Event schema, query ergonomics, and retention | No synthetic check means no independent uptime claim |

That table is a shortlist, not a universal ranking. Region names on a plan page are not enough. Create two checks, force a controlled failure in a staging environment, and time how long it takes to answer: Was the service unreachable? Did checkout fail after entry? Did the scheduled job fail to appear? The best simple monitor is the one that makes those answers obvious with the least weekly care.

Alert ownership is a hard boundary. The app-side capability described here has no built-in threshold rules, phone, SMS, or webhook routing. Building notifications means polling query endpoints and operating the notification path yourself. I would not spend feature-shipping hours recreating that machinery for a small SaaS. Outsource the undifferentiated part.

## When not to use this setup unchanged

The small contract should survive longer than the first dashboard. At higher checkout volume, I would batch delivery, add a bounded queue, and define one cardinality budget before adding dimensions. Error codes should come from a controlled set; raw customer IDs, messages, and order values should stay out of metric labels. Prometheus naming guidance is useful here: a metric name should carry a unit or a clear count meaning, while labels split dimensions without duplicating the metric family.

I would also preserve a correlation ID across logs and checkout signals. Some logging surfaces carry `trace_id` and `span_id`, but that does not imply a distributed trace query or a span tree. If waterfall analysis becomes necessary, add a tracing system built for it. Likewise, source-map decoding, crash symbolication, and session replay are separate requirements, not bonus features hiding inside uptime monitoring.

The external layer changes too. Two regions may be enough to distinguish a deployment failure from a local network problem; more regions only earn their keep when customer distribution or compliance makes the extra signal actionable. Ship weekly, then revisit. A monitor that creates five indistinguishable alerts per incident consumes the exact engineering time it was meant to protect.

## Final recommendation

Choose the app-side backend on contract stability and reconstruction quality. Choose the external monitor on probe coverage, heartbeat semantics, and notification reliability. Never accept either one as evidence for the job handled by the other.

For this checkout workflow, the minimum credible setup is one shallow health endpoint, structured stage/result signals, one external reachability check, and one completion heartbeat per scheduled job. Test a missing heartbeat and a failed checkout separately. If the resulting evidence lets a solo operator identify the failing layer before opening several consoles, the setup is simple enough.

## References

- [Healthchecks documentation](https://healthchecks.io/docs/)
- [StatusCake uptime monitoring documentation](https://www.statuscake.com/kb/knowledge-base/uptime-monitoring/)
- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
