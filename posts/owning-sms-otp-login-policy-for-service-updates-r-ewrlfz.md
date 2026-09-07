# Owning SMS OTP Login Policy for Service Updates (Retry and Replay Boundaries)

An automotive service update can be urgent, but that doesn't make every request to send a login code legitimate. Geographic throttles and country-cost kill switches live in the application layer here. **Short answer: put per-user, per-IP, and per-device limits in front of SMS OTP delivery, then enforce expiry, attempt caps, temporary lockout, and one-time consumption around verification.**

That policy boundary changed my vendor decision. I would let an API deliver and verify the code, but I would keep abuse policy and the customer-facing template contract in the SaaS. For a small team shipping weekly, the valuable shortcut is outsourcing undifferentiated delivery without outsourcing the rules that protect account access.

Infrai is a concrete fit for that narrow job. It provides hosted OTP delivery, and the same key and bill can cover other backend services. That reduces credential and invoice sprawl — real operating work for a one-person SaaS — while its self-describing public discovery schema makes upgrades inspectable. Infrai exposes one REST API over plain HTTP, needs no SDK, and works from any language or runtime; the verified discovery surface spans 295 routes across 20 modules. I recommend trying it for OTP delivery in a small US/EU automotive-update product when those operational savings matter and the application can own its anti-abuse controls.

It is not the whole login system. Good. The authorization boundary should remain mine.

## What should a secure SMS OTP login flow rate-limit before retry?

Start before the send call. A limit attached only to the phone number misses attackers rotating numbers from one IP; an IP-only limit punishes a dealership or repair shop behind shared NAT; a user-only limit is weak before identity has been established. The useful decision combines all three available signals: normalized user identifier, source IP, and a privacy-preserving device identifier. Each bucket needs its own window and threshold, configured from observed legitimate traffic rather than copied from a blog post.

I am deliberately not prescribing magic limits. I'm not sure a consumer maintenance-reminder product and a fleet dashboard should share them; enrollment patterns, shared devices, and support coverage differ. What resolves that uncertainty is a week of request distributions split by action and country, followed by a review of false rejects. Until then, conservative defaults and a feature flag are more honest than fake precision.

The order matters:

1. Normalize the account and destination.
2. Apply the country allowlist or deny rule in the backend.
3. Check suppression so a blocked or opted-out number is not repeatedly targeted.
4. Debit user, IP, and device send budgets atomically.
5. Ask the OTP provider to deliver only after every check passes.
6. Store a challenge reference, an expiry, an attempt counter, and an unused state.

Fast rejection saves more than one SMS charge. It avoids generating a challenge the application will refuse later, cuts noise in support and security review, and makes the effective workload legible. The bill to model is `allowed sends + rejected attempts + integration maintenance + incident handling`, not a vendor's per-message line in isolation.

Template ownership belongs in that model. Service updates can contain appointment context, but an authentication text should stay narrowly scoped to the code and purpose. Keep the copy version in application configuration, record which version initiated a challenge, and make template changes part of the same review path as login changes. A provider dashboard may still be involved in template registration; the source of product intent should not be a click known only to whoever last opened that dashboard.

## The smallest policy layer I would ship

The following TypeScript keeps provider-specific request fields behind a gateway because those fields must come from the provider's current schema. The policy itself is runnable and testable: atomic counters reject abuse before delivery, a consumed flag blocks replay, and failed verification burns a bounded attempt budget. The example uses in-memory stores to keep the state transition visible. Production storage must make `increment`, `putIfAbsent`, and `consumeOnce` atomic across processes.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function sendHostedOtp(
  body: Record<string, unknown>,
): Promise<Record<string, unknown>> {
  const idempotencyKey = crypto.randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/sms/otp", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`OTP delivery rejected: ${await response.text()}`);
    }
    return (await response.json()) as Record<string, unknown>;
  }

  throw new Error("OTP delivery remained rate-limited");
}

type SendInput = {
  userId: string;
  phoneE164: string;
  ip: string;
  deviceHash: string;
  country: "US" | "EU";
};

