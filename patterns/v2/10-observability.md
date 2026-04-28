# 10 — Observability: making silent failures loud

**Problem**: Agentic workflows fail in ways that don't show up as errors. A loop iterates over zero items, a conditional skips the only branch that matters, an LLM returns an empty string that downstream steps happily forward. The execution shows green; the outcome is wrong.

**Why it fails silently**: The orchestrator only knows about thrown errors. "Returned no rows", "skipped because of an unmet condition", "produced an empty array" are all valid outcomes from the runtime's perspective. Without explicit instrumentation, these look identical to a successful run that did real work. By the time someone notices the metric is flat, days of executions have produced nothing.

---

## What you actually need to see

For every execution, three layers must be queryable:

1. **Step-level outcome** — did this step run, skip, or fail? If it ran, did it produce data?
2. **Loop and branch counts** — how many iterations, how many branches taken? Zero is a finding, not a default.
3. **Business outcome** — did the workflow do its job (lead created, email sent, row updated)? This is **not** the same as "execution status: success".

The first two are infrastructure concerns. The third is workflow-specific and is the one teams skip. It's also the one that tells you the workflow is silently broken.

---

## Anti-pattern

Trust the green checkmark:

```
execution_id: exec_abc123
status: success
duration: 12.4s
steps:
  - fetch-emails:    ok   (0 messages)
  - process-loop:    ok   (0 iterations)
  - send-summary:    ok   (empty body)
✅ all green
```

The execution succeeded. Zero work was done. No alarm. The dashboard shows "100% success rate" while the inbox piles up.

---

## Correct pattern

### 1. Emit structured signals at every decision point

After every fetch, loop, branch, and sink, emit a structured log line that includes the **count or shape** of what happened. Not "fetched emails" — `fetched_emails count=0 query="-label:processed newer_than:1d"`.

```yaml
- id: fetch-emails
  action: GMAIL_FETCH_EMAILS
  emit:
    - signal: fetched_count
      value: "{{steps.fetch-emails.messages.length}}"

- id: process-loop
  loop: "{{steps.fetch-emails.messages}}"
  emit:
    - signal: iterations
      value: "{{loop.totalIterations}}"
    - signal: errors_in_loop
      value: "{{loop.errorCount}}"

- id: write-to-crm
  emit:
    - signal: rows_created
      value: "{{steps.write-to-crm.created}}"
```

The signal names are stable across runs so they're aggregatable. `fetched_count == 0` for three consecutive runs is an alert, not noise.

### 2. Define business-outcome assertions per workflow

Every workflow has a **success contract** beyond "no exception thrown". State it explicitly:

```yaml
success_contract:
  - assert: rows_created > 0
    when: fetched_count > 0
    on_fail: alert "intake produced no rows despite finding messages"
  - assert: errors_in_loop / iterations < 0.1
    on_fail: alert "loop error rate above 10%"
```

If the assertion can't be expressed in the orchestrator, encode it as a final code step that throws on contract violation. A thrown error is louder than a quiet zero.

### 3. Trace IDs that survive across child workflows

When an orchestrator calls a child workflow, propagate a `trace_id` so the child's logs can be joined to the parent's. Without this, debugging a 5-step orchestrator that calls a 10-step child is searching two separate log streams for matching timestamps.

```yaml
parent:
  - id: call-child
    action: call-workflow
    input:
      workflow_id: scoring
      trace_id: "{{execution.id}}"
      payload: "{{steps.normalize.item}}"

child:
  # every log line in the child workflow tags trace_id from input
```

### 4. Route alerts to the same channel that gets read

Email alerts go to a folder no one opens. Slack alerts to `#alerts-firehose` get muted. Route critical assertions (workflow produced no output, error rate spiked) to the channel the team **already monitors** for incidents — usually the same place pages go.

---

## Detection signals

Add these to a dashboard the team checks weekly:

| Signal | What it means | Action |
|---|---|---|
| `fetched_count == 0` for N consecutive runs | Source is empty, query is wrong, or auth expired | Check upstream, then auth |
| `iterations == 0` after fetch returned items | Loop config wrong (path/field mismatch) | Inspect loop's `over` expression |
| `errors_in_loop / iterations > 10%` | Per-item step is unstable | Add retry or fix root cause |
| `rows_created == 0` over 24h | Sink is broken or condition skipped everything | Check write step's entry conditions |
| Step duration p95 doubled | Upstream API slowing down or rate-limit retry | Inspect external service |

The pattern is the same: **count things, then alert on counts that deviate from the contract**.

---

## Anti-pattern: logging the input but not the count

```
log: "Processing emails: [{id: 1}, {id: 2}, ...]"
```

Fine for one execution. Across 1000 executions you cannot answer "how many runs had zero emails?" because the data is buried in unstructured strings. Log the **count** as a structured field; log the IDs only at debug level.

---

## One-line rule

> Every workflow must declare a business-outcome assertion ("rows_created > 0 when fetched_count > 0") and emit a structured count signal at every fetch, loop, and write — silent zeros are the failure mode, not exceptions.
