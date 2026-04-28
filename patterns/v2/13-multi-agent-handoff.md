# 13 — Multi-agent handoff: passing context without prompt drift

**Problem**: A multi-agent system (researcher → analyst → writer, or qualifier → enricher → outreach) loses context at each hop. The downstream agent gets a summary of a summary, hallucinates the missing details, and produces output that's plausible but disconnected from the original input.

**Why it fails silently**: Each handoff individually looks fine — the previous agent produced text, the next agent received text. There's no schema violation, no thrown error. The drift only shows up when you compare the final output to the original input and notice the company name changed, the user's request was reinterpreted, or a key constraint disappeared. By then the workflow has been "working" for weeks.

---

## Three handoff styles, ranked

### Worst: free-text relay

Agent A produces a paragraph. Agent B reads the paragraph and decides what's relevant. Agent C reads B's paragraph.

```
A: "User wants to find SaaS companies in fintech with 50-200 employees that recently raised Series A."
B: "Looking for fintech startups with recent Series A funding."          ← lost: SaaS, headcount
C: "Found Series A fintech startups."                                     ← lost: 50-200 employees, SaaS
```

Each agent compresses to fit its prompt. Fields silently disappear. This is the default if you do nothing — and it's wrong.

### Better: typed handoff with structured payload

```yaml
handoff:
  schema:
    user_intent: string
    constraints:
      industry: string
      employee_range: { min: int, max: int }
      funding_stage: string
      company_type: string
    raw_input: string         # always include the source for fallback
  required: [user_intent, constraints.industry, raw_input]
```

Agent A fills the schema. Agent B reads the schema (not a paraphrase of the schema). Validation rejects an A output that omits required fields. Drift can't accumulate because each agent reads the same canonical structure.

### Best: typed handoff + immutable source-of-truth reference

The structured payload is the working contract. Every agent also has read access to the **original input** (and the latest committed plan), not just its predecessor's output:

```yaml
agent_context:
  source: "{{trigger.original_input}}"     # immutable, set once at workflow start
  plan:   "{{state.current_plan}}"          # updated by planner agents only
  prior_outputs:                            # all upstream outputs, indexed by step
    research: "{{steps.research.output}}"
    analysis: "{{steps.analysis.output}}"
```

The downstream agent can always re-derive a missing field from the source rather than hallucinate it. Agents that need to revise the plan write to a single shared `state.current_plan` slot — not to free-form text the next agent has to interpret.

---

## Anti-pattern

```
[Researcher prompt]
"Research the user's request and write a summary for the analyst."

[Analyst prompt]
"You are an analyst. Read this research summary and produce findings:
{{steps.researcher.output}}"
```

The analyst has no access to the original request. If the researcher missed a detail or paraphrased it ambiguously, the analyst can't recover. The analyst will fill the gap with the most likely-sounding completion, and you won't know.

---

## Correct pattern

```yaml
- id: research
  type: agent
  prompt: |
    Original request: {{trigger.input}}
    Plan: {{state.plan}}

    Produce structured research output with these fields:
    - companies: [{ name, url, why_match }]
    - sources_used: [string]
    - confidence: number 0-1
    - unresolved_questions: [string]
  output_schema:
    companies: array
    sources_used: array
    confidence: number
    unresolved_questions: array
  validate:
    required: [companies, confidence]
    confidence_threshold: 0.6   # below this, escalate

- id: analyze
  type: agent
  prompt: |
    Original request: {{trigger.input}}                 # ← source of truth
    Plan: {{state.plan}}
    Research output: {{steps.research}}                  # ← typed payload

    Produce analysis with fields:
    - top_3: [{ company_name, fit_score, reasoning }]
    - flagged: [{ company_name, concern }]
  output_schema:
    top_3: array
    flagged: array
```

Three things stack:
1. **Original input** is passed to every agent — drift can be corrected at any hop.
2. **Output schema** is enforced — each agent's output is validated, not trusted.
3. **Confidence and unresolved questions** are explicit fields — agents declare uncertainty rather than guessing past it.

---

## When to use a planner-executor split

For tasks that span more than 3 agent hops, separate the **planner** (decides what to do) from the **executors** (do one thing each). The planner reads the source of truth, writes a structured plan, and dispatches executor agents with narrowly-scoped tasks. Executors never modify the plan — they only return results.

```
planner:    reads trigger.input → writes state.plan (list of executor calls)
executor_1: reads state.plan[0] → writes state.results[0]
executor_2: reads state.plan[1] → writes state.results[1]
synthesizer: reads state.plan + state.results → writes final output
```

This eliminates drift because every agent reads from a single canonical state, not from each other's output.

---

## Anti-pattern: passing the full conversation history forever

```
agent_2_prompt:
  Here is everything that has happened so far: {{conversation_history}}
  Now do your part.
```

The history grows with every hop. By hop 5 the prompt is 30k tokens. Costs balloon, and the agent's attention drifts to whichever part of the history happened to be most engaging. **Pass structured state, not transcripts.**

If a downstream agent needs to know what an earlier one decided, retrieve the **decision** (a single field), not the entire reasoning trace.

---

## Anti-pattern: implicit state via tool calls

Some frameworks let agents call tools that mutate shared state without declaring it. Agent A writes to memory; agent B calls a memory-recall tool and gets the value. This works until two agents run concurrently or the memory tool returns stale data.

**Make state writes explicit and ordered.** If shared state matters, write it as a step output or to a typed slot in the workflow's state object — not via implicit tool calls.

---

## Detection signals

| Signal | What it means |
|---|---|
| Final output references entities not present in trigger input | A hop hallucinated; check intermediate outputs for first appearance |
| Required input field disappears between hops | Free-text relay; switch to typed schema |
| Agent confidence drops monotonically across hops | Each agent is uncertain about the previous one; pass source of truth |
| Same input produces wildly different outputs across runs | Drift is non-deterministic; tighten schemas and lower temperature |

---

## One-line rule

> Pass typed structured payloads between agents and always include the original input as an immutable reference; free-text handoffs accumulate drift that can't be detected from any single hop.
