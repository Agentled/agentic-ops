# Anti-pattern — Event triggers for intake: using app events where polling + label dedup is correct

**The shape**: A workflow that processes an inbox, a folder, a new-record stream, or any "queue of incoming things" uses an app-event / push trigger ("on new email", "on file uploaded", "on row created"). The trigger fires per-event in real time. The workflow processes each event as it arrives.

**Why it's tempting**: It feels modern. Real-time intake sounds better than polling. The orchestrator's UI usually advertises event triggers as the headline feature. The first version works fine in testing — events arrive, workflow runs, processed. Production is where the failure modes show up.

---

## Why this fails (the four failure modes)

### 1. Re-delivery causes silent duplicates

Most event systems (Gmail Pub/Sub, Stripe webhooks, GitHub events) **redeliver** under at least these conditions:
- Receiver returns non-2xx (transient error, timeout)
- Receiver times out (slow downstream step)
- Provider's broker hiccups (more often than the docs admit)

Without explicit dedup on `event_id`, the same email gets processed twice, the same Stripe charge gets recorded twice, the same PR gets commented on twice. There's no error — the workflow ran successfully both times.

### 2. Lost events during outages have no recovery

Your workflow is down for 2 hours (deploy, infra issue, rate limit). Events that fired during the outage are gone. Some providers retry for a while; most stop after a few attempts and discard. There is no "list events from yesterday I missed" API for most event sources.

A polling workflow doesn't have this problem: when it comes back up, it queries the source for "everything since X" and processes the backlog. Polling is *naturally* recoverable; events aren't.

### 3. Per-event triggers create N×latency-bound concurrency

100 emails arrive in a minute. The orchestrator fires 100 workflow executions, each calling rate-limited APIs (LLM, enrichment provider). The first 20 succeed; the next 80 hit rate limits and fail. There's no natural backpressure.

A polling workflow batches by design: one execution processes the batch, paces the per-item work, and respects rate limits naturally.

### 4. Subscription state is its own failure surface

Event triggers depend on subscription state that you don't see and didn't create:

- Gmail watch tokens expire weekly — if no renewal, emails stop arriving silently
- Webhook secrets get rotated and the receiver isn't updated — events 401 at the source
- Cloud queue subscriptions get filtered by some upstream rule you forgot about

You discover the problem when someone says "we haven't gotten any leads in two days". The dashboard shows zero events arrived; the dashboard doesn't show *why* or *that something is wrong*. The default state is "silent".

A polling workflow has none of these. It runs on a schedule. If the source has items, it processes them. The subscription doesn't exist.

---

## Anti-pattern (broken)

```yaml
trigger:
  type: app_event
  app: gmail
  event: GMAIL_NEW_MESSAGE_RECEIVED
  filters:
    query: "from:investor"
  # ❌ no event_id dedup
  # ❌ no replay if subscription drops
  # ❌ no backfill mechanism
  # ❌ no rate-limit handling for spikes
```

---

## Correct pattern

For email and document intake, default to:

```yaml
trigger:
  type: schedule
  frequency: daily            # or hourly, depending on latency tolerance
  time: "08:00"

steps:
  - id: ensure-label
    action: gmail.create-label
    input: { name: "processed" }    # idempotent, returns existing if present

  - id: fetch-new
    action: gmail.list
    input:
      query: "-label:processed newer_than:1d"

  - id: process
    loop: "{{steps.fetch-new.messages}}"
    steps:
      - enrich
      - score
      - write-to-crm

  - id: mark-processed
    action: gmail.add-label
    input:
      message_id: "{{currentItem.id}}"
      label_id: "{{steps.ensure-label.id}}"
```

Five built-in properties:

1. **Idempotent** — the `-label:processed` filter excludes already-processed items
2. **Recoverable** — workflow down 2 days? Widen to `newer_than:3d` once
3. **Backpressure** — one execution processes the batch, naturally rate-paced
4. **No subscription drift** — there's no subscription
5. **Debuggable** — last execution log shows everything

---

## When event triggers are actually correct

Event triggers earn their complexity in three cases:

### 1. Sub-minute latency is a hard requirement

The user explicitly says "alert within 30 seconds" or "as soon as a customer emails support". You can't poll fast enough; events are required.

If the requirement is "within 5 minutes" or "real-time-ish", schedule every 5 minutes is simpler and good enough.

### 2. The workflow is a side effect, not a record producer

Posting an alert, firing a notification, kicking off a one-shot UI update — losing an event during an outage is acceptable because there's no expectation that every event produces a record.

If the workflow's job is to produce a row of truth (every email becomes a CRM lead), it's a record producer and event drops break the contract.

### 3. The event source has no pollable equivalent

A few sources are genuinely event-only (websocket streams, certain push-only APIs). For those, accept the complexity and add the discipline:

- Dedup by `event_id` against a persisted store
- A separate scheduled "reconciliation" workflow that polls the source weekly and catches anything dropped
- Monitoring that the event arrival rate is non-zero (silence is a signal)

You're now writing two workflows (the event consumer + the reconciliation) to get the equivalent of one polling workflow. That's the actual cost of event triggers.

---

## The decision in one minute

Ask: **what happens if my workflow is down for 2 hours?**

- "Items would be lost forever" → polling is the safe default
- "Items would queue up at the source and be processed when I come back" → either works; pick by latency
- "I'd lose only side-effect events that don't matter" → events are fine

Then: **do I need < 1 minute latency?**

- Yes, hard requirement → events
- No, minutes are fine → polling

In practice, 90% of "intake" workflows answer "items lost forever" + "minutes are fine" → polling.

---

## Detection signals

| Signal | Likely cause |
|---|---|
| "We haven't received any X in two days" | Event subscription expired silently |
| Same record appears multiple times in CRM | Re-delivery without `event_id` dedup |
| Events arrive in spikes that overwhelm rate limits | No batching, per-event invocation |
| Ops backfill required after every deploy | Events lost during deploy window |
| Workflow runs frequently with empty payloads | Filter applied wrong; events arriving for unrelated triggers |

If you've experienced two or more of these, switch to polling + label dedup. The reduced complexity will resolve all of them.

---

## One-line rule

> For email, document, and record intake, default to scheduled polling with label-based dedup; choose event triggers only when sub-minute latency is a hard requirement and you've budgeted for the dedup, recovery, and subscription-monitoring discipline they demand.
