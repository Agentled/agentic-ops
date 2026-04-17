# 05 — Child workflow contracts: composable workflows with typed returns

**Problem**: Monolithic workflows become unmaintainable, and child workflows called from orchestrators fail silently because their return contracts aren't defined — the calling workflow gets `undefined` for every field it tries to read.

**Why it fails silently**: A child workflow that ends with a `milestone` step instead of a `return` step completes successfully from the platform's perspective. The calling workflow receives no data and no error. Every field reference like `{{steps.call-child.score}}` resolves to empty string.

---

## The milestone vs return mistake

```yaml
# Wrong: child workflow ends with milestone
- id: score-company
  type: ai-action
  prompt: "Score this company 0-100..."
  responseStructure:
    score: "number"
    decision: "string"

- id: done           # ← milestone = terminal, no data returned to caller
  type: milestone
  name: "Complete"
```

The calling orchestrator runs `call-workflow` and gets back nothing. `{{steps.call-child.score}}` is empty. No error is raised.

---

## Anti-pattern

```yaml
# Wrong: child workflow
steps:
  - id: enrich
    ...
  - id: score
    ...
  - id: done           # milestone doesn't return data
    type: milestone

# Calling orchestrator:
- id: call-child
  type: call-workflow
  input: { companyUrl: "{{input.url}}" }

- id: use-result
  type: ai-action
  prompt: "Based on score {{steps.call-child.score}}..."
  # ^ always empty — milestone returned nothing
```

---

## Correct pattern

Child workflows must end with a `return` step that explicitly declares what they return:

```yaml
# Correct: child workflow
steps:
  - id: enrich
    ...
  - id: score
    type: ai-action
    responseStructure:
      score: "number 0-100"
      decision: "invest | pass | monitor"
      summary: "string"

  - id: return-results    # ← return step, not milestone
    type: return
    returnConfig:
      fields:
        - name: score          # the name the caller uses
          stepId: score        # which step produced it
          field: score         # which field from that step
        - name: decision
          stepId: score
          field: decision
        - name: summary
          stepId: score
          field: summary

# Calling orchestrator:
- id: call-child
  type: call-workflow
  input: { companyUrl: "{{input.url}}" }

- id: use-result
  type: ai-action
  prompt: "Based on score {{steps.call-child.score}}, decision: {{steps.call-child.decision}}..."
  # ^ now populated correctly
```

---

## Designing a return contract

A return contract is the interface between the child workflow and its callers. Treat it like a function signature:

1. **Be explicit**: list every field the caller might need — don't assume they'll dig into nested objects
2. **Use flat field names**: `score` not `scoringCard.total_score` — callers reference these as template variables
3. **Match names to caller expectations**: if the orchestrator uses `{{steps.call-child.decision}}`, the return contract must export a field named `decision`
4. **Version changes carefully**: adding fields is safe; renaming or removing fields breaks every orchestrator that calls this child

```yaml
# Comprehensive return contract
returnConfig:
  fields:
    - { name: score,           stepId: score-step,   field: total_score }
    - { name: decision,        stepId: score-step,   field: decision }
    - { name: summary,         stepId: score-step,   field: executive_summary }
    - { name: teamEvaluations, stepId: eval-team,    field: evaluations }
    - { name: rawData,         stepId: enrich,       field: companyProfile }
```

---

## The god-workflow anti-pattern

A single workflow with 25+ steps that handles intake, enrichment, scoring, routing, outreach, and CRM sync:

```
trigger → fetch → enrich → score → route → draft-email → approve → send → 
update-crm → update-kg → notify-slack → generate-report → archive → done
```

Problems:
- A failure in step 12 requires re-running steps 1-11
- You can't reuse the scoring logic in another context
- Testing requires running the entire pipeline end-to-end
- A single developer change can break the entire flow

**Break it into composable child workflows:**

```
Orchestrator:
  trigger → call: enrich-workflow → call: score-workflow → call: route-workflow → done

enrich-workflow:    fetch → enrich → return { profile }
score-workflow:     receive profile → score → return { score, decision }
route-workflow:     receive decision → route → draft → approve → send → return { sent }
```

Each child workflow can be tested independently, retried independently, and reused by other orchestrators.

---

## One-line rule

> Child workflows must end with a `return` step (not `milestone`) with an explicit `returnConfig.fields` list — milestone completes silently with no data returned to the caller.
