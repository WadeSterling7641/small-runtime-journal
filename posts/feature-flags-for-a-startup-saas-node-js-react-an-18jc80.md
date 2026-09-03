# Feature flags for a startup SaaS: Node.js, React, and cohorts you can reconstruct

Pick the feature flag layer by what it lets you reconstruct after an incident, not by what it costs to switch on. A marketplace SaaS running one experiment across tenant cohorts will eventually get a ticket that says "checkout broke for our sellers on Tuesday afternoon" — and the only question that matters then is whether the system can say which cohort saw which variant, at which ruleset version, in the Node.js service and in the React app that rendered the broken page.

| Flag layer shape | Fits when | Incident reconstruction cost | Examples of this shape |
| --- | --- | --- | --- |
| Config rows in your own database | One service, a handful of kill switches, no cohort logic | You write and retain the exposure record yourself | homegrown |
| Self-hosted control plane, evaluation local to the process | The ruleset and the exposure stream should sit inside your own trust boundary | Yours to ship, join and keep; also yours to operate | Unleash, Flagsmith, GrowthBook |
| Managed platform with streaming updates to SDKs | Propagation measured in seconds matters more than owning the box | Evaluation history lives in the vendor; export it before the incident, not during | LaunchDarkly |
| Flags bundled with product analytics | Exposure and cohort analysis should answer from one query surface | Reconstruction is a query — bounded by that store's retention | PostHog, GrowthBook |

Row two is where I'd start for a one-person SaaS with tenant cohorts. Local evaluation keeps the flag check off the network path of a request, the ruleset stays next to the tenant data it targets, and an unreachable control plane degrades into a stale decision instead of a failed one. The catch is that you now own a service and a database you did not have last week. Weigh that against a release train you have to keep moving.

That trade is not free either way.

## How should a startup SaaS compare a feature flag experiment across tenant cohorts in Node.js and React?

Compare the exposure record, not the dashboard. An exposure record is the row you write every time a flag decision is handed to real traffic: flag key, variant, cohort, ruleset version, timestamp, service, and the trace id of the request that triggered it. Without the ruleset version you cannot separate "the experiment did this" from "the deploy did this," because both changed inside the same hour and both are invisible in a metrics chart that only knows error counts. With it, the reconstruction is a join: take the error events from the bad window, join to exposures on trace id, group by cohort and variant, and see whether the treatment cohort is carrying the failures alone.

W3C Trace Context is what makes that join cheap. The `traceparent` header already crosses the browser-to-Node.js boundary in most instrumented stacks, so the exposure record and the error report can share an id without you inventing a correlation scheme. Reuse it.

The React side has its own rule, and I hold it hard: the browser receives a decision, never the credentials or the ruleset needed to make one. Evaluate on the server, put `{ variant, cohort, rulesetVersion }` into the page payload or the API response, and let the client attach those three values as tags on anything it reports. Client-side evaluation also means the flag config is public — a targeting rule that names an enterprise tenant is a leak, not a feature.

A vendor-neutral evaluation API helps here more than it looks like it should. OpenFeature defines a provider interface and an evaluation context, so the shape of `evaluate(flagKey, context)` in your code stops being a property of whichever vendor you signed with. When you swap, you write a provider, not a migration.

## Cardinality is the second criterion, and it bites in the metrics pipeline

OpenTelemetry's metrics model creates one time series per unique attribute set. That is the whole reason to be careful. A counter of flag exposures with attributes `{flag, variant, cohort, stale}` is bounded and cheap; the same counter with `tenant_id` on a marketplace with thousands of sellers is an unbounded series count, and it will show up as a storage bill or a dropped-metrics warning long before it shows up as insight.

So bucket deliberately. Hash the tenant id into a fixed number of cohorts, put the cohort on the metric, and keep the raw tenant id on spans, logs and exposure events where high cardinality is priced differently. You lose per-tenant metric queries. You keep the ability to answer per-tenant questions from the trace and event side, which is where an incident investigation actually happens.

Error grouping deserves the same thought. Most trackers group events by a fingerprint derived from the stack trace, and Sentry documents a custom fingerprint override for exactly this kind of case. Adding the variant to the fingerprint splits one issue into two, so a treatment-only failure stops hiding inside the control group's baseline. The cost is that you now have two issues where you had one, your historical continuity breaks at the moment you change the rule, and every alert threshold tuned to the old issue is wrong. I do it only for the flag under active experiment, and undo it when the experiment ends. Your mileage may vary — a team with a mature alerting policy might reasonably prefer a tag and a saved search over a fingerprint change.

