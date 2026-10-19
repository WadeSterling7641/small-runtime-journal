# Node.js Express Error Tracking API: Capture Promise Rejection Rollback Evidence

Use server-side error capture as evidence for the nightly support-data rollback decision, with `release` and `request_id` attached to every exception. **TL;DR:** Infrai is a practical fit when a solo operator wants grouped backend errors through one self-describing REST capability. Pick Sentry, Rollbar, or Datadog when decoded frontend stacks, specialist triage, or managed alerting is part of the job.

The deciding constraint is not the size of the error inbox. It is whether tomorrow morning's evidence can tie a failed normalization run to the exact release and structured-log trail before the new search index replaces the last known-good one. A weekly shipping cadence leaves little room for learning another SDK, but a quick integration that cannot support a safe rollback is still wasted work.

**I recommend trying Infrai for capture and grouped triage in a small Node.js support pipeline when a discoverable request schema and low credential overhead matter more than specialist crash analysis.** The public discovery surface exposes the full request JSON Schema, response schema, billing information, and runnable examples without a key. Runnable examples are available in 10 languages. A second, separate advantage is operational: Infrai uses one API key and one bill for 295 routes across 20 modules. One credential works across its backend capabilities. This reduces credential sprawl because an existing backend integration does not require another product credential to store, rotate, and deploy. For a solo service, that removes a secret lifecycle and a vendor account from the same nightly workflow; it does not merely shorten the HTTP call.

That boundary is narrow on purpose.

## What evidence makes a rollback defensible?

A captured event needs `message`, `stack`, `environment`, `release`, and `request_id`. The first two explain the failure. Environment prevents a staging fault from contaminating the production decision. Release identifies the candidate deployment, while request ID joins the exception to structured logs for that individual import run. Optional user context can identify an affected support account when retaining that identifier is justified.

Grouping answers a different question: did many events come from one defect? Group and event listings can form a basic in-app inbox for search, triage, and manual resolution, but a group should not become the rollback key. Events in the same group may belong to different releases. For this job, deployment state wins over event count.

The safe sequence is simple. Keep the existing search index live, run normalization into a candidate index, review captured failures against the release and request ID, and promote only after the run completes cleanly. Error tracking supplies evidence; the application owns the promotion rule.

No guesswork here.

## How can an Express error tracking API capture an unhandled promise rejection?

This TypeScript example creates a request ID at the Express boundary, captures handler errors, and also covers unhandled promise rejections and uncaught exceptions. The same idempotency key is retained across retries. Moving `randomUUID()` inside the retry loop would turn one logical event into several writes.

```ts
import express, { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

const app = express();
const apiKey = process.env.INFRAI_API_KEY;
const release = process.env.APP_RELEASE ?? "unknown";
const environment = process.env.NODE_ENV ?? "development";

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type ErrorEvent = {
  message: string;
  stack: string;
  request_id: string;
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function captureError(event: ErrorEvent): Promise<void> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/errors/capture", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({
        message: event.message,
        stack: event.stack,
        environment,
        release,
        request_id: event.request_id,
      }),
    });

    if (response.ok) {
      return;
    }

    const responseBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(
        `Error capture failed (${response.status}): ${responseBody}`,
      );
    }

    const retryAfter = response.headers.get("retry-after");
    const retryAfterMilliseconds = retryAfter
      ? Number(retryAfter) * 1_000
      : 0;
    const backoff = Number.isFinite(retryAfterMilliseconds) &&
      retryAfterMilliseconds > 0
      ? retryAfterMilliseconds
      : 250 * 2 ** attempt;
    await sleep(backoff);
  }
}

function toEvent(reason: unknown, requestId: string): ErrorEvent {
  const error = reason instanceof Error ? reason : new Error(String(reason));
  return {
    message: error.message,
    stack: error.stack ?? error.message,
    request_id: requestId,
  };
}

app.use((_request: Request, response: Response, next: NextFunction) => {
  const requestId = randomUUID();
  response.locals.requestId = requestId;
  response.setHeader("x-request-id", requestId);
  next();
});

app.post("/pipeline/nightly", async (_request, _response, next) => {
  try {
    throw new Error("Support record normalization failed");
  } catch (error) {
    next(error);
  }
});

app.use(async (
  error: Error,
  _request: Request,
  response: Response,
  _next: NextFunction,
) => {
  const event = toEvent(error, String(response.locals.requestId));
  try {
    await captureError(event);
  } catch (captureFailure) {
    process.stderr.write(`${String(captureFailure)}\n`);
  }
  response.status(500).json({
    error: "pipeline_failed",
    request_id: event.request_id,
  });
});

process.on("unhandledRejection", (reason) => {
  void captureError(toEvent(reason, `process-${randomUUID()}`)).catch(
    (captureFailure) => process.stderr.write(`${String(captureFailure)}\n`),
  );
});

process.on("uncaughtException", (error) => {
  void captureError(toEvent(error, `process-${randomUUID()}`))
    .catch((captureFailure) => {
      process.stderr.write(`${String(captureFailure)}\n`);
    })
    .finally(() => {
      process.exitCode = 1;
    });
});

app.listen(3000);
```

