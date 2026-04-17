# 07 — Error handling: skip vs stop vs wait

**Problem**: Developers use the same error behavior (`stop` or `skip`) everywhere, causing workflows to either halt on recoverable errors or continue silently with missing data.

**Why it fails silently**: `skip` silently passes empty/null data to downstream steps, which then produce wrong outputs with no error. `stop` halts workflows on optional steps that should have been bypassed. Neither produces a visible failure at the wrong step — the failure surfaces later, in a confusing place.

---

## The three behaviors

Every entry condition has an `onCriteriaFail` setting that determines what happens when the condition isn't met:

| Value | What it does | When to use |
|---|---|---|
| `skip` | Skip this step, pass `null`/empty to downstream steps, continue execution | Optional step — workflow is valid without it |
| `stop` | Halt the entire execution immediately | Hard prerequisite — nothing downstream is meaningful without this step |
| `wait` | Block this step until criteria become true | Async dependency — waiting for a loop or parallel group to finish |

---

## Anti-pattern: stop everywhere

```yaml
# Wrong: using stop for an optional enrichment step
- id: enrich-linkedin
  type: app-action
  entryConditions:
    onCriteriaFail: "stop"       # ← wrong: this is optional
    criteria:
      - variable: "{{input.linkedinUrl}}"
        operator: "isNotNull"
# If linkedinUrl is missing: entire workflow halts.
# But the workflow could run fine without LinkedIn data.
```

```yaml
# Correct: optional enrichment step uses skip
- id: enrich-linkedin
  type: app-action
  entryConditions:
    onCriteriaFail: "skip"       # ← correct: skip if no URL
    conditionText: "Skip LinkedIn enrichment if no URL provided"
    criteria:
      - variable: "{{input.linkedinUrl}}"
        operator: "isNotNull"
```

---

## Anti-pattern: skip for hard prerequisites

```yaml
# Wrong: using skip for required input validation
- id: validate-required-fields
  type: code
  entryConditions:
    onCriteriaFail: "skip"       # ← wrong: nothing downstream works without this
    criteria:
      - variable: "{{input.companyDomain}}"
        operator: "isNotNull"
# If domain is missing: validation is skipped.
# Next step tries to enrich with null domain.
# Enrichment fails with a confusing error 3 steps later.
```

```yaml
# Correct: hard prerequisite uses stop
- id: validate-required-fields
  type: code
  entryConditions:
    onCriteriaFail: "stop"       # ← correct: stop early with a clear failure
    conditionText: "Company domain is required"
    criteria:
      - variable: "{{input.companyDomain}}"
        operator: "isNotNull"
```

Fail fast. A clear stop at the prerequisite is better than a confusing error 5 steps later.

---

## `wait`: blocking on async completion

`wait` is for a specific use case: blocking a step until an async process finishes. The two common cases are loop completion and parallel group completion.

```yaml
# Wait for a loop to finish before consuming its output
- id: aggregate-results
  type: ai-action
  entryConditions:
    onCriteriaFail: "wait"
    conditionText: "Wait for all enrichment loop iterations to complete"
    criteria:
      - type: loop_completion
        stepId: enrich-loop
        operator: "=="
        value: true
  prompt: "Summarize these enriched companies: {{steps.enrich-loop.outputs}}"
```

`wait` is not a general-purpose retry mechanism. It's specifically for dependency ordering in async workflows.

---

## Decision guide

Ask these questions in order:

**1. Is this step required for the workflow to produce a valid output?**
- Yes → `stop`
- No → continue to question 2

**2. Is this step waiting for another step/loop/group to finish?**
- Yes → `wait`
- No → `skip`

Examples:

| Step | Required? | Async wait? | Use |
|---|---|---|---|
| "Validate required domain field" | Yes | No | `stop` |
| "Enrich LinkedIn (optional)" | No | No | `skip` |
| "Send Slack alert (if webhook configured)" | No | No | `skip` |
| "Aggregate loop results" | Yes | Yes (loop) | `wait` |
| "Score company (ICP match required)" | Yes | No | `stop` |
| "Add CRM note (nice to have)" | No | No | `skip` |

---

## Handling downstream null data after skip

When a step is skipped, its outputs are `null` or empty. Downstream steps that reference those outputs must handle nulls gracefully:

```yaml
# Pattern: null-safe reference in AI prompt
prompt: |
  Company: {{steps.enrich.company.name | default: "Unknown"}}
  LinkedIn headline: {{steps.enrich-linkedin.headline | default: "Not available"}}
  Score this company based on what's available.
```

Or use a code step to normalize before passing to downstream AI steps:

```javascript
// Code step: normalize potentially-null enrichment data
return {
  name: input.enrichData?.name ?? input.rawInput.companyName,
  industry: input.enrichData?.industry ?? "Unknown",
  employees: input.enrichData?.employees ?? null,
};
```

---

## One-line rule

> Use `skip` for optional steps, `stop` for hard prerequisites, and `wait` for async dependencies — and always handle `null` outputs from skipped steps in downstream steps.