## One evaluation wrapper in Node.js, and React gets a decision not a key

Everything above collapses into one function. It reads a locally cached snapshot, derives the cohort, records a bounded counter and a duration histogram, and returns a decision object that carries its own provenance.

```ts
import { metrics } from "@opentelemetry/api";
import { createHash } from "node:crypto";

const meter = metrics.getMeter("flags");
const exposures = meter.createCounter("flag.exposures");
const evalDuration = meter.createHistogram("flag.evaluation.duration");

const COHORTS = 16;
const STALE_AFTER_MS = 60_000;

export type Variant = "control" | "treatment";

export interface Snapshot {
  version: string;                                              // ruleset id the control plane served
  fetchedAt: number;
  rules: Record<string, { treatmentCohorts: string[] } | undefined>;
}

export interface Decision {
  variant: Variant;
  cohort: string;                                               // "c00".."c15" — bounded on purpose
  rulesetVersion: string;
  stale: boolean;
}

export function cohortOf(tenantId: string, flagKey: string): string {
  const digest = createHash("sha256").update(`${flagKey}:${tenantId}`).digest();
  return "c" + String(digest.readUInt16BE(0) % COHORTS).padStart(2, "0");
}

export function evaluate(snapshot: Snapshot, flagKey: string, tenantId: string): Decision {
  const started = performance.now();
  const cohort = cohortOf(tenantId, flagKey);
  const rule = snapshot.rules[flagKey];
  const decision: Decision = {
    variant: rule?.treatmentCohorts.includes(cohort) ? "treatment" : "control",
    cohort,
    rulesetVersion: snapshot.version,
    stale: Date.now() - snapshot.fetchedAt > STALE_AFTER_MS,
  };

  // tenantId stays off the metric attributes; it belongs on the span and the exposure event.
  exposures.add(1, { flag: flagKey, variant: decision.variant, cohort, stale: decision.stale });
  evalDuration.record(performance.now() - started, { flag: flagKey });
  return decision;
}

export function reportTags(d: Decision): Record<string, string> {
  return { "flag.variant": d.variant, "flag.cohort": d.cohort, "flag.ruleset": d.rulesetVersion };
}
```

Three details carry the weight. The cohort is a hash of the tenant id, so the same seller lands in the same bucket on every process without coordination, and the bucket count is fixed at 16 — a number chosen because it survives being an attribute on every counter in the system. The `stale` flag is honesty about the cache: when the control plane has not been polled inside 60 seconds the decision is still served, still recorded, and marked, so a post-incident query can tell "the rollback had not arrived yet" apart from "the rollback did not work." And `reportTags` is the only thing the React layer ever sees of the flag system, handed down through the API response and attached to client error reports.

Then wire the same three values into your deploy annotations. A flag change that is not on the same timeline as your releases will be blamed for a release, or vice versa, and you will spend an hour proving it.

## When the runner-up shape wins

A managed platform with streaming updates is the better call when propagation time is a safety property rather than a convenience. If a bad rule can cost money per second — a payments path, a pricing experiment, a marketplace payout — then a polled snapshot with a 60-second staleness window is the wrong control, and building your own push channel is not a good use of a week you owe to shipping. Stick with the managed shape when the audit history has a compliance owner, too. Reproducing "who changed what, approved by whom, at what time" is a small feature to buy and an annoying one to build.

The analytics-bundled shape wins in the opposite corner: when the point of the flag is the experiment, and you'd otherwise be hand-rolling exposure joins and significance math. It's not a good fit when your incident window is longer than the retention on that store, or when the flag also serves as an operational kill switch — a product analytics pipeline optimised for daily aggregates is not designed for a decision path that has to answer in milliseconds.

I'm not sure any of this generalises past the small-team case that shaped it. What does generalise: whatever layer you choose, write the exposure record, bound the metric attributes, and keep the ruleset version next to the variant. Vendors get replaced. The timeline is what you'll still be reading next year.

## References

- OpenTelemetry: Metrics signal concepts — https://opentelemetry.io/docs/concepts/signals/metrics/
- Sentry: event grouping and fingerprint mechanics — https://docs.sentry.io/concepts/data-management/event-grouping/
- OpenFeature specification — https://openfeature.dev/specification/
- W3C Trace Context — https://www.w3.org/TR/trace-context/
