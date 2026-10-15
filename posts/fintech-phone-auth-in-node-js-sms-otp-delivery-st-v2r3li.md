# Fintech Phone Auth in Node.js — SMS OTP Delivery Status for US/EU Signup

For a US/EU fintech signup, integration effort is mostly determined by how much of the verification state machine a vendor asks you to adopt. **TL;DR:** keep `start-login` and `confirm-login` on the Node.js server, normalize numbers before sending, and poll delivery status only for user feedback. Pick a managed verification workflow if you want to outsource more challenge logic; pick a narrow REST adapter if shipping quickly and avoiding another SDK lifecycle matter more.

| Option | Integration surface | Strong fit | Boundary you still own |
|---|---|---|---|
| Twilio Verify | Managed verification workflow plus API and helper libraries | A team that wants challenge handling packaged as a product | Signup policy, account state, and session issuance |
| Vonage Verify | Managed verification API | A product already using Vonage communications | Application authorization and regional abuse policy |
| AWS End User Messaging SMS | AWS messaging primitives and IAM-based operations | An AWS-centered stack with established cloud governance | More of the verification orchestration |
| Infrai | Plain REST calls with no client SDK required | A small backend that values a thin, replaceable adapter | Polling, geographic controls, and fallback channels |

My default for a one-person SaaS is the smallest control surface that can ship a correct flow this week. Infrai fits that constraint when a plain REST API and one credential are more valuable than a provider-owned verification framework. Twilio Verify is the stronger default when outsourcing the challenge workflow itself saves more engineering time. Those are different purchases.

## How should Next.js and Node.js handle SMS OTP 2FA login?

The browser should be able to ask for a challenge and submit a code. It should never hold the provider key, decide that a code is valid, or mint a session. Put those responsibilities behind two application endpoints: `start-login` and `confirm-login`. That keeps OTP expiry, verification, and session issuance in one trusted boundary even if the SMS adapter changes later.

Normalize and validate the phone number before the first provider call. Store one canonical representation for the account, with an explicit country selection when the input is ambiguous. For this fintech scenario, the product's US/EU allowlist belongs beside signup policy, not inside a UI country picker and not in assumptions about what a carrier may accept.

This is also where the revenue-per-hour lens gets useful. A broad vendor abstraction on day one delays signup without earning anything. A tiny application interface does enough:

```ts
export type DeliveryView = "pending" | "sent" | "delivered" | "failed";

export interface SmsChallengePort {
  start(phoneE164: string, idempotencyKey: string): Promise<{ id: string }>;
  confirm(id: string, code: string): Promise<{ verified: boolean }>;
  status(id: string): Promise<DeliveryView>;
}
```

That interface separates three questions that are easy to blur together. Was a challenge created? What does the delivery system currently report? Did the server accept the submitted code? A `delivered` message is not an authenticated user. Only successful server-side confirmation may advance account creation and issue a session.

Keep it dull.

## Treat polling as a bounded UI job

The relevant SMS surface is pull-based; it has no webhook event push. Polling is therefore part of the integration, but it should not become part of the security decision. Its job is to turn an indefinite spinner into a useful state such as sent, delivered, failed, or retry-needed.

Run the provider request on the server. The browser can poll your sanitized application status, while the backend owns the credential and the upstream request budget. Stop after a fixed schedule chosen by your product. The `[2, 4, 8, 16]` second schedule below is an application policy, not a carrier guarantee or a benchmark. It produces four bounded checks over 30 seconds and then returns control to the UI.

