# 04 — Loop patterns: iterating without N+1 or data loss

**Problem**: Loops in agentic workflows silently drop items, produce N+1 API calls, or pass incomplete results to downstream steps because the loop hasn't finished yet.

**Why it fails silently**: A loop that processes 10 items looks the same in logs as one that processes 9 — the missing item has no error, just an absence. Downstream steps that read loop output before completion get partial data with no warning.

---

## The loop completion trap

The most common loop mistake: a downstream step reads loop results before the loop has finished.

```yaml
# Wrong: downstream step starts before loop finishes
steps:
  - id: enrich-companies
    type: loop
    over: "{{input.companies}}"
    step: enrich-each

  - id: generate-report   # starts immediately, reads partial results
    type: ai-action
    input: "{{steps.enrich-companies.results}}"
```

In async execution, `generate-report` may start with 3 of 10 companies enriched. The report is incomplete. No error is raised.

---

## Anti-pattern

```yaml
# Wrong: no loop completion gate
- id: process-items
  type: loop
  over: "{{steps.fetch.items}}"
  step: process-each

- id: summarize   # may run with 0 items if loop is still in flight
  type: ai-action
  prompt: "Summarize these results: {{steps.process-items.outputs}}"
```

---

## Correct pattern

Add a `loop_completion` entry condition on every step that consumes loop output:

```yaml
- id: process-items
  type: loop
  over: "{{steps.fetch.items}}"
  step: process-each

- id: summarize
  type: ai-action
  entryConditions:
    onCriteriaFail: "wait"          # block until condition is met
    conditionText: "Wait for all processing to complete"
    criteria:
      - type: loop_completion
        stepId: process-items       # which loop to wait for
        operator: "=="
        value: true
  prompt: "Summarize these results: {{steps.process-items.outputs}}"
```

`onCriteriaFail: "wait"` blocks this step until all loop iterations finish. The step then runs once with the complete output.

---

## Pairing loop results back to source records

After a loop that calls an external API or runs an AI step per item, you often need to pair each result back to the original record for a KG or CRM write.

The problem: loop outputs are indexed by iteration order, not by the original record's ID.

```javascript
// Code step: pair loop outputs with source records
const sourceItems = input.sourceItems;        // original array
const loopOutputs = input.loopOutputs;        // same-length array of results

return sourceItems.map((item, index) => ({
  ...item,                                     // original fields
  ...loopOutputs[index],                       // enriched fields
  sourceId: item.id,                           // explicit ID link
}));
```

Place this code step after the loop completion gate, before the write step.

---

## N+1: when to loop vs when to batch

A loop that calls an LLM or enrichment API once per item is an N+1 pattern. For 100 items: 100 API calls, 100 credit charges, 100× the latency.

**Ask: does the API support batch input?**

```yaml
# Wrong (N+1): one LLM call per item
- id: classify-each
  type: loop
  over: "{{input.emails}}"
  step:
    type: ai-action
    prompt: "Classify this email: {{currentItem.body}}"

# Correct (batch): one LLM call for all items
- id: classify-all
  type: ai-action
  prompt: |
    Classify each of these emails. Return a JSON array in the same order.
    Emails: {{input.emails}}
  responseStructure:
    classifications: "array of { id: string, category: string, priority: string }"
```

Not every step supports batching — enrichment APIs often don't. But AI steps almost always do. Default to batch for AI classification, extraction, and scoring over lists.

---

## Fire-and-forget anti-pattern

```yaml
# Wrong: loop dispatches child workflows with no completion tracking
- id: dispatch-scoring
  type: loop
  over: "{{input.candidates}}"
  step:
    type: call-workflow
    workflowId: score-candidate
    input: "{{currentItem}}"

- id: aggregate-scores   # starts immediately — child workflows haven't finished
  type: ai-action
  prompt: "Aggregate these scores: {{steps.dispatch-scoring.outputs}}"
```

When the loop calls child workflows, completion tracking is especially important — child workflow execution time varies. Always add a `loop_completion` gate before aggregating.

---

## One-line rule

> Always gate the step that consumes loop output on `loop_completion` with `onCriteriaFail: "wait"` — loops run asynchronously and downstream steps will read partial data without it.
