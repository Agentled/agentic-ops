# Anti-pattern — Fire-and-forget: async steps with no completion signal

**The shape**: A workflow kicks off an async operation (long-running enrichment, browser-use task, child workflow, external job) and continues to the next step immediately. Downstream steps assume the async operation finished. Sometimes it has. Often it hasn't.

**Why it's tempting**: Async APIs return quickly. The step "succeeded" — it dispatched the request. The orchestrator marks the step green. The next step runs. Everything looks fast and clean. The bug is that "dispatched" is not "done".

---

## The classic broken shape

```yaml
- id: kick-off-enrichment
  action: enrichment.start
  input: { entity_id: "{{currentItem.id}}" }
  # Returns immediately with { task_id: "abc" }

- id: use-enriched-data
  type: aiAction
  prompt: |
    Use this enriched data: {{steps.kick-off-enrichment.result}}
    # ❌ result doesn't exist yet — only task_id was returned
```

The AI step receives `undefined` or an empty object. The model produces nonsense from nothing — no error, just plausible-looking output disconnected from real data.

---

## Three ways this fails silently

1. **Empty input becomes "no result found"** — The downstream LLM dutifully reports "no information available" because that's all the prompt told it.
2. **Race condition: sometimes works** — The async job is fast enough on small inputs that the next step gets results sometimes. The bug only fires under load.
3. **Forwarded `task_id` flows downstream** — The next step gets the task ID instead of the data, treats it as content, hallucinates around it.

The common thread: no error is raised at the step boundary. The execution is "successful" on the dispatch, and downstream code can't tell the difference between "completed empty" and "still running".

---

## Correct pattern: poll until done, or wait for callback

### Synchronous wait (small jobs, < 60s)

```yaml
- id: enrichment
  action: enrichment.run_and_wait    # the action itself blocks until the job completes
  input: { entity_id: "{{currentItem.id}}" }
  timeout: 60s

- id: use-enriched-data
  type: aiAction
  prompt: "Use this: {{steps.enrichment.result}}"   # safe — completed before this step runs
```

When the underlying API supports a synchronous "do it and return" call, prefer that over fire-and-forget. The complexity of polling/callbacks isn't free.

### Polling with explicit completion gate

```yaml
- id: kick-off-enrichment
  action: enrichment.start
  input: { entity_id: "{{currentItem.id}}" }

- id: wait-for-enrichment
  type: poll
  action: enrichment.status
  input: { task_id: "{{steps.kick-off-enrichment.task_id}}" }
  until: "{{response.status == 'completed'}}"
  every: 5s
  timeout: 5m
  on_timeout: stop

- id: fetch-result
  action: enrichment.get_result
  input: { task_id: "{{steps.kick-off-enrichment.task_id}}" }

- id: use-enriched-data
  type: aiAction
  prompt: "Use this: {{steps.fetch-result.result}}"
```

Three explicit steps, but each has a clear contract. The poll step blocks the workflow on the *real* completion event, not on the dispatch.

### Callback / webhook resume

```yaml
- id: kick-off-enrichment
  action: enrichment.start
  input:
    entity_id: "{{currentItem.id}}"
    callback_url: "{{workflow.resume_url}}"

- id: wait-for-callback
  type: pause
  resume_on:
    webhook: "{{trigger.callback_token}}"
    timeout: 30m
    on_timeout: stop

- id: use-enriched-data
  prompt: "Use this: {{steps.wait-for-callback.payload}}"
```

For long-running jobs (hours), polling burns credits — use a callback to resume. The orchestrator must support pause/resume primitives; if it doesn't, polling is the fallback.

---

## Loop-of-async: the worst case

```yaml
loop:
  over: "{{items}}"
  steps:
    - kick-off-enrichment           # fires, doesn't wait
    - use-result                    # uses empty data
```

100 items, 100 fires, 100 immediate "completions", 100 wrong results. And worse: the loop "finishes" before any of the async jobs are done — the workflow's final step runs while the enrichments are still in flight. The final output is from data that doesn't exist yet.

**Fix**: complete each iteration before starting the next, OR fire all N then wait for all N to complete before consuming results (see v1 pattern 04 for `loop_completion` gates).

---

## Detection signals

| Signal | Likely cause |
|---|---|
| Downstream step receives an ID instead of data | Forgot to fetch result after dispatch |
| Same workflow produces different outputs on the same input | Race condition on async completion |
| Final step references "no data found" despite earlier dispatch | Async didn't complete before final step |
| External system shows jobs still queued after workflow "succeeded" | Workflow exited before async work finished |

A simple test: run the same workflow 10 times on the same input. If outputs vary in shape (sometimes full data, sometimes empty), you have a fire-and-forget race.

---

## When fire-and-forget is actually correct

Fire-and-forget is the right shape only when:

- The downstream workflow doesn't need the result
- The async operation is genuinely a side effect (notification, log entry, background reindex)
- The result will be consumed by a *different* workflow at a later time

If you can list any downstream step that reads the result, it's not fire-and-forget — it's poll-and-consume, and you have to write the polling.

---

## One-line rule

> Async dispatch is not async completion; every workflow step that consumes async results must wait on the completion event (poll, callback, or sync wrapper), not on the dispatch.