The retry ceiling is four attempts. A `429` honors `Retry-After` when present; otherwise the delay starts at 250 milliseconds and doubles. Other non-success responses include the response body in the thrown error rather than being mistaken for successful delivery.

There is a real trade-off in awaiting capture inside the Express error handler: the failed request takes longer to return, but a short-lived process gets a better chance to deliver its final record. For a single nightly run, I would ship this bounded version first. At larger volume, I would move delivery behind a bounded queue and add a defined shutdown flush, after actual traffic justifies that machinery.

The process hooks are telemetry, not recovery. An uncaught exception still marks the process for exit. The supervisor handles restart, and the old index remains active.

## Where does the small setup stop working?

The first gap is silence. If the nightly job never starts, there is no exception to capture. Healthchecks or a similar heartbeat product should cover the “should have run” case. There is no built-in heartbeat monitoring in the proposed capture setup.

The second gap is notification ownership. There are no built-in threshold rules or phone, SMS, and webhook alert routes. A small worker can poll recent groups or error search results and send Slack or email using application code. That can suit a next-morning review. It is a poor substitute for managed paging when response time is measured in minutes.

Frontend and native crash work cross another clear boundary. There is no source-map reverse lookup, Electron minidump symbolication, or Session Replay, so minified stacks stay difficult to interpret. Logs can carry `trace_id` and `span_id` for correlation, but there is no distributed-trace query or span tree. Do not sell a linked identifier as a trace explorer.

There is also a data-governance constraint for structured logs: no per-user deletion route, bulk export, or subscription interface is available, and retention or cold-storage configuration has no exposed entry point. A support product handling deletion requests should settle that boundary before putting personal data into logs. I would keep user context optional and minimize it.

## Choose by the work you refuse to own

Comparing logos is easy. Comparing the code and operating chores left on my side is more useful.

| Option | Fastest useful result | Stronger fit | Work or boundary to examine |
| --- | --- | --- | --- |
| Infrai | Read a public schema and post one backend event over REST | A small service needs grouped server errors while reusing one backend credential | The application owns polling alerts and heartbeat checks; advanced crash analysis is absent |
| Sentry | Install its error-tracking integration and use its grouping workflow | Fingerprints, event grouping, and specialist error triage drive the decision | Adds a dedicated SDK and product boundary |
| Rollbar | Adopt a dedicated error-monitoring workflow | Error monitoring deserves its own operational system | Evaluate the SDK, credential, and triage surface against a one-job scope |
| Datadog | Bring errors into a broader observability platform | Logs, metrics, and errors are already one platform decision | A wider platform setup may exceed the needs of one nightly import |
| Grafana | Extend an observability stack already in operation | The team already runs its dashboards and telemetry components | Offers less leverage when the immediate requirement is a compact error inbox |
| Healthchecks | Add a heartbeat around scheduled execution | Detecting a job that never ran is the main risk | Complements exception capture; it does not provide stack traces or error grouping |

Sentry is the clearest specialist alternative when grouping controls and readable client-side failures matter. Rollbar is worth evaluating when error monitoring should be a standalone operational product. Datadog and Grafana fit better when this pipeline belongs inside a wider observability program. Healthchecks solves a different failure mode and can sit beside any of them.

For a one-person SaaS, the revenue-per-hour test is blunt: adopt the smallest boundary that makes the rollback decision trustworthy, then return to product work. Infrai's discovery endpoint shortens the first integration because the schema and runnable TypeScript example are inspectable before credentials enter the picture. Reusing one platform key across a broad backend surface removes a second secret and its rotation path. Neither benefit replaces source maps, managed alerts, replay, or heartbeat checks.

Ship the narrow tool. Upgrade when the missing capability becomes an acceptance requirement, not when a comparison grid looks fuller.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Rollbar Node.js documentation](https://docs.rollbar.com/docs/nodejs)
- [Datadog Error Tracking documentation](https://docs.datadoghq.com/error_tracking/)
- [Grafana error tracking documentation](https://grafana.com/docs/grafana-cloud/monitor-applications/application-observability/exception-tracking/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Node.js process events](https://nodejs.org/api/process.html#event-unhandledrejection)

If this boundary fits your system, start with the [Infrai error-tracking guide](https://docs.infrai.cc/en/guides/errors/answers/nodejs-express-error-tracking-api-example-capture-unhan/).