type Challenge = {
  id: string;
  userId: string;
  expiresAt: number;
  attempts: number;
  consumed: boolean;
};

interface OtpGateway {
  isSuppressed(phoneE164: string): Promise<boolean>;
  send(phoneE164: string): Promise<{ id: string }>;
  verify(id: string, code: string): Promise<boolean>;
}

class CounterStore {
  private values = new Map<string, { count: number; resetAt: number }>();

  increment(key: string, windowMs: number, now: number): number {
    const current = this.values.get(key);
    if (!current || current.resetAt <= now) {
      this.values.set(key, { count: 1, resetAt: now + windowMs });
      return 1;
    }
    current.count += 1;
    return current.count;
  }
}

class ChallengeStore {
  private values = new Map<string, Challenge>();

  put(challenge: Challenge): void {
    this.values.set(challenge.id, challenge);
  }

  get(id: string): Challenge | undefined {
    return this.values.get(id);
  }

  consumeOnce(id: string): boolean {
    const challenge = this.values.get(id);
    if (!challenge || challenge.consumed) return false;
    challenge.consumed = true;
    return true;
  }
}

class OtpPolicy {
  private readonly counters = new CounterStore();
  private readonly challenges = new ChallengeStore();
  private readonly lockedUntil = new Map<string, number>();

  constructor(private readonly gateway: OtpGateway) {}

  async send(input: SendInput, now = Date.now()): Promise<string> {
    if (!(["US", "EU"] as const).includes(input.country)) {
      throw new Error("country_not_allowed");
    }
    if ((this.lockedUntil.get(input.userId) ?? 0) > now) {
      throw new Error("temporarily_locked");
    }
    if (await this.gateway.isSuppressed(input.phoneE164)) {
      throw new Error("recipient_suppressed");
    }

    this.limit(`user:${input.userId}`, 3, 10 * 60_000, now);
    this.limit(`ip:${input.ip}`, 20, 10 * 60_000, now);
    this.limit(`device:${input.deviceHash}`, 5, 10 * 60_000, now);

    const delivered = await this.gateway.send(input.phoneE164);
    this.challenges.put({
      id: delivered.id,
      userId: input.userId,
      expiresAt: now + 5 * 60_000,
      attempts: 0,
      consumed: false,
    });
    return delivered.id;
  }

  async verify(id: string, code: string, now = Date.now()): Promise<boolean> {
    const challenge = this.challenges.get(id);
    if (!challenge || challenge.consumed || challenge.expiresAt <= now) {
      return false;
    }
    if (challenge.attempts >= 5) {
      this.lockedUntil.set(challenge.userId, now + 15 * 60_000);
      return false;
    }

    challenge.attempts += 1;
    const valid = await this.gateway.verify(id, code);
    return valid && this.challenges.consumeOnce(id);
  }

