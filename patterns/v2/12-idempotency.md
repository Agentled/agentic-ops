# 12 — Idempotency: making retries safe at every level

**Problem**: A workflow retries a failed step and the side effects fire twice. An email gets sent twice, a row gets inserted twice, a payment gets charged twice. The "fix" — turning off retries — replaces silent duplication with silent loss.

**Why it fails silently**: Most orchestrators offer "retry on failure" as a checkbox. The checkbox doesn't ask whether your step is safe to retry — it assumes you've made it safe. If the step calls `POST /charges` without an idempotency key, the second retry charges the customer a second time and the API returns 200 OK both times. There's no error to see.

---

## Idempotency is a per-level concern

Every level of the stack needs its own dedup key:

| Level | Dedup key | Purpose |
|---|---|---|
| Trigger | `event_id` or `message_id` | Prevent re-delivery from re-processing |
| Step | Step-level `idempotency_key` | Prevent retry from double-firing the side effect |
| External API | API-provided idempotency key | Prevent network retry / duplicate request from charging twice |
| Sink (DB / CRM) | `unique_constraint` or `userKey` | Prevent two concurrent runs from inserting two rows |

You need all four, not one. A workflow that has trigger dedup but no step idempotency double-charges the customer if the orchestrator retries the charge step. A workflow with step idempotency but no DB unique constraint duplicates rows when two operators rerun the same execution.

---

## Anti-pattern

Trust the retry button:

```yaml
- id: charge-customer
  action: stripe.charge
  retry:
    on_failure: retry
    max_attempts: 3
  input:
    amount: 5000
    customer: "{{steps.lookup.customer_id}}"
```

The first attempt times out at the network layer but succeeds at Stripe. The retry fires. Customer gets charged twice. The execution log shows "succeeded on attempt 2" — perfectly green.

---

## Correct pattern

### 1. Always pass an idempotency key to external APIs that support them

```yaml
- id: charge-customer
  action: stripe.charge
  input:
    amount: 5000
    customer: "{{steps.lookup.customer_id}}"
    idempotency_key: "{{execution.id}}-charge-{{steps.lookup.invoice_id}}"
  retry:
    on_failure: retry
    max_attempts: 3
```

Stripe (and most modern APIs: Stripe, Anthropic, OpenAI, GitHub, Plaid, Twilio) accept an idempotency header. With a stable key, the second call returns the result of the first instead of charging again.

The key must be:
- **Stable across retries** of the same step (don't include a timestamp or `Math.random()`)
- **Unique per logical operation** (so different invoices for the same customer don't collide)

`{{execution.id}}-charge-{{invoice_id}}` is a good shape — execution-scoped, semantically distinct.

### 2. Make sinks idempotent at the storage layer

Even with API-level keys, two concurrent executions can race. Defend at the storage layer:

```sql
-- Unique constraint on the natural key
CREATE UNIQUE INDEX idx_outreach_dedup
  ON outreach_log (recipient_email, campaign_id, sent_date);
```

The second insert fails on the constraint. The workflow can catch the conflict and treat it as a no-op success.

For document stores or KV (DynamoDB, Mongo, Firestore), use `userKey`-style upserts: a caller-supplied stable id that maps 1:1 to the logical record. Same key in = same row updated, never a new row inserted.

### 3. Side-effect steps must be the last write of their kind in a chain

```
❌ Bad:  enrich → write-row → score → write-row-with-score → send-email → write-final-row
✅ Good: enrich → score → compose-email → write-row (one write, one place)
```

Three writes mean three places to dedup. One write means one. If you must write multiple times, write to a single row that's idempotently keyed and `merge` the new fields.

### 4. Defend against trigger re-delivery

Webhooks and event triggers will redeliver. Always:

```yaml
- id: dedup-event
  store:
    table: processed_events
    key: "{{trigger.event_id}}"
    on_conflict: stop_workflow   # already processed, exit
  next: process-event
```

The first invocation inserts. The redelivery hits the unique constraint and the workflow exits cleanly. No duplicate processing.

---

## Idempotency keys for AI steps

LLM calls are not free, but they are *idempotent in business terms* — calling them twice with the same prompt produces equivalent semantic output (not byte-identical, but interchangeable). The cost is wasted credits, not duplicate side effects.

For AI steps, the idempotency boundary is around the **side effect that consumes the AI output**, not the AI call itself. Don't over-engineer dedup for the AI step; over-engineer it for the email-send step that follows.

Exception: if the AI call writes to memory or a knowledge graph, treat it as a side effect and pass a stable key on the write step.

---

## Anti-pattern: timestamp in the idempotency key

```yaml
idempotency_key: "charge-{{now}}"
```

Every retry has a different `now`. The key changes. Stripe sees each retry as a new charge. Customer is charged 3 times.

**Idempotency keys must be deterministic across retries of the same logical operation.** Build them from execution_id, step_id, and inputs — never from time.

---

## Anti-pattern: idempotency by "checking first"

```yaml
- id: check-existing
  query: "SELECT id FROM outreach WHERE recipient = $1"
- id: maybe-send
  if: "{{steps.check-existing.empty}}"
  action: send-email
```

Two concurrent executions both pass the check (database is empty), both insert, both send. The check-then-act pattern is a race condition every time concurrency exists.

**Always rely on a unique constraint or atomic upsert.** "Check first" is not idempotency.

---

## Detection: how do you know it worked?

Test by deliberately retrying:

1. Run the workflow once, note the side effects (rows, emails, charges).
2. Manually retry the side-effect step.
3. The side effects must not multiply — exactly the same end state as after run 1.

If retry produces 2× rows, the dedup is broken at the storage layer. If retry produces 2× emails, the API key isn't being passed. If retry produces a fresh charge, the idempotency key is changing across attempts.

---

## One-line rule

> Every step that produces an external side effect needs a deterministic idempotency key — derived from execution and inputs, never from time — and every sink needs a unique constraint as the last line of defense.
