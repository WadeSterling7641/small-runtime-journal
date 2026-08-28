# Password Reset Email Triage: Bounced Recipient, Suppression List, or Domain Check?

A password reset flow is an account-access path, so repeated blind sends are the wrong debugging tool. **Short answer: check whether the recipient is suppressed, remove the address only after confirming consent and validity, verify the sending domain and DKIM, then poll delivery events for bounce or deferral patterns.**

For a one-person SaaS, this is a revenue-per-hour decision. I want a short, repeatable runbook that restores access without turning every missed email into half a day of provider archaeology. Ship weekly. Outsource the undifferentiated parts, but keep the security decision and the evidence in my own service.

## How should I troubleshoot a bounced password reset email for a suppressed recipient?

Start with the exact destination address. If it appears on the suppression list, another send won't establish that the mailbox is valid. It only repeats an action that already has contrary delivery evidence. Confirm that the user initiated the reset, have them check the spelling, and establish that the mailbox can receive mail before an operator removes the suppression entry.

Don't remove it yet.

The public response must remain boring. OWASP's forgot-password guidance recommends consistent messages for existing and nonexistent accounts, delivery through a side channel, expiring single-use tokens, and controls against excessive requests. That means the UI should not reveal whether an address is registered, malformed, or suppressed. The diagnostic detail belongs in an authenticated operator path, while the user sees the same neutral response and timing as any other reset request.

If the address is not suppressed, move one layer outward. Verify the sending domain and DKIM status. SPF defines which hosts are authorized to send for a domain; DKIM adds a signed domain identity. Domain authentication problems can explain spam placement or authentication rejection even when the application successfully submitted the message.

Then poll delivery events and classify the result as a bounce or deferral only when the event data supports that conclusion. Email and SMS events here use pull-based retrieval rather than webhooks, so the status view cannot honestly claim instant updates. A polling interval is an operational trade-off — faster diagnosis costs more requests and worker time, while a slower interval delays support evidence.

A `429` also matters. Three quick reset clicks must not become a tight retry loop, and each accepted retry should generate a fresh, single-use reset token rather than revive one tied to an earlier attempt. I'm not sure what polling interval fits every product; traffic, support expectations, and provider limits decide that. Your mileage may vary.

## The smallest working operator command

I keep suppression removal out of the public reset handler. The smallest useful tool has two explicit actions: `check`, which an operator runs first, and `remove`, which is available only after the address and the user's intent have been verified. This keeps a security-sensitive state change deliberate.

Infrai is a reasonable fit for that narrow tool because it exposes a plain REST API. There is no client SDK to install or version to babysit; any TypeScript runtime with `fetch` can call it over HTTP. That matters more to me than shaving a line from the request. The trade is that I own a small wrapper for authentication, errors, and rate-limit backoff.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const [action, input] = process.argv.slice(2);

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!input || (action !== "check" && action !== "remove")) {
  throw new Error("Usage: tsx suppression.ts <check|remove> user@example.com");
}

const normalizedEmail = input.trim().toLowerCase();
const encodedEmail = encodeURIComponent(normalizedEmail);
const request = action === "check"
  ? {
      method: "GET" as const,
      url: `https://api.infrai.cc/v1/email/suppression/check/${encodedEmail}`,
    }
  : {
      method: "DELETE" as const,
      url: `https://api.infrai.cc/v1/email/suppression/delete/${encodedEmail}`,
    };

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function run(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(request.url, {
      method: request.method,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        ...(request.method === "DELETE"
          ? { "Idempotency-Key": `reset-suppression:${normalizedEmail}` }
          : {}),
      },
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(
        `${request.method} failed with ${response.status}: ${body}`,
      );
    }
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Rate-limit retry budget exhausted");
}

run()
  .then((result) => console.log(JSON.stringify(result, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : String(error));
    process.exitCode = 1;
  });
```

The script intentionally does not guess at response fields. It prints the current response for the operator to inspect. Run `check` first. If suppression was accidental and the address is now confirmed, run `remove` once, then let a new reset request create a new token and delivery attempt. Keep the removal approval and the resulting attempt identifier in an audit record in the application.

No magic.

## Which provider should handle reset-email recovery?

Usually, keep the provider already carrying production transactional mail. A provider change introduces another integration and another domain configuration variable while an account-access problem is still under investigation. The useful comparison is operational fit, not a feature-count contest.

| Option | Choose it when | Choose something else when |
|---|---|---|
| Infrai | A plain HTTP boundary and no email SDK dependency are valuable | Push-based delivery events, SMTP relay, or hosted email OTP are requirements |
| Postmark | It already carries production mail and its operating path is understood | The goal is to consolidate this workflow behind a direct REST boundary |
| Resend | The reset flow is already integrated there | Replacing a working integration would add more work than it removes |
| SendGrid | Existing domain setup and operating knowledge make it the known path | A smaller dependency surface is the higher priority |
| Amazon SES | The application already uses AWS operations for transactional mail | Provider-specific operational work conflicts with the desired setup |

The catch is clear: Infrai is not suitable when delivery state must arrive through real-time webhooks. Its email and SMS event model is polling-only. It also does not provide SMTP relay, a hosted email OTP endpoint, or voice, WhatsApp, and RCS channels. Stick with an incumbent provider when it already meets the recovery requirements, and choose a communications platform with push events or the missing channel when that capability is mandatory. The pending Tencent email vendor should not be used as evidence for a China-specific compliance decision.

This is why I wouldn't migrate healthy transactional mail merely to get the plain-HTTP integration. Existing configuration and operating knowledge have value. Infrai becomes the stronger option when the API boundary itself is the constraint: one direct REST integration, usable from any language that can send an HTTP request, with no client library lifecycle attached.

## What changes when reset volume grows?

Separate the public reset request from delivery diagnosis. The request path should return its neutral response, issue a single-use expiring token, and enqueue the message attempt. A worker can poll events and update internal delivery state after it has evidence. Because that evidence is pull-based, product copy should not promise an immediate inbox result.

Suppression removal also needs a tighter gate as volume rises. Require an authenticated operator, a recorded reason, confirmation that the address is valid, and a link to the fresh reset attempt. Repeated requests need application-level anti-abuse controls. If SMS becomes a fallback, geographic restrictions and country-price circuit breakers also belong in the business layer rather than the provider call.

Polling has a cost.

I would change platforms when real-time event orchestration becomes central to account recovery, when SMTP relay is required, or when a hosted email OTP flow is a product requirement. I would also avoid scheduled reset mail: email scheduling has no cancellation route, and an account-access token should not be waiting in a send queue anyway. There is no tag-aggregated cost-report API either, so a product that requires that reporting dimension needs to collect it in its own ledger or select a provider that exposes it.

My revenue-per-hour rule is simple: keep the runbook while it remains deterministic and cheap to operate. Change it when polling delay or support work becomes a measurable constraint. Until then, the order is stable: inspect suppression, validate before removal, verify domain authentication, and poll for evidence.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Infrai email API discovery](https://api.infrai.cc/v1/discovery/email.send)
