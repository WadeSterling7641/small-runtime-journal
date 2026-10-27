# SMS OTP 2FA Login API Explained for B2B Report Authorization

TL;DR: For a B2B SaaS product that emails a generated report, the least complex 2FA design is a short-lived OTP challenge with one send operation, a guarded resend, and a server-owned status record. Treat the SMS as a delivery attempt, not proof that the user received or used the code. Store enough evidence to explain the login and the report email later: challenge state, timestamps, destination country, attempt counts, provider-neutral delivery state, and the authenticated action that followed.

The practical choice is a small state machine. It is easier to audit than a client-side timer and easier to operate than a workflow spread across webhooks, polling, and email templates. Keep the report attachment separate from the OTP message. The report job can finish once; the email sender should attach the resulting file only after authorization succeeds. That boundary protects the report even when SMS delivery is slow.

| Choice | Use it when | Evidence you retain |
| --- | --- | --- |
| SMS OTP plus server status | The user needs a familiar second factor and the risk is moderate | Challenge, sends, verification observations |
| Authenticator or passkey | Account takeover has high impact or SMS is a weak factor | Enrollment and assertion records |
| Email link or code | The account already has a trusted mailbox and latency is acceptable | Token use and message event |

For a one-person SaaS, this is a revenue-per-hour decision. Ship the smallest flow that can answer an auditor's five-minute question, then outsource carrier-specific delivery work to a standards-compliant messaging service. Spending a week tuning carrier quirks is rarely the highest-value use of founder time.

Keep the failure visible.

No magic.

If the report is generated before the login is complete, the file can sit in temporary storage without becoming downloadable. If the SMS fails, the user can retry within policy; the report does not need to be rendered again. That separation is a small operational detail with a large debugging payoff: support can inspect the challenge and the report request independently, while a failed delivery cannot accidentally trigger duplicate report jobs or duplicate attachments.

## How should a Node.js 2FA login API handle SMS OTP?

A successful response from an SMS API proves very little about identity. It may mean that a request was accepted or that a message was handed to a downstream system. It does not prove delivery to a handset, and delivery does not prove that the person entering the code is the account owner. The application must keep those claims distinct.

For each challenge, record a random challenge identifier, a hash of the OTP, creation and expiry times, the normalized destination region, and counters for sends and verification attempts. A six-digit code has a finite search space, so rate-limit both verification attempts and new sends. Bind the challenge to the login transaction and invalidate it after a successful verification. Never put the raw code in logs, analytics events, report filenames, or the attachment itself.

The report email needs its own evidence. Record which authenticated user requested the report, which report version was rendered, the attachment checksum, and the time the message entered the outbound queue. These records answer a different question from SMS delivery: “Did the application authorize and generate this exact artifact?” Keep both trails, with a shared request ID for investigation.

## Which failure modes matter in OTP delivery?

Resend is a write operation. Polling is a read operation. Conflating them creates an accidental SMS loop when a browser retries a request or a mobile client wakes from the background. The server should enforce a resend interval, a per-challenge send ceiling, and an overall expiry. The client may display a countdown, but the server remains authoritative.

Polling should stop on a terminal state such as delivered, failed, expired, or verified. Use backoff instead of a fixed tight loop; a first interval of 2 seconds followed by 4, 8, and 15 seconds is enough for a short login window. Do not expose carrier payloads to the browser. Map them to a small internal vocabulary and retain the original event server-side when it is needed for support or evidence.

Here is the shape of the boundary. The transport adapter is intentionally generic, so the login policy does not depend on a vendor SDK.

```ts
type DeliveryState = "queued" | "sent" | "delivered" | "failed";

type OtpChallenge = {
  id: string;
  userId: string;
  destinationRegion: "US" | "EU" | "other";
  otpHash: string;
  createdAt: number;
  expiresAt: number;
  sendCount: number;
  verifyFailures: number;
  delivery: DeliveryState;
};

interface SmsTransport {
  send(input: { destination: string; body: string; requestId: string }): Promise<{
    messageId: string;
  }>;
}

async function resendOtp(
  challenge: OtpChallenge,
  destination: string,
  sms: SmsTransport,
  now = Date.now(),
): Promise<{ id: string; delivery: DeliveryState }> {
  const resendGapMs = 30_000;
  const maxSends = 3;

  if (now >= challenge.expiresAt) throw new Error("challenge_expired");
  if (challenge.sendCount >= maxSends) throw new Error("send_limit_reached");
  if (now - challenge.createdAt < resendGapMs && challenge.sendCount > 0) {
    throw new Error("resend_too_soon");
  }

  const requestId = `otp-${challenge.id}-${challenge.sendCount + 1}`;
  const result = await sms.send({
    destination,
    body: "Your sign-in code expires soon.",
    requestId,
  });

  challenge.sendCount += 1;
  challenge.delivery = "queued";
  await recordDeliveryAttempt({ challengeId: challenge.id, messageId: result.messageId, requestId });
  return { id: challenge.id, delivery: challenge.delivery };
}

async function recordDeliveryAttempt(input: {
  challengeId: string;
  messageId: string;
  requestId: string;
}): Promise<void> {
  // Persist an append-only event; do not store the OTP here.
  await auditLog.write({ type: "sms_otp_queued", ...input, at: Date.now() });
}
```

The example leaves persistence and hashing behind an application boundary on purpose. A production implementation should use a cryptographic random source, a slow password-hash-style function for the OTP hash, an atomic counter update, and an idempotency key for retries. Those are policy decisions worth testing directly.

## How do US and EU delivery rules change the design?

The country is not decoration. Store the destination country or region after normalization, apply the messaging program and consent rules appropriate to that route, and keep an opt-out path for non-essential traffic. CTIA's messaging guidance is a useful reference for US messaging expectations, while local rules and counsel remain necessary for EU deployments. An OTP needed to complete a login is operationally different from a marketing message, but it still deserves a clear purpose, sender identity, and abuse controls.

Email has a parallel operational concern. If the generated report is delivered by email, configure sender authentication and monitor the sending domain's reputation. Google's sender guidance covers authentication, spam rates, and transport expectations. Do not use the report email as a second factor merely because it is already available; the mailbox, the report recipient, and the login session may have different trust assumptions.

A small country-aware policy object is enough for the first release. Avoid embedding country rules in UI code. That makes a later policy change a deployment concern instead of a full client release.

## When is SMS the wrong runner-up?

SMS is a reasonable second factor when setup friction matters and the account's risk profile permits it. This design has a hard limit: delivery status cannot establish possession of a phone, and an OTP cannot compensate for weak account recovery. The trade-off is real: SMS is not suitable for privileged administration, recovery of high-value accounts, or environments where SIM-swap and number-recycling risks dominate. In those cases, a passkey or authenticator-based factor deserves priority, with SMS reserved for recovery only under additional checks.

The decision should be made from the harm of a false acceptance, the support cost of a failed delivery, and the evidence required by customers. A second factor that produces a green delivery status but no usable audit record is not a complete control. A modest flow with clear expiry, bounded retries, and a durable report checksum can be easier to defend than a more elaborate system that nobody can explain.

Ship weekly. Outsource the undifferentiated carrier plumbing. Keep the security policy, state transitions, and evidence model in your own codebase. That division preserves the part that affects customer trust while protecting the hours available for product work.

## Further reading

- Google, “Email sender guidelines”: https://support.google.com/a/answer/81126
- CTIA, “Messaging Interoperability and Compliance Best Practices”: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- OWASP, “Authentication Cheat Sheet”: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- NIST, “Digital Identity Guidelines”: https://pages.nist.gov/800-63-3/