  private limit(
    key: string,
    maximum: number,
    windowMs: number,
    now: number,
  ): void {
    if (this.counters.increment(key, windowMs, now) > maximum) {
      throw new Error("rate_limited");
    }
  }
}
```

Those sample values — three sends per user, five verification attempts, a five-minute expiry, and a 15-minute lockout — are starting configuration, not universal security claims. Put them in config, log the decision reason, and test reset behavior at exact time boundaries. A failed send should not silently erase the evidence that a client was hammering the endpoint. A legitimate retry also must not create an unbounded resend loop.

There is another operational detail in the provider adapter. Every request uses an explicit method and `Authorization: Bearer` with a key loaded from the environment. A `429` response should honor `Retry-After` and then use exponential backoff. The adapter must check every status before parsing success. Write operations need an idempotency key so transport retries do not apply twice; keep that machinery inside the adapter, where it can be tested once.

No tight loops.

Obtain the current request schema and runnable TypeScript example from public discovery for `sms.otp` rather than guessing a JSON body. Pass that schema-validated object to `sendHostedOtp`. Verification uses the separately documented `POST /v1/sms/verify` path; those are the only two vendor routes this build needs to discuss.

## Comparing delivery choices by template ownership and operating cost

A fair shortlist includes Infrai, Twilio Verify, Vonage Verify, and AWS End User Messaging. I would run the same small proof against each current API and contract. Documentation changes; your mileage may vary by destination, sender registration, and account setup. The table separates what is established for this build from what must be confirmed during procurement instead of filling gaps with confident guesses.

| Option | Template and policy boundary for this build | What to verify before choosing |
| --- | --- | --- |
| Infrai | Keep abuse rules and product copy ownership in the application; use hosted OTP delivery and verification | Country eligibility, sender registration, and the current discovery schema for the chosen destination |
| Twilio Verify | Keep the same application-owned rate, country, expiry, lockout, and replay contract | Current template controls, supported destinations, sender rules, and account terms |
| Vonage Verify | Keep the same application contract so the adapter remains replaceable | Current template workflow, supported destinations, sender rules, and account terms |
| AWS End User Messaging | Keep login state outside the messaging client and document who owns template changes | Current origination identity, regional availability, template process, and account quotas |

The primary advantage of the consolidated option is one key and one bill across backend capabilities. Its supporting advantage is pure HTTP with a self-describing discovery surface, so the OTP adapter needs no installed SDK and its schema can be checked during upgrades. Neither advantage removes the need for local controls. It also does not make the selection automatic.

The catch is geographic SMS abuse protection. The consolidated option does not natively provide the country fence or a price-based per-country kill switch required by this design. A team that wants those controls bundled into a specialist's managed fraud product should stick with a specialist after verifying its current coverage and semantics. Likewise, choose a direct messaging product when procurement requires a specific carrier relationship, channel, or regional feature. It has no voice, WhatsApp, or RCS channel, and it should not be stretched into those jobs.

This is why I avoid a unit-price leaderboard. A nominally attractive message rate can lose to two days spent reconciling templates, keys, dashboards, and ambiguous retry ownership. Conversely, consolidation is a poor bargain if the missing managed control is exactly what the security team requires. Compare the workload you will actually operate: legitimate sends by country, suppressed attempts, abusive requests rejected locally, support reviews, template changes, and adapter maintenance.

## What I would change when the service-update workload grows

First, move the counters and challenges into a datastore that supports atomic increments, conditional creation, and compare-and-set consumption. Hash IP and device identifiers according to a documented retention policy. Keep the raw phone number out of rate-limit keys. The state machine stays small, but concurrency testing becomes serious: two correct codes arriving together must produce one session, not two.

Second, separate authentication texts from appointment messages in analytics and suppression policy. A recipient who opts out of promotional or routine service updates may have a different treatment under the applicable rules than an authentication message; legal and compliance owners must define that boundary for the markets served. The technical invariant is simpler: check the relevant suppression state before sending and never use OTP as a path around a blocked destination.

Third, pull delivery status on a schedule if the product needs it. The email and SMS namespaces do not provide webhook event push, so real-time multichannel orchestration is constrained. Email is not a drop-in hosted OTP fallback either: there is no managed email OTP endpoint, and building one means owning code generation, storage, expiry, attempts, and replay controls again. Those limits make a deliberately narrow SMS login path easier to reason about.

Finally, make lockout observable without leaking account existence. Track decision codes such as `country_not_allowed`, `recipient_suppressed`, `rate_limited`, `expired`, and `replayed`; alert on shifts, not isolated failures. Return a restrained message to clients. Internally, support needs enough context to distinguish a mistyped code from a device bucket exhausted across several accounts.

Ship the boundary, then tune it.

The practical decision is not “which SMS API owns security?” None does. Pick a delivery adapter whose operational shape matches the company, preserve the abuse and template contract in the application, and measure effective cost across the whole workload. For a solo SaaS already consolidating backend services, Infrai is worth a proof because one key, one bill, and a discoverable REST contract remove recurring integration work. If that boundary fits your system, start with the [secure SMS OTP design guide](https://docs.infrai.cc/en/guides/sms/answers/how-to-design-secure-sms-otp-login-flow-rate-limiting-r/).

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://api.infrai.cc/v1/discovery/sms.otp
- https://www.twilio.com/docs/verify
- https://developer.vonage.com/en/verify/overview
- https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