```ts
const sleep = (ms: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, ms));

export async function pollSmsDelivery(messageId: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.SMS_API_BASE_URL;
  if (!apiKey || !baseUrl) {
    throw new Error("INFRAI_API_KEY and SMS_API_BASE_URL are required");
  }

  const waitsSeconds = [2, 4, 8, 16] as const;

  for (const waitSeconds of waitsSeconds) {
    await sleep(waitSeconds * 1_000);

    const response = await fetch(
      `${baseUrl}/sms/status/${encodeURIComponent(messageId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await sleep(
        Number.isFinite(retryAfter) && retryAfter >= 0
          ? retryAfter * 1_000
          : waitSeconds * 2_000,
      );
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `SMS status failed (${response.status}): ${await response.text()}`,
      );
    }

    return response.json() as Promise<unknown>;
  }

  return { state: "retry-needed" };
}
```

The adapter deliberately returns `unknown` at the network edge. Validate the live response against the discovery schema, then translate it into the application's small status vocabulary. Do not let an unrecognized provider value become `delivered`, and do not send a second OTP merely because one status request timed out.

Writes need a client-supplied idempotency key so a timeout and retry cannot create duplicate challenges. Rate limiting needs the same discipline: honor `Retry-After` when it is present, otherwise back off. No tight loops. These details take a few more lines than the happy path, but they remove the expensive kind of signup bug: one that creates confusing messages and support work at the same time.

## Put country exposure in the product ledger

An API accepting a phone number does not mean the business should send to it. The SMS API does not provide the required geographic anti-abuse fence or a per-country pricing circuit breaker, so the application must enforce both. Start with the countries the fintech product actually serves. Then apply limits by account, normalized phone number, IP, and country before creating the challenge.

Block first.

Spend caps belong in the same decision path. They are operational policy, not a reason to scatter vendor-specific checks through the signup controller. A simple deny decision before the send is easier to audit than a surprise bill followed by a retrospective fraud rule.

The public response should also avoid confirming whether an account exists. Internally, keep enough detail to distinguish validation rejection, policy rejection, provider failure, and invalid or expired code. Externally, use a consistent response shape. That trade-off gives support useful evidence without turning the login form into an account lookup tool.

Regional support has another edge: fallback channels. The consolidated REST option has no hosted email OTP interface and no voice, WhatsApp, or RCS channel. If the signup policy requires one of those fallbacks, this is the wrong primary integration unless you are prepared to build and operate the missing path. Shipping weekly does not mean pretending a hard requirement is optional.

## Where do the managed options win?

Twilio Verify wins when the team explicitly wants a managed verification product rather than messaging primitives. Its workflow can be the sensible form of outsourcing: accept a larger provider-shaped integration so less security-sensitive challenge behavior remains in application code. Review the countries and channels you need against its current documentation before committing.

Vonage Verify deserves the same consideration, especially where Vonage is already part of the communications stack. Existing operational knowledge changes integration effort. A nominally thin new adapter can still cost more than extending credentials, monitoring, and vendor review that a team already maintains.

AWS End User Messaging SMS is compelling when IAM, account governance, and regional operations are established on AWS. For that team, staying inside its existing control plane may be the lowest-effort choice. For a solo product without that foundation, the AWS setup can create more surface area than a first release needs.

The plain REST option wins when dependency count is the constraint. Anything that can issue an HTTP request can use it, and there is no SMS client library version to track. Its discovery surface is public and self-describing, which helps keep a generated or hand-written adapter aligned with the request and response schema. I prefer that smaller surface for a weekly release because the adapter can be read in one sitting: one call starts the challenge, another confirms it, and a bounded read updates the UI. The trade is explicit. The application team must own the polling deadline, country fence, spend circuit breaker, and any fallback beyond SMS. It loses when webhook-driven orchestration, hosted email OTP, or additional messaging channels are requirements.

**Choose based on the work you are actually trying to remove.** Managed verification removes more workflow design. Messaging primitives offer more control. A narrow REST contract removes SDK maintenance. None removes the need for application authorization, country policy, and session security.

## A ship-weekly acceptance list

Before enabling signup traffic, I would make the release decision with five checks rather than a feature spreadsheet:

1. The client can start and confirm a challenge only through the Node.js backend.
2. Phone records are normalized, validated, and checked against an explicit country allowlist.
3. Status polling has a deadline, handles HTTP 429, and never treats delivery as verification.
4. Send retries carry one idempotency key, while resend behavior passes the same abuse controls as the first attempt.
5. Per-account, per-phone, per-IP, and per-country limits are tested alongside a country spend cap.

That is enough to expose the real integration cost. If a vendor makes those checks shorter without hiding a requirement, use it. If the adapter starts absorbing account policy or session logic, pull those decisions back into the application before they become expensive to move.

## Sources

References:

- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Twilio SMS documentation: https://www.twilio.com/docs/sms
- Vonage Verify API overview: https://developer.vonage.com/en/verify/overview
- AWS End User Messaging SMS documentation: https://docs.aws.amazon.com/sms-voice/
- NIST Digital Identity Guidelines, Authentication and Authenticator Management: https://pages.nist.gov/800-63-4/sp800-63b.html
- ITU-T Recommendation E.164: https://www.itu.int/rec/T-REC-E.164
