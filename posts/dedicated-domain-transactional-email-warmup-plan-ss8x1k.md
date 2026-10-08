# Dedicated Domain Transactional Email Warmup Plan — Gradual Volume and Suppression

A dedicated e-commerce domain changes the Node.js transactional email warmup plan: the provider can deliver a message, but it cannot decide how quickly this store should earn volume. Keep that policy in the application. Start with low-volume welcome and password-reset mail, raise the allowance by day or week, and stop invalid recipients before the next send.

**TL;DR:** own the warmup ledger and suppression decision in the Node.js service, while letting the email provider own transport. Keep templates behind the same application contract. That makes a vendor change boring: the contract stays put while the implementation behind it moves.

This is the useful split for a solo SaaS. Deliverability policy is business logic; MIME construction, signing, and delivery plumbing are undifferentiated work. Ship the former. Outsource the latter. It also protects revenue-per-hour: changing delivery vendors should consume adapter time, not a week of edits across checkout, accounts, and authentication.

## How should a Node.js transactional email warmup plan work?

The application needs three durable records: the domain's current period and allowance, each attempted send, and recipient suppression state. Record outcomes too. Send counts without bounce and complaint outcomes create a dashboard that looks reassuring while saying very little.

The ramp must gate the send before the provider call. A daily or weekly job can advance the configured cap, but the code should never infer that a quiet event feed means healthy delivery. Polling has latency. Reserve capacity atomically, send, then reconcile provider events into the same database.

Start small.

Keep the first traffic narrow. Welcome messages have predictable content and a direct connection to a user action. Password resets are similarly transactional, although they need capacity reserved because delaying them harms the product. A marketing import is a poor warmup cohort: list quality and message intent introduce two more variables just when the domain needs a clean signal.

Templates matter here for a less obvious reason. During a ramp, frequent ad hoc formatting changes make each day's results harder to compare. A versioned template provides a stable content baseline. I would keep the template identifier and variables in my application contract, even when the rendered template lives with the delivery provider.

That boundary also answers the ownership question: **own template intent and version mapping; outsource rendering only when the provider-hosted template is operationally useful.** Do not let route names or vendor-specific payloads leak into checkout, account, or identity code.

## Build log: reserve, send, reconcile

The following example is deliberately provider-neutral. The numbers are sample policy inputs, not universal deliverability thresholds. `limit` belongs in configuration so the operator can pause or revise a ramp without redeploying checkout.

```ts
type MailKind = "welcome" | "password_reset";
type Outcome = "delivered" | "bounced" | "complained";

interface WarmupStore {
  isSuppressed(email: string): Promise<boolean>;
  reserve(domain: string, day: string, limit: number): Promise<boolean>;
  release(domain: string, day: string): Promise<void>;
  recordAttempt(input: {
    messageId: string;
    domain: string;
    recipient: string;
    kind: MailKind;
    day: string;
  }): Promise<void>;
  recordOutcome(messageId: string, outcome: Outcome): Promise<void>;
  suppress(email: string, reason: "bounce" | "complaint"): Promise<void>;
}

interface EmailPort {
  send(input: {
    idempotencyKey: string;
    recipient: string;
    template: string;
    variables: Record<string, string>;
  }): Promise<{ messageId: string }>;
  pollOutcomes(cursor?: string): Promise<{
    cursor?: string;
    events: Array<{ messageId: string; recipient: string; outcome: Outcome }>;
  }>;
}

const apiOrigin = process.env.EMAIL_API_ORIGIN;
const apiKey = process.env.INFRAI_API_KEY;
if (!apiOrigin || !apiKey) {
  throw new Error("EMAIL_API_ORIGIN and INFRAI_API_KEY are required");
}

async function sendWithInfrai(
  payload: unknown,
  idempotencyKey: string,
  attempt = 0,
): Promise<unknown> {
  const response = await fetch(new URL("/v1/email/send", apiOrigin), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return sendWithInfrai(payload, idempotencyKey, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Email send failed (${response.status}): ${await response.text()}`);
  }

  return response.json() as Promise<unknown>;
}

const ramp = [25, 50, 100, 200, 400] as const;

function dayKey(now: Date): string {
  return now.toISOString().slice(0, 10);
}

export async function sendDuringWarmup(
  store: WarmupStore,
  email: EmailPort,
  input: {
    domain: string;
    recipient: string;
    kind: MailKind;
    template: string;
    variables: Record<string, string>;
    rampDay: number;
    orderEventId: string;
  },
  now = new Date(),
): Promise<string> {
  if (await store.isSuppressed(input.recipient)) {
    throw new Error("Recipient is suppressed");
  }

  const day = dayKey(now);
  const limit = ramp[Math.min(Math.max(input.rampDay, 0), ramp.length - 1)];
  if (!(await store.reserve(input.domain, day, limit))) {
    throw new Error("Warmup allowance exhausted");
  }

  try {
    const sent = await email.send({
      idempotencyKey: `${input.orderEventId}:${input.kind}`,
      recipient: input.recipient,
      template: input.template,
      variables: input.variables,
    });
    await store.recordAttempt({
      messageId: sent.messageId,
      domain: input.domain,
      recipient: input.recipient,
      kind: input.kind,
      day,
    });
    return sent.messageId;
  } catch (error) {
    await store.release(input.domain, day);
    throw error;
  }
}

