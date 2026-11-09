# Node.js Web App SMS Alerts: Approval Authority Before Batch Notification Release

A nonprofit volunteer marketplace needs to change notification wording without changing which support queue receives a contact form. That constraint changes the selection process. Decide who can approve a template, and what an approval authorizes, before choosing a simple SMS notification service for the web app.

Short answer: keep contact routing and template release authority in the application. For US and EU batch alerts without webhooks, require evidence that the transport supports retrievable message status and the suppression information the workflow needs. Compare interfaces on how they preserve an approved revision through submission and review. Keep the support queue usable while an alert's delivery outcome remains unknown.

This build log describes a proposed first release, not a production incident. The governing question is template ownership: can the nonprofit change a sentence without giving that editor authority over destinations, recipients, or pending batches? The practical economic lens for a solo SaaS operator is revenue per hour. Every manual approval reconciliation competes with the next feature.

## Who approves SMS notification templates for web app batch alerts?

Use a deliberately small design: 2 support queues, scheduling and safeguarding, serving US and EU coordinators. Those are example inputs, not measured capacity. A contact form supplies a validated reason. Application policy maps that reason to a queue. The nonprofit's designated reviewer approves notification wording for that queue and region; the application owner releases the approved revision.

Separate those permissions.

The constraint that changes the choice is pending work. Suppose a scheduling alert batch references revision 3 when a reviewer approves revision 4. This design keeps the pending batch on revision 3 until someone explicitly cancels or replaces it. Otherwise, an ordinary copy edit becomes an unreviewed change to messages already authorized for dispatch. A remote template editor may still be useful, but require a documented way to bind the approved content to the submitted message. An editable template name alone is insufficient evidence for that requirement.

Write down the approval boundary before comparing services:

| Decision | Proposed owner | Evidence retained |
| --- | --- | --- |
| Map a contact reason to a support queue | Application owner | Routing policy revision |
| Approve queue notice wording | Nonprofit reviewer | Exact text, queue, region, revision |
| Release a batch | Application owner | Approved revision and recipient scope |
| Withdraw pending content | Authorized reviewer and operator | Cancellation decision and affected dispatches |

This adds release work. The payoff is a reviewable boundary around copy changes. To ship weekly, keep that boundary narrow: reviewers edit the notice, while routing and recipient permissions follow their own review path. Outsource the undifferentiated transport work once the interface can preserve these decisions.

Safeguarding also changes the content policy. For this design, the SMS says that a request needs review; it does not copy the contact form's free text. Staff read the report inside the authorized application. Approval for scheduling notices grants no authority to receive safeguarding notices, and a region label grants no queue access.

## Build a release manifest, then evaluate the transport

The smallest working component is a release check. It receives the queue chosen by trusted routing policy and rejects a batch whose content approval belongs to another queue or region. It also requires the release to carry the exact approved text. This is a local TypeScript example for a Node.js application, with no assumed commercial API.

```ts
type Queue = "scheduling" | "safeguarding";
type Region = "US" | "EU";

type Approval = Readonly<{
  id: string;
  queue: Queue;
  region: Region;
  revision: number;
  body: string;
  state: "approved" | "withdrawn";
}>;

type Release = Readonly<{
  batchId: string;
  queue: Queue;
  region: Region;
  approvalId: string;
  revision: number;
  body: string;
}>;

function checkRelease(release: Release, approval: Approval): void {
  if (
    approval.state !== "approved" ||
    release.approvalId !== approval.id ||
    release.queue !== approval.queue ||
    release.region !== approval.region ||
    release.revision !== approval.revision ||
    release.body !== approval.body
  ) {
    throw new Error("Release does not match the approved content scope");
  }
}

const approval: Approval = {
  id: "scheduling-eu-r3",
  queue: "scheduling",
  region: "EU",
  revision: 3,
  body: "A scheduling request needs review.",
  state: "approved",
};

const release: Release = {
  batchId: "volunteer-contact-42",
  queue: "scheduling",
  region: "EU",
  approvalId: approval.id,
  revision: approval.revision,
  body: approval.body,
};

checkRelease(release, approval);
```

