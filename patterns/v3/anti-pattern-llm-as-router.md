# Anti-pattern — LLM as router: using AI for binary decisions a condition handles for free

**The shape**: A workflow needs to decide "should we proceed?", "is this a HOT lead?", "is the score above 70?" — and uses an AI step to make the decision. The model returns "yes" or "no" (or "HOT" / "WARM" / "COLD") and the next step branches on its output.

**Why it's tempting**: The decision feels nuanced. "Look at the score and category and decide if we should send the email" sounds like the kind of judgment LLMs are good at. The first version, written quickly, is an AI step with a prompt that says "Decide: yes or no."

---

## What it actually costs you

| Concern | LLM router | Conditional |
|---|---|---|
| Latency | 1–5 seconds | < 1ms |
| Cost | $0.001–0.05 per decision | Free |
| Determinism | Stochastic (same input, sometimes different answer) | Always identical |
| Auditability | "The model said yes" | "score > 70 AND category == 'HOT'" |
| Debugging | Inspect the model's reasoning, hope it's stable | Read the boolean expression |

For a binary decision based on values that already exist in step output, you trade four orders of magnitude on latency and cost for *less* reliability than a hardcoded condition.

---

## Anti-pattern (broken)

```yaml
- id: score-lead
  type: aiAction
  prompt: "Score this lead..."
  output: { score: number, category: HOT|WARM|COLD }

- id: should-alert
  type: aiAction
  prompt: |
    Based on this scored lead:
    Score: {{steps.score-lead.score}}
    Category: {{steps.score-lead.category}}

    Should we send a HOT alert? Reply "yes" or "no".
  output: { decision: string }

- id: send-alert
  if: "{{steps.should-alert.decision == 'yes'}}"
  action: slack.post
```

The AI router step:
- Costs credits to evaluate `category == "HOT"` — a comparison the model didn't help shape
- Sometimes returns "Yes." vs "yes" vs "Yes, alert." breaking the equality check
- Adds 2–5 seconds of latency per execution
- Has no audit trail beyond a raw model response

---

## Correct pattern

```yaml
- id: score-lead
  type: aiAction
  prompt: "Score this lead..."
  output: { score: number, category: HOT|WARM|COLD, reasoning: string }

- id: send-alert
  if: "{{steps.score-lead.category == 'HOT'}}"     # ← deterministic, free, audit-friendly
  action: slack.post
```

The AI step did the hard part — looking at the lead and producing structured output. **The decision based on that output is a condition, not another AI step.** The whole point of getting structured output from the model is so downstream code doesn't need to ask the model again.

---

## When AI for routing is actually correct

LLM routing is the right call when:

- **The input is unstructured text** that must be classified before code can branch on it (incoming email → support / billing / sales). No deterministic rule exists.
- **The decision genuinely requires judgment** (this customer's complaint is escalation-worthy vs the usual). The criteria can't be enumerated.
- **There are many output classes** with overlapping criteria — a regex tower would be unmaintainable.

In all three, the LLM is doing the **classification**, not the routing. Use one AI step to classify; use a condition to route on the classification's output.

```
unstructured_email → AI classify → category enum → condition route on category
```

Not:

```
unstructured_email → AI classify → AI decide where to send → action
```

The second AI call is doing the work the condition could do for free.

---

## The "hidden" LLM router

Sometimes the router hides inside an action that "just" composes a message:

```yaml
- id: maybe-email
  type: aiAction
  prompt: |
    If the lead is HOT, write an email.
    If WARM, write a follow-up reminder.
    If COLD, return an empty string.
  output: { message: string }

- id: send
  if: "{{steps.maybe-email.message != ''}}"
  action: send-email
```

The AI step is doing routing AND composition. Symptoms: empty-string outputs sometimes contain "I have no message to write." instead of `""`, breaking the condition. The model occasionally writes for COLD leads anyway.

**Fix**: split the routing (condition on `category`) from the composition (AI step that composes only when the condition fires).

```yaml
- id: compose-hot-email
  if: "{{steps.score-lead.category == 'HOT'}}"
  type: aiAction
  prompt: "Write a HOT-lead outreach to..."
- id: send
  if: "{{steps.score-lead.category == 'HOT'}}"
  action: send-email
```

Now each AI step has one job, and routing is in the condition where it belongs.

---

## Detection signals

| Signal | Likely cause |
|---|---|
| AI step's only purpose is returning "yes" / "no" / a category | LLM router |
| AI step's output is a small enum and the next step branches on it | LLM router (the prior step should have set the enum) |
| Tests fail intermittently because the model's wording shifts | LLM router with non-deterministic output |
| Adding a new branch requires editing a prompt instead of a condition | LLM router |
| Same execution, same input, different routing on rerun | Stochastic LLM in the routing path |

---

## One-line rule

> Use AI to produce structured output, use conditions to branch on it; an LLM step whose only output is a category that the next step branches on should always be a condition instead.