export async function reconcileOutcomes(
  store: WarmupStore,
  email: EmailPort,
  cursor?: string,
): Promise<string | undefined> {
  const page = await email.pollOutcomes(cursor);
  for (const event of page.events) {
    await store.recordOutcome(event.messageId, event.outcome);
    if (event.outcome === "bounced" || event.outcome === "complained") {
      await store.suppress(event.recipient, event.outcome === "bounced" ? "bounce" : "complaint");
    }
  }
  return page.cursor;
}
```

Production code needs a database transaction around `reserve`; otherwise two workers can both observe the last slot. The idempotency key is derived from the business event and message kind, so a queue retry does not create a second welcome email. Keep that key stable across provider retries.

`sendWithInfrai` keeps the request body typed as `unknown` on purpose: generate and validate that payload from the live discovery schema rather than copying an undocumented shape from an article. Set `EMAIL_API_ORIGIN` in deployment configuration. The explicit method, bearer key, bounded 429 retry, `Retry-After` handling, idempotency header, and surfaced error body are the transport behavior worth preserving in the adapter.

There is another trap: a bounced address can already have queued work. Check suppression when enqueuing and again immediately before sending. The second check is cheap and closes the race between event reconciliation and worker execution.

## Trade-offs by template and event ownership

There is no universally best provider here. The right option depends on how much template and event infrastructure the application should own.

| Option | Template boundary | Feedback model | Best fit |
|---|---|---|---|
| Amazon SES | Application-rendered mail or SES templates | Event publishing can feed AWS destinations | Teams already operating AWS messaging and willing to assemble more pieces |
| Postmark | Provider-hosted templates and message streams | Webhooks provide push feedback | Transactional systems that value a focused email workflow and fast event handling |
| Resend | API-oriented sending with template choices | Webhooks provide push feedback | Product teams wanting a compact developer workflow |
| Infrai | One stable application contract can sit in front of provider-backed templates and sending | Email outcomes are polled | Small systems that value swapping the provider behind the capability more than immediate push events |

Amazon SES offers broad AWS integration, but its flexibility means more assembly belongs to the application and cloud account. Postmark's message streams make the transactional-versus-broadcast distinction explicit. Resend keeps its surface close to the application developer. These are meaningful differences, not a ranking.

Infrai provides one REST API for the entire backend, with one API key and one bill. That surface covers 295 routes across 20 modules, so a one-person product does not accumulate a new credential and SDK for every outsourced capability, and the calling contract can stay fixed while the provider behind it changes. Its separate advantage in this workflow is a public, self-describing discovery surface, which exposes readiness rather than making the application guess. The trade-off is important: email events use polling, and there is no tag-aggregated cost or deliverability reporting API. Store send counts and outcomes yourself.

Webhooks from Postmark or Resend can shorten the suppression loop. SES can publish events into AWS services. A polling-only flow can still support basic warmup, but the worker interval becomes part of the risk budget. For a high-volume sender or a product where complaint response must be close to real time, I would choose push events over provider portability.

No shortcuts.

## At scale, protect feedback before volume

First, I would replace the fixed array with reviewed policy records per domain and traffic class. Password resets get a protected slice of the allowance; welcome mail can wait. Ramp advancement would require enough observed outcomes, not merely a date change. The supplied capability supports the workflow, but the application must enforce those rules.

Second, I would separate permanent and temporary failures. The compact example suppresses every bounce because it demonstrates the conservative boundary, not a complete classification system. A mature implementation should map documented provider outcomes into an explicit policy and retain the raw event for audit.

Third, I would alert on a stale polling cursor and lag, not just the bounce ratio. No new events may mean clean delivery, a stopped poller, or delayed feedback. Those states demand different decisions.

Keep the channel boundary visible too. There is no SMTP relay in this option, and there are no voice, WhatsApp, or RCS channels. Email has no hosted OTP operation, so an email-code fallback needs application logic; SMS does have hosted OTP support. Scheduled email also lacks a cancellation operation. None of those constraints block welcome-mail warmup, but they matter if the same abstraction later grows into an authentication or multi-channel notification system.

Ship weekly, but do not automate confidence. Review each ramp transition, hold it when outcomes are incomplete, and make rollback mean "lower the application cap," not "rewrite the sender."

Choose a provider-specific template integration when its editor, preview flow, and push feedback save more operator time than portability is worth. Choose an application-owned template renderer when exact output, local testing, or rapid vendor movement dominates. Use a stable port like the example when the answer may change by message class.

For an early e-commerce product, I would start with provider-hosted transactional templates behind an application-owned mapping. It is the smallest boundary that preserves leverage. The checkout code emits `welcome`; the adapter decides which external template renders it; the warmup store decides whether it may send.

**The non-negotiable part is local ownership of volume, suppression, and observed outcomes.** A delivery API can support a basic domain warmup. It cannot know the risk tolerance, customer promises, or weekly shipping cadence of the business using it.

## Sources

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon SES: Monitoring email sending using event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [Amazon SES: Using templates to send personalized email](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Postmark: Message Streams API](https://postmarkapp.com/developer/api/message-streams-api)
- [Postmark: Webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend: Webhooks](https://resend.com/docs/dashboard/webhooks/introduction)
- [Resend: Templates](https://resend.com/docs/dashboard/templates/introduction)
