# Anti-pattern — Prompt-in-loop: calling an LLM per iteration when batching works

**The shape**: A loop iterates over N items. Inside the loop, an AI step prompts the model to "evaluate this item" — once per item. N=100 means 100 LLM calls.

**Why it's tempting**: Each prompt is small and self-contained. "Score this one company" is easier to write than "score all 100 companies and return a structured array". The first version of any scoring workflow looks like this because it's the obvious first thing to type.

---

## The cost compounds along three axes

| Axis | Per-iteration cost | Batched cost |
|---|---|---|
| **API calls** | 100 round trips | 1–5 round trips |
| **Tokens** | 100× the system prompt + criteria | 1× the system prompt + criteria, items are content |
| **Wall clock** | 100× sequential latency (or rate-limit bound) | One call's latency |
| **Credits / $** | 10–50× more for the same logical work | Baseline |

A loop that calls an LLM 100 times typically costs 10–50× a single batched call doing the same work, takes 30–120× longer, and is more likely to hit rate limits.

---

## Anti-pattern (broken)

```yaml
- id: process-list
  type: loop
  over: "{{steps.fetch.items}}"
  steps:
    - id: score-item
      type: aiAction
      prompt: |
        Score this item against our criteria:
        Item: {{currentItem}}
        Criteria: {{state.criteria}}

        Return: { score: number, reasoning: string }
```

100 items → 100 calls → 100× the system prompt → ~50 minutes wall clock at 30s/call serially.

---

## Correct pattern: batch-prompt with structured output

```yaml
- id: score-batch
  type: aiAction
  prompt: |
    Score each item against our criteria.

    Items:
    {{steps.fetch.items}}

    Criteria:
    {{state.criteria}}

    Return an array with one entry per input item, in the same order:
    [{ id: string, score: number, reasoning: string }]
  output_schema:
    items:
      type: array
      items: { id: string, score: number, reasoning: string }
  validate:
    output_length_matches_input: true
```

One call. The system prompt and criteria appear once. The model is good at "do this for each item" — that's a single coherent task.

---

## When per-item is actually correct

Batching is wrong when:

- **Item processing requires a tool call** that depends on the item (web fetch, KG lookup). Each item legitimately needs its own AI-tool loop.
- **Items are too large to fit together** in the model's context. (Try summarizing first; if you can't, batch what fits and chunk.)
- **Per-item failures must be isolated.** If item 50 throws, batched output is lost; per-item retries are easier.
- **Outputs feed downstream steps differently per item** (different branches, different recipients). A loop is then the right shape, but only the *fan-out* part — the AI evaluation can still be batched upstream.

The default should be **batch first, loop only when batching is impossible**, not the other way around.

---

## Hybrid: batch evaluate, loop only the side effect

The most common correct shape:

```
fetch items
  → batch-evaluate (1 LLM call returning scored array)
  → loop over scored items (only the side effect — write to CRM, send email)
```

The expensive step (LLM) runs once. The cheap step (per-item write) loops. Cost stays bounded as N grows.

---

## Detection signals

| Signal | Likely cause |
|---|---|
| AI credit usage scales linearly with input list size | Prompt-in-loop |
| Wall clock runtime scales with N (not log-N) | Prompt-in-loop |
| Rate limit errors during loops | Prompt-in-loop hitting per-second limits |
| Same system prompt appears 100× in execution logs | Prompt-in-loop, classic |

If your AI cost doubles when your input list doubles, you're in the loop. Refactor to a batch prompt.

---

## When you must loop, batch the loop

If genuine per-item AI work is required (each item needs its own tool calls, etc.), still batch the overhead:

- Reuse the same system prompt across iterations (cache where possible).
- Group items by category and run one AI step per category, not per item.
- For very large N, run M parallel calls of size N/M instead of N serial calls.

Every order-of-magnitude reduction in call count compounds against credit cost, latency, and rate-limit risk.

---

## One-line rule

> Default to a single batch prompt with array output; loop the LLM only when each item requires distinct tool calls — and when you must loop, batch by category, not by item.
