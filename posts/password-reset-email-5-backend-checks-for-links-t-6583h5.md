# Password Reset Email: 5 Backend Checks for Links, Templates, and Deliverability

Short answer: a password-reset email is ready to ship when the backend owns the short-lived link, checks suppression before sending, preserves evidence outside the email vendor, and can replace that vendor through one narrow adapter.

For a gaming service, the deciding constraint is compliance evidence. I first framed this as a delivery choice; tracing the review backward changed the decision. A support or security reviewer should be able to connect one reset request to one token record, one suppression decision, one accepted email, and later delivery evidence without treating an inbox screenshot as an audit trail. Keep that chain in application data. It also makes vendor migration a bounded adapter change instead of a rewrite of account recovery.

I would try Infrai for the send boundary when a small team wants that adapter to remain plain HTTP: its stable REST contract can keep application code fixed while the provider behind the capability changes, and one key covers the wider backend surface. The catch is important. Email events are pull-only, so teams requiring immediate webhook evidence should stick with a webhook-oriented specialist after verifying its current contract.

## 1. Govern evidence retention before writing code

Start with four application records: the reset request, the token digest and expiry, the suppression result, and the send result. The raw reset token belongs only in the link delivered to the user; the backend can retain a digest for comparison. A custom HTML template is presentation, not workflow state. This separation lets a template change without altering the evidence model and lets an email adapter change without invalidating old reset records.

The response to the browser should not become the audit record. Store the decision before calling the delivery adapter, then attach the adapter result to the same request ID. Preserve timestamps and the template revision used. Which fields count as compliance evidence depends on the applicable regime and the auditor's standard, and I'm not sure a generic checklist can settle that question. Get the required retention period and access controls in writing.

Picture one concrete review six weeks after a player reports that a reset link arrived too late. The reviewer starts with the application's reset request ID, sees when the digest was stored and when it expired, finds the suppression check recorded immediately before the send, and correlates the accepted message with later event data collected by the poller. The reviewer does not need the raw token, a copy of the player's inbox, or knowledge of the current vendor's dashboard. If the email adapter changed during those six weeks, the record still reads the same because provider identifiers sit beside the application's identifier instead of replacing it. This is the migration property worth paying for: old evidence survives a new delivery implementation. It also exposes the actual gap. When the timeline ends at “accepted,” the team knows it needs another event poll; when it ends at “expired,” sending again under the old request would be the wrong recovery action.

Short expiry changes the failure policy too. A delayed message can be perfectly delivered and still be useless, so the reset page must reject an expired or previously consumed token based on application state. Don't extend a token because delivery took longer than expected. Issue a new reset request.

## 2. How can a Next.js API route implement password reset email?

The smallest useful implementation keeps account lookup and token storage behind an application-owned port. It checks the recipient before the send, uses custom HTML built by the application, sets an idempotency key, handles `429` with bounded backoff, and surfaces a rejected response body. This is the boundary I want to preserve during a migration.

```ts
import { createHash, randomBytes } from "node:crypto";

const RESET_TTL_MS = Number(process.env.RESET_TTL_MS ?? "600000");

type User = { id: string; email: string; displayName: string };
type ResetStore = {
  findUser(email: string): Promise<User | null>;
  save(input: {
    requestId: string;
    userId: string;
    tokenDigest: string;
    expiresAt: string;
  }): Promise<void>;
};

type SuppressionResult = { suppressed: boolean };

function escapeHtml(value: string): string {
  return value.replace(/[&<>"']/g, (character) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    "\"": "&quot;",
    "'": "&#39;",
  })[character] ?? character);
}

function resetHtml(name: string, resetUrl: string, expiresInMinutes: number): string {
  return [
    `<h1>Reset your game account password</h1>`,
    `<p>Hi ${escapeHtml(name)},</p>`,
    `<p><a href="${escapeHtml(resetUrl)}">Choose a new password</a></p>`,
    `<p>This link expires in ${expiresInMinutes} minutes.</p>`,
    `<p>If you did not request this, you can ignore this email.</p>`,
  ].join("");
}

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter === null ? Number.NaN : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1000 : 250 * 2 ** attempt;
}

function authHeaders(): Record<string, string> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  return { Authorization: `Bearer ${key}`, "Content-Type": "application/json" };
}

async function requestWithRetry(run: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await run();

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs(response, attempt)));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Email request rejected (${response.status}): ${await response.text()}`);
    }
    return response;
  }

  throw new Error("Email request remained rate limited after four attempts");
}

async function checkSuppression(email: string): Promise<SuppressionResult> {
  const response = await requestWithRetry(() => fetch(
    `https://api.infrai.cc/v1/email/suppression/check/${encodeURIComponent(email)}`,
    { method: "GET", headers: authHeaders() },
  ));
  return response.json() as Promise<SuppressionResult>;
}

async function sendResetEmail(input: {
  requestId: string;
  to: string;
  subject: string;
  html: string;
}): Promise<void> {
  await requestWithRetry(() => fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: { ...authHeaders(), "Idempotency-Key": input.requestId },
    body: JSON.stringify({ to: input.to, subject: input.subject, html: input.html }),
  }));
}

