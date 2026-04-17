# 00 — Why agentic-ops

> The ops discipline that makes AI agents production-ready.

---

## The pitch deck demo works. Production doesn't.

You've seen it: an AI agent that looks brilliant in a demo — processes a lead, writes a personalized email, updates the CRM. Then you run it on 500 leads a week and the wheels fall off. Duplicates. Missing records. Costs 3× what you expected. No way to know what actually went through.

This isn't an AI problem. It's an **architecture problem**.

The fix has a name in every other engineering discipline: operations. CI/CD, observability, idempotency, retries, caching, audit trails — these didn't exist on day one of web development either. Developers invented them because raw execution didn't scale. Agentic workflows are at the same inflection point.

This is **agentic-ops**: the patterns that make the gap between "demo" and "production" crossable.

---

## 1. Unit economics collapse without dedup discipline

This is the argument that lands before developers hit the wall themselves.

You're processing 500 inbound leads per week. Your workflow enriches each one (5 credits). No dedup gate — the workflow polls every hour, no label marking what's been processed.

- Average lead appears in 3 polls before it's handled: **1,500 enrichments instead of 500**
- At 5 credits each: **5,000 wasted credits per week**
- At scale, you're paying for work you already did

This isn't hypothetical. It's the default outcome of any polling workflow without a dedup gate. See [02-dedup-gates](02-dedup-gates.md) for the fix.

---

## 2. Non-determinism at scale

A one-off prompt works once. Run the same prompt 1,000 times against your pipeline and you get 1,000 slightly different outputs — different field names, different score ranges, different JSON shapes. No two downstream steps can reliably consume the previous one.

Structured workflows define a **response contract**: the output schema is declared upfront, validated at runtime, and consistent across every execution. The AI fills in the values; the workflow enforces the shape.

Without this contract, you're parsing surprises, not outputs.

---

## 3. No audit trail, no trust

"The AI processed our deal flow last week" — can you tell me exactly which companies it looked at, what data it used, what score it assigned, and why three were dropped? If not, you can't trust it in any context that matters: regulated industries, board reporting, customer-facing processes.

Structured workflows produce **per-step execution records**: inputs, outputs, duration, status, timestamp. You can replay any run. You can compare two runs on the same input. You can show an auditor exactly what happened.

Ad-hoc prompts produce a chat message and nothing else.

---

## 4. Retries without idempotency destroy partial state

An AI agent that crashes halfway through is worse than one that never started. You have partial writes: some records enriched, some not; some CRM entries created, some missing. You don't know where it stopped.

Structured workflows solve this at two levels:
- **Step-level retry**: resume from the exact failed step with the same inputs — no re-running the work that succeeded
- **Idempotency gates**: dedup keys, label markers, and processed flags ensure a retried step produces the same outcome as the original, not a duplicate

See [07-error-handling](07-error-handling.md) and [02-dedup-gates](02-dedup-gates.md).

---

## 5. Caching requires stable step boundaries

Enriching the same LinkedIn company URL twice in the same workflow shouldn't hit the API twice. But caching requires a **stable cache key**: a defined input that uniquely identifies the computation.

Ad-hoc prompts have no stable key — the prompt is the key, and it changes with every rephrasing. Structured steps have explicit inputs, which means the platform can cache at the step level, deduplicate in-flight requests, and short-circuit repeated work.

No step boundary = no cache key = no caching.

---

## 6. Production requirements don't exist in raw prompting

Running AI agents at production scale without structured execution is like running a web server without a framework. You'll reinvent every wheel:

- **Observability**: knowing which step is running, how long it's taking, when it silently failed
- **Rate limiting**: preventing a burst of executions from hammering a downstream API
- **Concurrency control**: ensuring two parallel runs don't write conflicting records to the same CRM entry
- **Backpressure**: slowing intake when the processing queue backs up

These are solved problems in structured workflow systems. They don't exist as primitives in a prompt-and-hope architecture.

---

## 7. Human-in-the-loop gates need a first-class abstraction

"Have a human review the AI's output before it sends an email" sounds simple. In practice it requires: a UI to surface the output for review, a way to approve or reject, a mechanism to block the next step until the human acts, and a timeout path if they don't.

This is a first-class primitive in workflow systems — an approval gate with configurable behavior. In raw prompting, you build it from scratch every time.

Any workflow that touches customers, sends communications, or writes to a system of record should have a human-in-the-loop gate. See [09-human-in-the-loop](../v2/09-human-in-the-loop.md) (v2).

---

## The pattern

Every production failure in agentic workflows traces back to one of these seven gaps. The patterns in this repo address each one directly — not as theoretical advice, but as the specific implementations that fix them.

Start with the one that matches your current pain:
- Burning credits → [03-credit-efficiency](03-credit-efficiency.md)
- Processing duplicates → [02-dedup-gates](02-dedup-gates.md)
- Steps silently skipping → [06-conditional-routing](06-conditional-routing.md)
- Crashes with partial state → [07-error-handling](07-error-handling.md)
- Wrong trigger choice → [01-trigger-design](01-trigger-design.md)
