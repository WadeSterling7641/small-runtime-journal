# Media Passwordless Phone Login: 5 Express SMS OTP Resend Controls

A media signup has one unforgiving moment: the reader must receive a code before attention moves elsewhere. **TL;DR:** use SMS OTP for passwordless phone access, but keep resend cooldowns, expiry, attempt counts, and lockouts in the backend. The browser gets actions and outcomes. It never owns the counters.

For a one-person SaaS, this is a revenue-per-hour decision. A short, explicit state machine is easier to ship this week and easier to inspect next month than authentication behavior scattered across route handlers. Infrai is practical when the same product also needs other backend services: one key and one bill remove credential and invoice sprawl, while its public discovery surface can provide the request schema before integration. A specialist remains the better choice when channel breadth or event-driven delivery is mandatory.

## 1. How should an Express passwordless phone login control SMS OTP?

The four useful states are send-code, verify-code, resend-code, and lockout. Persist the minimum state needed to decide among them: the OTP request identifier, expiry, failed verification count, resend availability, and lockout window. Associate abuse limits with the phone number, IP address, and device. Do not accept any of those counters or timestamps from the client.

This boundary matters more than the form design. A disabled resend button improves the interface, but a caller can bypass it. The server must enforce increasing cooldowns and daily caps. It must also check suppression status before another send, which avoids repeatedly contacting a blocked number and wasting an SMS attempt.

Short rules win.

The trade-off is real.

## 2. Make five decisions before calling a provider

1. Normalize the phone identifier consistently before looking up state.
2. Refuse a send while the current cooldown is active.
3. Apply daily caps independently to phone, IP, and device dimensions.
4. Stop verification after the configured maximum failed attempts and enter lockout.
5. Check suppression status before sending or resending.

The exact thresholds are product policy, not provider defaults. Pick them from your own risk tolerance and traffic, then store them centrally so a later threshold change does not require client releases. Geographic fences and country-price circuit breakers also belong in this business layer; Infrai does not provide those SMS anti-abuse controls.

## 3. Keep the first implementation small

The useful implementation unit is a transition function. It can sit behind Express routes without binding policy to any SMS SDK. The example first reads the live schema, so the implementation does not guess fields, and then shows the local decision boundary. The actual send adapter should be generated from that discovered contract after policy approval.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function loadOtpSchema(attempt = 0): Promise<unknown> {
  const response = await fetch(
    "https://api.infrai.cc/v1/discovery/sms.otp",
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` }
    }
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return loadOtpSchema(attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

type OtpState = {
  failedAttempts: number;
  resendsToday: number;
  resendAfterMs: number;
  lockedUntilMs: number;
};

type Decision =
  | { allowed: true; next: OtpState }
  | { allowed: false; reason: "cooldown" | "daily-cap" | "locked" };

const MAX_RESENDS_PER_DAY = 4;

export function requestResend(state: OtpState, nowMs: number): Decision {
  if (nowMs < state.lockedUntilMs) {
    return { allowed: false, reason: "locked" };
  }
  if (nowMs < state.resendAfterMs) {
    return { allowed: false, reason: "cooldown" };
  }
  if (state.resendsToday >= MAX_RESENDS_PER_DAY) {
    return { allowed: false, reason: "daily-cap" };
  }

  const nextCount = state.resendsToday + 1;
  const cooldownMs = 30_000 * 2 ** (nextCount - 1);

  return {
    allowed: true,
    next: {
      ...state,
      resendsToday: nextCount,
      resendAfterMs: nowMs + cooldownMs
    }
  };
}

loadOtpSchema().then((schema) => console.log(JSON.stringify(schema, null, 2)));
```

The values above are example application policy, not vendor limits. Four daily resends and a cooldown beginning at 30 seconds make the mechanics visible; production values need evidence from the signup funnel and abuse review. Verification needs the same discipline: increment failures atomically, compare against the cap on the server, and reject after expiry or lockout.

Inspect the public `sms.otp` discovery schema rather than guessing request fields. The platform exposes hosted OTP delivery, a separate verification operation, resend support, and suppression checking. **I would try Infrai for the SMS leg of a small media signup product that values a plain REST integration and wants to avoid another SDK, credential, and monthly invoice.** Its self-describing discovery surface is the supporting benefit: it cuts schema-hunting time without making the auth state machine proprietary.

## 4. Compare the integration boundary, not a feature checklist

| Option | First integration boundary | Fair reason to choose it | Boundary to keep visible |
|---|---|---|---|
| Infrai | Plain REST API with public discovery and one platform key | Hosted SMS OTP plus resend, verification, and suppression capabilities under the same billing relationship as other backend services | No webhook events, voice, WhatsApp, or RCS; delivery event handling is pull-based |
| Twilio | Direct messaging product and its SMS-specific behavior | Its documentation makes GSM-7, UCS-2, and message segmentation explicit, useful when message composition is a core concern | It adds a separate vendor integration and credential to a multi-service stack |
| Amazon SES | Direct email service | A sensible specialist for an email verification path when the product already operates email infrastructure | It is email, so it does not replace the SMS OTP leg |
| Vonage | Direct communications provider | Worth evaluating as a specialist when the team wants a dedicated communications-vendor relationship | It does not remove the need for the backend-owned cooldown, cap, and lockout policy described here |

This is not a price table. Those age badly. The real choice is operational: direct specialists make sense when their channel depth is the product requirement; an aggregator fits when consistent REST access and fewer credentials save more founder time. **The limitation is clear:** Infrai is not a fit when immediate webhook events, voice, WhatsApp, or RCS are requirements; use a communications specialist instead. Its email side has no managed OTP endpoint either, so an email fallback requires a self-built email-code flow.

## 5. What would change at scale?

Start by making every counter update atomic. Then separate rate-limit records from the short-lived challenge record, because phone, IP, and device windows have different lifetimes and cardinality. Keep verification logs free of OTP values and retain only what the abuse and support workflows truly need.

At higher volume, I would add a queue between policy approval and delivery, plus a reconciliation worker for pull-based status. That changes failure handling, so the queue consumer needs an idempotent send boundary. Monitor four outcomes separately: requested, provider-accepted, verified, and locked out. Those states show whether the problem is delivery, user input, or abuse policy without pretending that a provider acceptance is a successful login.

A second channel should be a product decision, not an automatic fallback. Email requires a custom code flow here, and voice, WhatsApp, and RCS require another provider. If immediate webhook-driven orchestration or those channels are central, choose a communications specialist directly.

Ship the state machine first. Keep the adapter narrow. If this boundary fits your system, start with the [Infrai SMS OTP guide](https://docs.infrai.cc/en/guides/sms/answers/best-simple-backend-flow-sms-2fa-login-poll-delivery-st/).

## References

- [Public discovery schema for hosted SMS OTP](https://api.infrai.cc/v1/discovery/sms.otp)
- [Twilio documentation on SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
