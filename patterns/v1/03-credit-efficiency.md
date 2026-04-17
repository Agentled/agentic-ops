# 03 — Credit efficiency: not burning money while building

**Problem**: Developers restart full workflow executions to debug a single failed step, burning credits on work that was already done correctly.

**Why it fails silently**: The restarted execution appears to succeed. The wasted spend accumulates in the background — 3–5× expected credit usage during development — until the invoice arrives.

---

## The core discipline: fix → retry → verify

Every debugging cycle should follow exactly this sequence:

1. **Identify** the failed step and its error
2. **Fix** the configuration, prompt, or code
3. **Retry from the failed step** — not from the beginning
4. **Verify** the step output

Starting a new full execution to debug a failed step is the most expensive habit in agentic development. It re-runs every step that already succeeded: the enrichment API call, the LLM prompt, the database read. All paid again. None of them changed.

---

## Anti-pattern

```
Execution fails at step 5 (AI scoring)
→ Developer reads the error
→ Fixes the prompt
→ Starts a NEW execution from step 1
→ Steps 1-4 run again: enrichment (5 credits), profile fetch (2 credits), web scrape (0 credits), data parse (0 credits)
→ Step 5 runs with the fixed prompt
→ Total wasted: 7 credits × every debug cycle
```

In a workflow with 3 debug cycles per feature: 21 wasted credits before it works.

---

## Correct pattern

```
Execution fails at step 5 (AI scoring)
→ Developer reads the error
→ Fixes the prompt in the workflow config
→ Retries from step 5 — the platform reuses outputs from steps 1-4
→ Step 5 runs with the fixed prompt
→ Total wasted: 0 credits
```

Most workflow platforms expose a "retry from this step" action on failed executions. Use it every time.

---

## Test steps in isolation before wiring them

Before adding a step to a live workflow, test it standalone with representative input data:

```bash
# Test an AI step with real input — no execution, no credits for upstream steps
test_ai_action(
  template: "Analyze this company: {{input.company}}. Score fit 0-100.",
  responseStructure: { score: "number", reasoning: "string" },
  input: { company: { name: "Stripe", industry: "fintech", employees: 4000 } }
)

# Test a code step in the same sandbox as production
test_code_action(
  code: "return input.items.filter(i => i.score > 70)",
  input: { items: [{ name: "A", score: 85 }, { name: "B", score: 60 }] }
)
```

This catches errors before they're in a running execution. Zero credits for upstream steps.

---

## Mock downstream steps with prior output

When you need to test a downstream step (step 6) but don't want to re-run expensive upstream steps (steps 1-5):

1. Find a prior execution where steps 1-5 succeeded
2. Copy the output of step 5 from that execution
3. Use it as mock input to `test_ai_action` or `test_code_action` for step 6

```javascript
// Prior execution step 5 output (saved from execution abc-123):
const priorOutput = {
  company: { name: "Stripe", score: 85, signals: ["YC", "series B"] }
};

// Test step 6 in isolation using that output
test_ai_action(
  template: "Based on this profile, draft a 3-sentence outreach: {{input.company}}",
  input: priorOutput
)
```

No re-enrichment. No re-fetching. No wasted credits.

---

## One execution at a time

Don't start a new execution while one is in flight for the same workflow. Reasons:
- Parallel executions on the same data produce duplicate writes
- You can't read the output of execution A while debugging it if execution B is also running
- If both fail, you now have two half-processed states to reconcile

The discipline: start → observe → retry or fix → verify. Sequential, not parallel.

---

## Credit cost by step type (reference)

| Step type | Typical cost | Notes |
|---|---|---|
| AI action (standard model) | 5–15 credits | Varies by model tier and output length |
| Data enrichment (LinkedIn, Hunter) | 2–5 credits | Per-record cost |
| Web scrape | 0 credits | Free |
| HTTP request | 0 credits | Free |
| Code step | 0 credits | Free |
| Knowledge graph read/write | 1 credit | Flat |
| Browser automation | 10–15 credits | Per task |

Expensive steps are AI and enrichment. These are the ones you never want to re-run unnecessarily.

---

## One-line rule

> When a step fails, fix it and retry from that step — never start a new execution; use isolated step testing to catch errors before they're in a running workflow.
