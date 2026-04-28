# Anti-pattern — God workflow: 30-step monoliths instead of composable child workflows

**The shape**: One workflow does sourcing, enrichment, scoring, outreach, and reporting end-to-end. 30+ steps in a single graph. Every change requires understanding the whole thing. Every failure restarts the whole thing.

**Why it's tempting**: It's the natural first draft. You start with "here's what I want to happen" and add steps until the goal is reached. There's no friction asking you to stop and split. The orchestrator allows arbitrary length, and a long graph runs the same as a short one — until you have to maintain it.

---

## The five things that break

### 1. Failure restarts the entire pipeline

Step 27 fails. The workflow has produced 26 steps of valuable intermediate output. Retry restarts at step 1. You burn the credits twice and pray the upstream APIs return the same data.

A composable shape (orchestrator → child workflow per logical phase) lets each phase retry independently. A sourcing failure doesn't re-run scoring.

### 2. No reuse across workflows

You write a 30-step "sourcing + enrichment + scoring + outreach" pipeline for SaaS startups. Next week you need the same enrichment + scoring for a fintech list. You copy-paste 20 steps and edit a prompt. Now you have two pipelines that drift.

A child workflow (`enrichment + scoring`) is called from both — one source of truth.

### 3. Concurrency is impossible

A god workflow has one input → one output. If you want to process 10 items in parallel, you can't — the graph is shaped for one. To parallelize, you'd need to fork the entire workflow.

A child workflow can be called in a loop with concurrent dispatch. The parent stays simple; concurrency is an orchestrator-level concern, not encoded in the child's structure.

### 4. Testing requires running the whole thing

To test step 18, you must produce the upstream context for steps 1–17. That means running the workflow. Iteration loop: edit prompt → run 17 unrelated steps → see step 18 output → repeat. Every cycle costs credits.

Child workflows are unit-testable: call them with sample input, see the return value. Iteration on the scoring child workflow doesn't require running sourcing first.

### 5. Cognitive load grows quadratically

A 30-step graph has roughly 30² possible interactions to reason about. New maintainers can't read it; existing maintainers can't change one step without worrying what they break elsewhere.

A 5-step parent that calls 4 child workflows of 5 steps each has the same total work, but each piece is independently comprehensible.

---

## Anti-pattern (broken)

```
trigger
  → fetch-companies
  → normalize
  → enrich-1
  → enrich-2
  → enrich-3
  → score-against-criteria-A
  → score-against-criteria-B
  → combine-scores
  → filter-top-N
  → look-up-contacts (loop)
  → enrich-contacts (loop)
  → find-emails (loop)
  → draft-outreach (loop)
  → approval-gate
  → send-emails (loop)
  → write-to-crm (loop)
  → generate-report
  → store-report
  → notify-team
  → milestone
```

20+ steps, 4 loops, one approval gate. One bug anywhere and the whole graph reruns.

---

## Correct pattern: orchestrator + child workflows

```
parent (orchestrator)
  → trigger
  → call: sourcing            (child: fetch + normalize)
  → call: enrichment-batch    (child: enrich + dedup)
  → call: scoring             (child: score + filter)
  → loop over top results
      → call: outreach        (child: contact-lookup + draft + approval + send)
  → call: reporting           (child: generate + store + notify)
  → milestone
```

The parent is 5 logical steps. Each child is 5–10 steps with a clear contract (`return` step with typed output). Each child is independently:
- Testable (run with sample input, inspect return)
- Retryable (rerun the failed child, not the whole pipeline)
- Reusable (call from any other workflow that needs the same phase)
- Replaceable (swap an enrichment provider by editing one child)

---

## How to split: the contract test

When you're not sure where to split, ask: **what's the typed interface I'd write for this slice of the workflow?**

- Sourcing: `() → list of companies`
- Enrichment: `company → enriched company`
- Scoring: `enriched company → { score, decision, reasoning }`
- Outreach: `(enriched company, criteria) → { sent: bool, message_id: string }`

Anything you can write a typed contract for is a candidate child workflow. If you can't write a contract because the slice's behavior depends on too many implicit upstream details, the slice isn't well-bounded — that's also a signal to refactor.

---

## When a single workflow is actually correct

Don't over-split. A single workflow is fine when:

- The workflow has fewer than ~10 steps total
- Every step depends linearly on the previous and has no reuse value
- It's a one-off (a custom report, a migration), not a recurring production pipeline
- The team is one person and there's no maintenance horizon

Splitting has a real cost: more files, more contracts to maintain, more orchestration logic. Don't split for splitting's sake. Split when reuse, retry isolation, or testability would benefit — or when the graph crosses ~15 steps.

---

## Detection signals

| Signal | Likely cause |
|---|---|
| Workflow has >20 steps in a single graph | God workflow |
| Editing one step requires re-running the whole workflow to test | God workflow |
| Two different workflows have 5+ identical steps | Missing child workflow |
| Failures consistently happen late and force full reruns | Phases should be separate children |
| New maintainers spend > 1 hour understanding the graph | Cognitive overload from monolith |

---

## One-line rule

> If a workflow has more than ~15 steps or contains slices you'd reuse elsewhere, split into a parent orchestrator that calls typed child workflows; god workflows make every failure expensive and every change risky.