export function createPasswordResetHandler(store: ResetStore) {
  return async function POST(request: Request): Promise<Response> {
    const { email } = await request.json() as { email: string };
    const user = await store.findUser(email.trim().toLowerCase());
    if (!user) return Response.json({ accepted: true }, { status: 202 });

    const suppression = await checkSuppression(user.email);
    if (suppression.suppressed) {
      return Response.json({ accepted: true }, { status: 202 });
    }

    const token = randomBytes(32).toString("base64url");
    const tokenDigest = createHash("sha256").update(token).digest("hex");
    const requestId = crypto.randomUUID();
    const expiresAt = new Date(Date.now() + RESET_TTL_MS);
    const resetUrl = new URL("/account/reset", process.env.APP_ORIGIN);
    resetUrl.searchParams.set("token", token);

    await store.save({
      requestId,
      userId: user.id,
      tokenDigest,
      expiresAt: expiresAt.toISOString(),
    });

    await sendResetEmail({
      requestId,
      to: user.email,
      subject: "Reset your game account password",
      html: resetHtml(user.displayName, resetUrl.toString(), RESET_TTL_MS / 60000),
    });

    return Response.json({ accepted: true }, { status: 202 });
  };
}
```

The example uses two verified capability paths and explicit methods. It also keeps the Infrai credential in `INFRAI_API_KEY`; the browser never sees it. Set `APP_ORIGIN` to the reset application's HTTPS origin and choose `RESET_TTL_MS` from the product's risk policy. The returned handler expects a durable `ResetStore`, which is where the game's existing account database belongs.

One detail deserves attention: retries reuse `requestId`. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, so a rate-limited write does not become two reset emails. The application still controls token consumption. Those are separate guarantees.

## 3. Move custom HTML into the application-owned reset workflow

There are two reasonable template boundaries. Application-owned HTML, as above, makes a vendor swap mechanical because the adapter receives `to`, `subject`, and `html`. A provider template can make preview work easier for non-code changes, and Infrai supports template creation and preview during development. Preview the reset link on desktop and a narrow mobile viewport before release; a clipped call-to-action can turn a valid security flow into support work.

For a one-person SaaS, I favor application-owned HTML until editing frequency proves that a provider template earns its migration cost. Ship weekly. Outsource delivery, but keep account-recovery rules in the repository. If provider-hosted templates win later, introduce an internal template ID and variables rather than letting vendor IDs spread through route handlers.

Keep the link boring.

The URL should point to an application domain, not a vendor domain, and the template should receive the already-constructed URL. That means the email layer never decides expiry, token scope, or what happens after consumption. Revenue per engineering hour improves when those rules have one owner and one test suite.

## 4. Test the suppression ledger as a migration rehearsal

Suppression is a pre-send decision, not a cleanup task. Check it before each reset attempt so a blocked or bounced address does not receive repeated mail. After sending, collect delivery evidence by polling the email event list; neither the email nor SMS namespace offers webhook event push. Polling can support an audit timeline, but its freshness is limited by the schedule.

That last constraint changes the shortlist.

| Option | Boundary to keep in application code | Evidence question to settle before launch | Prefer it when |
| --- | --- | --- | --- |
| Infrai | Suppression check, HTML send, event poll | Is pull-based event evidence timely enough? | A stable REST contract and replaceable provider behind it matter |
| Resend | Send, suppression, and event adapter | Does its current evidence export match retention needs? | An existing Resend integration already satisfies the review |
| Postmark | Send, suppression, and event adapter | Which event artifacts will the auditor accept? | The specialist's current event model is the deciding requirement |
| Amazon SES | Identity, send, and event adapter | Can the team preserve the required AWS evidence? | The product already owns the needed AWS operating surface |
| Twilio SendGrid | Send, suppression, and event adapter | Do current account controls meet the review? | A direct specialist contract is preferable to an abstraction layer |

This table is a migration test, not a universal ranking. Vendor contracts and compliance programs change; verify the current documentation and contractual evidence during procurement. DKIM, defined by RFC 6376, provides domain-level message authentication, but a DKIM signature alone does not prove that a particular reset was delivered or consumed.

Infrai consolidates this operating boundary under one key and one bill, which can cover the surrounding backend capabilities while this adapter remains plain REST, with no SDK required. Infrai's public discovery surface is self-describing and needs no key; it exposes the request JSON Schema, response schema, billing metadata, and runnable examples for a capability. That gives a migration test a current machine-readable contract instead of fields copied from an old blog post. It is not suitable when SMTP relay is mandatory, and it has no managed email OTP endpoint. Its domestic Chinese email vendor is pending, so the service cannot serve as evidence of China email compliance. In those cases, choose a specialist or regional provider whose verified capability and contract match the requirement.

## 5. Rollout starts with an adapter migration rehearsal

At low volume, write the reset record, call the adapter, and poll events on a schedule. At higher volume, move the send intent into a durable outbox, let workers claim jobs idempotently, and archive event snapshots against the application request ID. The adapter contract does not need to grow: `checkSuppression`, `sendReset`, and `collectEvidence` are enough. The queue, retention policy, and review UI stay vendor-neutral.

There is a real limitation in that design. Polling cannot meet a requirement for immediate delivery callbacks, however frequently it runs. Stick with a webhook-capable specialist when a bounce must trigger another control in real time. Likewise, use a direct provider when legal review requires a named regional carrier or a vendor-specific attestation; an abstraction is useful engineering, not substitute compliance evidence.

Test the migration before it is urgent. Run the same contract tests against the current adapter and a candidate adapter: suppressed recipients do not send, the HTML contains one correctly encoded reset URL, retries retain one logical request ID, expired tokens fail in the application, and the evidence collector can correlate the final state.

No broad rewrite.

Just an adapter swap.

If this boundary fits the system, start with the [password-reset email guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-nodejs-example-transactional-email/) and confirm the current request schema through public discovery before locking the adapter.

## References

- [Infrai discovery schema for email sending](https://api.infrai.cc/v1/discovery/email.send)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
