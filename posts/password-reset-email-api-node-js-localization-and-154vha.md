# Password Reset Email API: Node.js Localization and HTML Template Review

Short answer: keep reset and signup-verification decisions in Node.js, and use a transactional email API only as the delivery boundary for reviewed, localized HTML templates. For an e-commerce signup flow, the deciding constraint is compliance evidence: you need to show which message, locale, and link were sent, not merely that a request reached an email service.

That sounds less exciting than choosing a template editor. Good. A one-person SaaS has limited revenue-per-hour to spend on infrastructure, and the email must survive a security review while the product still ships weekly. Outsource the undifferentiated presentation work. Keep the security state and evidence in your application.

## A verification link is a state machine

Start with the event, not the markup. When a shopper creates an account, the application should create a verification record with a user identifier, a one-time token reference, an expiration timestamp, the selected locale, and the template version that was approved for that locale. The message contains a link derived from application state. It does not create that state.

## The evidence ledger comes before the email API

The evidence record should capture when the send was requested, which destination was used, and the delivery provider's accepted or rejected result. Keep the reset or verification secret out of ordinary logs. A record such as `token=abc123` is a credential leak wearing an audit label. Store a digest or an internal record identifier instead, and give the evidence record a documented retention and access policy.

This is the useful distinction:

| Concern | Application owns | Template or delivery layer owns |
|---|---|---|
| Account state | token creation, expiry, one-time redemption | nothing |
| Message presentation | approved template ID and locale selection | HTML, copy, and translated layout |
| Evidence | event ID, template version, request time, result | provider response metadata |
| Abuse controls | throttling, replay handling, and account-enumeration policy | transport and delivery attempts |

For password reset, the same record shape applies. The reset token must be invalidated after redemption, and a newer request should have a clear relationship to an older one. A pretty preview cannot prove either rule. It only proves that a particular set of variables rendered in a particular template.

Tiny detail. Big consequence.

## How should a Node.js API handle password reset email templates, HTML preview, and localization?

Treat preview as a release check. Render every supported locale with test data that includes a long display name, a long URL, and the exact expiry wording the application will pass at runtime. Review the HTML and the plain-text alternative, then record the approved template version. The live handler should select from an allowlist; it should never accept a template name supplied by the browser.

Separate templates per locale are easier to review than one large template containing conditional language branches. They create more records, though. If translations already move through pull requests and the team needs one atomic code release, repository-owned rendering may be the better fit. Your mileage may vary. The correct choice is the one that makes the rendered artifact and its reviewer easy to identify after an incident.

The application-to-provider contract can stay small. The adapter below deliberately contains no provider-specific route or payload. That omission is useful: a real integration can be tested against the selected API documentation without allowing vendor fields to spread into account-recovery code.

```ts
import { randomBytes, createHash, randomUUID } from "node:crypto";

type Locale = "en-US" | "de-DE" | "ja-JP";

type VerificationState = {
  accountId: string;
  tokenDigest: string;
  expiresAt: Date;
};

type TransactionalMessage = {
  to: string;
  templateId: string;
  locale: Locale;
  variables: {
    verificationUrl: string;
    expiresAt: string;
  };
};

type Evidence = {
  eventId: string;
  accountId: string;
  locale: Locale;
  templateId: string;
  requestedAt: string;
  destination: string;
  deliveryResult: "accepted" | "rejected";
};

type Dependencies = {
  saveState: (state: VerificationState) => Promise<void>;
  send: (message: TransactionalMessage) => Promise<{ result: Evidence["deliveryResult"] }>;
  saveEvidence: (evidence: Evidence) => Promise<void>;
};

const templateByLocale: Record<Locale, string | undefined> = {
  "en-US": process.env.VERIFICATION_TEMPLATE_EN_US,
  "de-DE": process.env.VERIFICATION_TEMPLATE_DE_DE,
  "ja-JP": process.env.VERIFICATION_TEMPLATE_JA_JP,
};

function digest(token: string): string {
  return createHash("sha256").update(token).digest("hex");
}

export async function sendVerificationEmail(
  account: { id: string; email: string; locale: Locale },
  origin: string,
  expiresAt: Date,
  dependencies: Dependencies,
): Promise<void> {
  const templateId = templateByLocale[account.locale];
  if (!templateId) throw new Error(`No approved template for ${account.locale}`);

  const token = randomBytes(32).toString("base64url");
  const verificationUrl = new URL("/verify-account", origin);
  verificationUrl.searchParams.set("token", token);

  await dependencies.saveState({
    accountId: account.id,
    tokenDigest: digest(token),
    expiresAt,
  });

  const requestedAt = new Date().toISOString();
  const delivery = await dependencies.send({
    to: account.email,
    templateId,
    locale: account.locale,
    variables: {
      verificationUrl: verificationUrl.toString(),
      expiresAt: expiresAt.toISOString(),
    },
  });

  await dependencies.saveEvidence({
    eventId: randomUUID(),
    accountId: account.id,
    locale: account.locale,
    templateId,
    requestedAt,
    destination: account.email,
    deliveryResult: delivery.result,
  });
}
```

