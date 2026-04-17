# 06 — Conditional routing: conditions that actually fire

**Problem**: Entry conditions on workflow steps silently fail to apply — steps run unconditionally, or skip unconditionally — because the wrong field names are used to configure them.

**Why it fails silently**: The wrong field names (`conditions` instead of `criteria`, `field` instead of `variable`) are not validation errors. The platform silently ignores unrecognized fields and applies no condition at all. The step runs every time, or skips every time, with no visible indication that the condition isn't being evaluated.

---

## The field name trap

This is the most subtle silent failure in workflow configuration. Two pairs of field names look interchangeable but aren't:

| Wrong | Correct |
|---|---|
| `conditions` | `criteria` |
| `field` | `variable` |

```yaml
# Wrong: unrecognized field names — condition is silently ignored
entryConditions:
  onCriteriaFail: "skip"
  conditions:                    # ← wrong: should be "criteria"
    - field: "{{input.score}}"   # ← wrong: should be "variable"
      operator: ">"
      value: 70

# Correct: recognized field names — condition is evaluated
entryConditions:
  onCriteriaFail: "skip"
  criteria:                      # ← correct
    - variable: "{{input.score}}"  # ← correct
      operator: ">"
      value: 70
```

The wrong version doesn't error. The step runs unconditionally, as if no condition existed.

---

## Anti-pattern

```yaml
# Wrong: routes HOT leads to Slack, but the condition is silently ignored
- id: notify-hot-lead
  type: app-action
  app: slack
  entryConditions:
    onCriteriaFail: "skip"
    conditions:                         # ← wrong field name
      - field: "{{steps.score.category}}"  # ← wrong field name
        operator: "=="
        value: "HOT"
  # Result: every lead triggers a Slack notification, including COLD and WARM
```

---

## Correct pattern

```yaml
# Correct: HOT leads only
- id: notify-hot-lead
  type: app-action
  app: slack
  entryConditions:
    onCriteriaFail: "skip"
    conditionText: "Only notify for HOT leads"
    criteria:                                    # ← correct
      - variable: "{{steps.score.category}}"    # ← correct
        operator: "=="
        value: "HOT"
```

---

## How to verify a condition is actually firing

When a condition behaves unexpectedly (step always runs, or always skips), check:

1. **Field names**: is it `criteria`/`variable`? Not `conditions`/`field`?
2. **Variable path**: does `{{steps.score.category}}` actually resolve? Check the referenced step's output in execution logs.
3. **Value type**: comparing a string `"70"` to a number `70` with `>` may not work as expected — check that types match.
4. **Null safety**: `isNotNull` checks should come before value comparisons to avoid null reference issues.

```yaml
# Pattern: null-safe condition chain
criteria:
  - variable: "{{steps.enrich.company}}"
    operator: "isNotNull"          # check existence first
  - variable: "{{steps.score.total}}"
    operator: ">"
    value: 70                      # then compare value
```

---

## The LLM-as-router anti-pattern

Using an AI step to make a binary routing decision that a simple condition can handle:

```yaml
# Wrong: burning 10 credits to decide yes/no
- id: decide-routing
  type: ai-action
  creditCost: 10
  prompt: |
    Given this lead's score of {{steps.score.value}}, should we send a Slack alert?
    Score above 80 means yes. Below 80 means no.
    Return: { shouldAlert: boolean }

- id: send-alert
  entryConditions:
    criteria:
      - variable: "{{steps.decide-routing.shouldAlert}}"
        operator: "=="
        value: true
```

```yaml
# Correct: free condition, no LLM call needed
- id: send-alert
  entryConditions:
    onCriteriaFail: "skip"
    conditionText: "Only alert for high-score leads"
    criteria:
      - variable: "{{steps.score.value}}"
        operator: ">"
        value: 80
```

Use AI for decisions that require reasoning, judgment, or context. Use conditions for decisions that follow a deterministic rule.

---

## `onCriteriaFail` reference

| Value | Behavior |
|---|---|
| `"skip"` | Skip this step, continue to the next |
| `"stop"` | Stop the entire execution |
| `"wait"` | Block this step until criteria are met (used for loop/group completion) |

Common mistakes:
- Using `"stop"` when you mean `"skip"` — stops the whole workflow instead of just bypassing one step
- Using `"skip"` when you mean `"wait"` — the step runs immediately with incomplete data instead of waiting for a loop to finish

---

## One-line rule

> Always use `criteria` (not `conditions`) and `variable` (not `field`) in entry conditions — wrong field names are silently ignored and the condition never evaluates.
