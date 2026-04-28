# Anti-pattern — Missing dedup: running polling workflows without a dedup gate (the cost math)

**The shape**: A workflow polls a source (email inbox, RSS feed, CRM, database) every N minutes/hours and processes everything it finds. There is no dedup mechanism. Every poll re-processes the items it found last time.

**Why it's tempting**: Skipping dedup is the path of least resistance on day one. The workflow runs, items get processed, the result looks correct because the first poll's output is right. The duplication only manifests on poll #2, by which time the workflow looks "live" and you've moved on to the next problem.

---

## The cost math (this is the hook)

Take a workflow that:
- Polls a 50-message inbox **hourly**
- Each message triggers an AI enrichment step costing **10 credits**
- One credit ≈ $0.001 to the user

**Without dedup:**
```
50 messages × 10 credits × 24 polls/day = 12,000 credits/day
                                        = $12/day
                                        = $360/month
```

For one workflow processing the *same 50 messages over and over*. Most of those credits buy nothing.

**With dedup:**
```
~5 new messages/poll × 10 credits × 24 polls/day = 1,200 credits/day
                                                  = $1.20/day
                                                  = $36/month
```

Same workflow, dedup gate added, **10× lower cost** — and the missing 90% wasn't producing any new value.

The cost compounds when:
- Polls are more frequent (every 10 min = 6× more wasted credits)
- Per-item cost is higher (multi-step enrichment with web search)
- The source is large (1000-row CRM table with 5 new rows/day)

A daily-polling workflow on a large source without dedup can cost **100× more than the same workflow with dedup gate** while doing the same useful work.

---

## What "no dedup" looks like

```yaml
trigger:
  type: schedule
  frequency: hourly

steps:
  - id: fetch-emails
    action: gmail.list
    input: { query: "from:investors" }     # ❌ no exclusion of already-processed
  - id: process
    loop: "{{steps.fetch-emails.messages}}"
    steps:
      - enrich
      - score
      - write-to-crm
```

Every hour, every message in the inbox gets re-enriched, re-scored, re-written. The CRM accumulates duplicate rows. The enrichment APIs get hammered with the same lookups. The bill grows.

---

## Three dedup mechanisms that work

### 1. Source-side filter (best when supported)

Mark items as processed at the source, query for unprocessed:

```yaml
- id: fetch-emails
  action: gmail.list
  input: { query: "-label:processed newer_than:1d" }
- id: process
  # ...
- id: mark-processed
  action: gmail.add-label
  input: { message_id: "{{currentItem.id}}", label_id: "{{label.id}}" }
```

The source itself excludes processed items. Cheapest, simplest, no extra storage.

Available in: Gmail (labels), Outlook (categories), Slack (reactions), most ticketing systems (statuses), most CRMs (custom field), HubSpot (lifecycle stage).

### 2. Workflow-side state (when source doesn't support marking)

Maintain a `processed_ids` table; check before processing:

```yaml
- id: fetch-items
  action: source.list

- id: dedup-check
  query: "SELECT id FROM processed_ids WHERE id IN ({{ids}})"

- id: process
  loop: "{{steps.fetch-items.items - steps.dedup-check.found}}"
  steps:
    - enrich
    - record-as-processed     # writes to processed_ids
```

Use when the source is read-only (RSS feed, public API, third-party data you can't mutate).

### 3. Idempotent sink (last line of defense)

Even with source-side dedup, races and bugs leak. Make the destination idempotent:

```sql
CREATE UNIQUE INDEX uniq_outreach
  ON outreach (source_id, recipient_email);
```

The second insert fails the unique constraint. Your workflow catches the error and treats it as a no-op success. Even if dedup is bypassed somewhere upstream, you don't end up with duplicate rows. (See v2 pattern 12 — Idempotency.)

**Layer all three** for production polling workflows. Each catches what the others miss.

---

## Detection: how to spot a leak

| Signal | What it means |
|---|---|
| AI credit usage steady-state instead of trending toward zero | Reprocessing the same items |
| CRM has duplicate rows for the same source entity | Sink not idempotent + dedup missing |
| External APIs return 429 rate limit during polls | Hitting them with redundant requests |
| Per-poll runtime is constant regardless of new-item volume | Loop processes all items, not just new ones |
| Cost-per-result trending up over time | The denominator (real new work) shrinks; the numerator stays constant |

The most diagnostic metric: **credits-per-unique-output**. If it climbs as the source grows, dedup is missing or partially broken.

---

## Special case: the "I'll add dedup later" trap

The most common path to a missing dedup gate is "we'll add it after we ship". The workflow runs, costs accrue, and the dedup gate is never the most important thing on the backlog. Six months later you discover the pipeline has been processing the same 50 messages every hour since launch.

**Add the dedup gate before the first run.** Even a stub (`processed_ids` table + simple insert) is enough — you can refine the strategy later, but starting with no gate means starting with cost runaway. The gate is *not* an optimization; it's a correctness requirement for any polling workflow.

---

## When you can skip dedup

Only when:
- The workflow is one-shot (manual trigger, runs once, never again)
- The source is empty or near-empty by design (workflow runs on initially-empty queues)
- "Reprocessing" is genuinely the goal (re-scoring all leads weekly, on purpose, with full awareness of the cost)

In all three cases, document the decision in the workflow description so the next maintainer doesn't add dedup thinking it was forgotten.

---

## One-line rule

> Polling workflows without a dedup gate burn credits proportional to source size × poll frequency on work that was already done; add the gate before first run, not after the bill arrives.