The check passes for this example. Changing the release's queue to safeguarding makes it throw. So does changing the body while retaining the approval identifier. Test those cases along with a withdrawn approval and the wrong region. These are useful acceptance tests because they exercise the permission boundary, not just the happy path.

The function checks consistency. It does not authenticate a reviewer, persist an approval, or validate untrusted JSON at runtime. Load approvals from trusted storage and authorize the release caller separately. Treat the public form as input to validation and routing, never as the source of an approval record. Persist the contact's queue and dispatch intent together when the storage transaction supports it, then let a worker process the intent.

Now the service comparison has a concrete test. If an interface accepts message text, check that the worker submits the pinned body. If it requires a remote template identifier, demand evidence of how that identifier resolves to the approved revision. Reject a design that cannot establish what text a pending batch will use. Neither interface shape removes the need for an owner.

No ranking is needed. The relevant distinction is how much template synchronization and review the application must own.

## Make no-webhook operation part of the release review

Template approval is only one release condition. For a batch alert, record an individual dispatch intent for each eligible coordinator. At submission time, recheck queue access, current notification permission, suppression state, and whether the approval has been withdrawn. A recipient list created earlier must not freeze eligibility indefinitely. Retain a stable local dispatch key, but don't assume that the key makes a remote submission idempotent.

Ask for evidence.

For each candidate interface, verify status lookup by the returned message identifier, the meaning of each status, lookup retention, request limits, and support for the actual US and EU destinations. These are selection requirements, not capabilities established by the references below. Submission acceptance must remain distinct from confirmed delivery in the application. Aggregate the batch from its recipient records so a successful submission for one coordinator cannot hide an unresolved result for another.

A no-webhook requirement also needs a suppression retrieval story. Determine how the application learns about a transport-side opt-out and how quickly that information can affect pending work. A local preference flag cannot represent an external decision the application has never received. If documented retrieval cannot satisfy the workflow, revisit the no-webhook constraint before release. Do not quietly treat missing information as permission.

Keep retry decisions specific. A failed status lookup can leave the last observation intact while a later check is scheduled. A send timeout leaves uncertainty about whether submission happened; retrying with a fresh request may duplicate the alert. Evaluate documented idempotency or reconciliation behavior before enabling automatic resubmission. Bound polling by the documented limits and an explicit observation deadline. Record unresolved outcomes as unknown.

There is also a boundary between checking eligibility locally and submitting remotely. Define when cancellation takes effect and retain the decision state used for each submission. Don't promise an atomic operation across independent systems.

The release review should include one suppressed coordinator, one pending dispatch whose approval is withdrawn, and one ambiguous submission. These are proposed test cases, not reported incidents. Check that each leaves a useful support record and an explainable delivery record. For regional eligibility, assign someone to review the applicable messaging requirements. The FTC guide concerns commercial email; it cannot establish SMS eligibility for volunteers in either region. Likewise, email documentation cannot establish SMS coverage or status behavior.

## What I would change at scale

For a larger deployment, I would separate copy review from batch release in the operator interface and introduce explicit cancellation of pending dispatches by approval revision. This is a proposed next step. It adds administration, so defer it until the volume of approvals makes the simpler workflow hard to review.

Keep the operational view small: pending age, unresolved delivery outcomes, suppressed dispatches, and the approval revision attached to each batch. Leave contact-form bodies out of routine transport logs. Release a controlled batch before expanding dispatch, and verify that workers preserve the approved scope under concurrency. A faster worker must not acquire broader permissions.

The trade-off is deliberate application ownership of content policy in exchange for a smaller transport boundary. Evaluate operating effort across review, polling, suppression reconciliation, and ambiguous sends. The support queue should remain useful throughout. That is the release criterion for volunteer coordination: the right people can find the work, and every outgoing notice has an accountable owner.

## References

The sources below establish the distinction between email documentation and commercial-email guidance. They do not verify SMS service features, regional coverage, or volunteer messaging eligibility. Those remain evidence requirements for the transport evaluation described here; the release manifest and ownership rules are proposed application design.

## Sources

- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://resend.com/docs/introduction
