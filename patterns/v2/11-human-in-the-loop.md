# 11 — Human-in-the-loop: approval gates that don't stall the workflow

**Problem**: Workflows that need human review before an irreversible action (sending an email, creating a record, posting publicly) either skip the gate entirely or block forever waiting on a reviewer who never sees the request.

**Why it fails silently**: An approval gate without a notification is just a workflow stuck in "pending review" — no error, no timeout, no escalation. A week later a teammate asks "did this run?" and the answer is "yes, it's been waiting for you to click approve for six days". Without an explicit timeout and escalation path, the gate becomes a permanent stall.

---

## The four parts of a working approval gate

1. **The gate** — the workflow stops at this step until a human takes action.
2. **The notification** — the reviewer is told there's something to review, in a channel they actually read.
3. **The review surface** — the reviewer can see what's being approved (preview, not raw JSON).
4. **The timeout + escalation** — if no one acts within X, do something (escalate, auto-approve, auto-reject, or ping a backup reviewer).

Skipping any one breaks the loop:

| Skipped | Failure mode |
|---|---|
| Notification | Reviewer never knows; workflow stalls indefinitely |
| Review surface | Reviewer sees raw JSON, can't make a decision, ignores it |
| Timeout | One reviewer on vacation = pipeline frozen for two weeks |
| Escalation | Critical approvals miss SLA; nothing prompts a re-route |

---

## Anti-pattern

```yaml
- id: send-email
  type: approval_gate
  message: "Please approve before sending"
  on_approve: send-email-action
  # ❌ no notification — reviewer must remember to check the dashboard
  # ❌ no preview — reviewer sees a step ID and a JSON blob
  # ❌ no timeout — pending forever
```

This is technically a gate. Operationally it's a way to make sure nothing ever ships.

---

## Correct pattern

```yaml
- id: review-outreach
  type: approval_gate
  preview:
    renderer: email           # show the email as it will be sent, not as JSON
    fields:
      from: "{{steps.compose.email.from}}"
      to: "{{steps.compose.email.to}}"
      subject: "{{steps.compose.email.subject}}"
      body: "{{steps.compose.email.body}}"
  notify:
    channel: slack
    target: "#outreach-reviews"
    message: "New outreach to {{steps.enrich.contact.name}} — review: {{review_url}}"
  timeout:
    after: 24h
    on_timeout: escalate
  escalate:
    notify:
      channel: slack
      target: "@head-of-sales"
      message: "Outreach pending >24h, please review or assign"
    extend_by: 24h
    on_second_timeout: auto_reject   # or auto_approve, depending on risk profile
  on_approve:
    next: send-outreach
  on_reject:
    next: log-rejection
```

The shape generalizes. Substitute `slack` for any messaging channel, `email` renderer for any content type, and `auto_reject` for whatever default is safe in your context.

---

## Choosing the right `on_timeout` default

The default action when no one reviews is the most consequential design choice in the gate.

| Action type | Default that's safe | Why |
|---|---|---|
| Send external email | `auto_reject` | Better to miss a touchpoint than send unreviewed |
| Internal CRM update | `auto_approve` | Stale data is worse than imperfect data |
| Post publicly (social, blog) | `auto_reject` | Unreviewed public content is high-risk |
| Spend money (paid ads, refunds) | `auto_reject` | Always |
| Triage / label / categorize | `auto_approve` | Low-risk, stalling is the bigger cost |
| Delete records | `auto_reject` | Always |

**Rule: if the action is irreversible or externally visible, the default on timeout is `auto_reject`.** Approve must be a deliberate act.

---

## Async review without polling

Reviewers shouldn't have to log into a dashboard to act. Two patterns:

### Inline approval link in the notification

The notification contains a signed URL that approves or rejects in one click. The link payload encodes `(execution_id, step_id, decision, signature)` so the workflow can resume without a logged-in session.

```
"Outreach to Acme — review here: https://app.example.com/approve?token=abc..."
```

The token expires when the timeout fires.

### Reply-to-act in chat

In Slack/Teams, the notification renders as a message with Approve / Reject / Edit buttons. The message includes the preview inline. Reviewers act in the channel they're already in.

Both eliminate the "I'll review later" failure mode where "later" never comes.

---

## Anti-pattern: routing approvals to a single named person

```
notify:
  target: "@alice"
```

Alice goes on vacation. The pipeline stops for two weeks.

Always route to a **role or channel**, not a person. Use `@head-of-sales`, `#outreach-reviews`, `@oncall-reviewer` — anything that someone else can pick up when the primary is unavailable.

---

## Anti-pattern: silent revisions

When a reviewer rejects, the workflow should branch to a path that captures **why** — for retraining the prompt, for human follow-up, or for escalation. A bare `on_reject: end` discards the signal.

```yaml
on_reject:
  capture:
    reason: "{{review.rejection_reason}}"   # required field on reject
    reviewer: "{{review.reviewer}}"
  next: store-rejection-feedback
```

Rejection reasons are the highest-quality training data the workflow produces. Don't throw them away.

---

## When to skip the gate entirely

Approval gates have a real cost — every gate adds latency and reviewer load. Skip the gate when:

- The action is reversible **and** cheap to undo (internal data update with audit log)
- The action is bounded by another safety mechanism (rate limit, budget cap, allowlist of recipients)
- The model has been validated on this exact decision over thousands of examples and the false-positive cost is acceptable

Don't gate every AI step "to be safe". A workflow with five gates is a workflow no one runs.

---

## One-line rule

> Every approval gate must have notification, preview, timeout, and escalation; route to a role not a person, and default to the safe action (usually reject) when the timeout fires.