The sample's contract is the important part. Test the `node:crypto` behavior in the project's target Node.js version; I’m not claiming a universal runtime setup here. Also validate that `origin` is an approved application origin, rather than accepting it from a request body.

The send operation needs explicit handling for timeouts, provider rejection, and retry safety. A retry after an ambiguous network result can duplicate a message unless the delivery API and your adapter support an idempotency strategy. Do not solve that uncertainty by logging the verification URL. Record an internal event ID and the provider's safe response metadata instead.

## Release checks catch drift, not security failures

Preview fails when it is treated as evidence of delivery or security. A screenshot cannot prove that the production locale mapping was the one used. It cannot prove that the expiry timestamp came from the same record as the token. It cannot show that redemption rejects a used token.

The failure mode I would watch first is configuration drift. Picture a normal Tuesday release: someone approves `en-US` version 12 after fixing a legal name, the staging preview shows version 12, and an environment variable in production still points to version 11. A shopper signs up, receives the old wording, and support asks which copy was sent. If the application recorded only a provider message ID, the answer now depends on a dashboard search and a human remembering what changed. If it recorded the selected locale and template version beside the signup event, the incident has a bounded question and a reproducible trail. The fix is not a more elaborate editor. Make template IDs and locale mappings explicit configuration, validate them at startup, and persist the selected version with each event.

The second failure is localization drift. A translator changes the visible expiry sentence but the application still sends a different duration. Pass the expiry value as data, keep its formatting rule inside the locale template, and test the complete variable contract. Long names matter because clipped text can hide the action or the warning. Plain text matters because some clients do not render HTML as expected.

The third failure is account enumeration. A reset endpoint that responds differently for a known and unknown address teaches an attacker which accounts exist. The email boundary cannot repair that response behavior. Return the same public response, throttle requests, and keep security telemetry separate from message content.

Short path. Fewer surprises.

## The delivery boundary has sharp edges

Compare the boundary, not the number of dashboard features. A focused email API can be a good fit when the team wants stored templates, a stable send operation, and provider-side delivery metadata. A repository-owned renderer can be better when review, localization, and deployment already live in code. SMTP can fit an organization with mature mail operations, but it does not remove the need for application-owned token state or evidence.

The catch is that a delivery API is not a compliance program. If the business needs a tamper-evident audit trail, defined data residency, or a particular deletion schedule, those requirements need confirmation from the chosen provider and the application's own records. The two reference documents below do not establish a universal legal rule for every market. I’m not sure which retention period or regional provider is correct without the product's jurisdictions and counsel; those inputs resolve the decision.

For the e-commerce signup scenario, I would choose the option that makes these questions easy to answer six months later: which account event triggered the message, which locale was selected, which reviewed template rendered, which link state was created, and what delivery result came back? If an option makes those answers depend on a dashboard screenshot or a manual search, it is a poor fit regardless of its template editor.

SMS deserves separate treatment. It is not a fallback HTML channel. Sender identity, consent, opt-out behavior, country rules, and message-length constraints belong in a channel-specific policy. CTIA's messaging guidance is useful evidence that interoperability and compliance are operational concerns, but a password reset that only needs email should not inherit an SMS workflow just because the provider offers both.

## What would I change at scale?

I would add a durable outbox between account state and delivery. The signup transaction writes the verification state and an outbound event; a worker claims that event, sends it with a stable idempotency key, and records an accepted, rejected, or explicitly unknown result. This keeps a provider timeout from turning the web request into a second token-generation path.

Then I would add contract tests for every locale, preview fixtures in CI, synthetic checks for the link's host and expiry wording, and dashboards that aggregate delivery results without exposing addresses or tokens. Those checks cost time once and pay back every week a copy change lands without a full application release.

Do not add them all on day one. For a small product, the minimum useful version is an allowlisted locale, a stored template version, hashed token state, one-time redemption, a redacted evidence record, and a tested adapter. The right boundary lets the business grow without making every feature release an email migration.

Keep the security decision local. Keep the reviewed message identifiable. Let the delivery service do the boring transport work, and change it when the evidence or operational requirements say to.

## References

- https://resend.com/docs/introduction
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
