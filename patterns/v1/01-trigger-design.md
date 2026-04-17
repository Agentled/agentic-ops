# 01 — Trigger design: polling vs event triggers

**Problem**: Developers default to event triggers for email/document intake workflows, creating fragile pipelines that drop records, can't backfill, and are hard to debug.

**Why it fails silently**: Event triggers appear to work in testing (low volume, reliable delivery). At production scale, re-deliveries cause duplicates, Pub/Sub TTL loses events during outages, and there's no way to backfill records missed during downtime — without any visible error.

---

## Decision framework

| | Schedule (polling) | App Event (real-time) |
|---|---|---|
| **Latency** | minutes–hours | seconds |
| **Idempotency** | trivial — label/flag marks processed | must dedupe on messageId; re-deliveries happen |
| **Backfill** | built-in — widen the query window | doesn't exist; needs a separate bootstrap run |
| **Replay after outage** | automatic on next scheduled run | events can be permanently lost (TTL) |
| **Debugging** | read last execution log | subscription status + delivery + filter + dedupe all need checking |
| **Infrastructure** | none | webhook receiver, watch renewal, Pub/Sub |

**Default rule: polling for intake, events for reactions.**

---

## Anti-pattern

Using an event trigger for email intake because it "feels more real-time":

```yaml
# Wrong: event trigger for deal flow email intake
trigger:
  type: app_event
  app: gmail
  event: GMAIL_NEW_MESSAGE_RECEIVED
  filters:
    query: "from:investor subject:pitch"
```

Problems:
- Duplicate delivery means the same email gets processed 2-3× with no dedup mechanism
- Gmail watch tokens expire — you need a renewal job or emails stop arriving silently
- No backfill: if the workflow is down for 2 days, those emails are gone
- Debugging requires checking: is the watch active? Did the webhook fire? Did the filter match? Did the dedup run?

---

## Correct pattern

Schedule trigger with label-based dedup:

```yaml
# Correct: scheduled polling with dedup gate
trigger:
  type: schedule
  config:
    frequency: daily
    time: "08:00"

steps:
  - id: fetch-emails
    action: GMAIL_FETCH_EMAILS
    input:
      query: "-label:processed newer_than:1d"
      max_results: 50

  - id: process-email
    type: loop
    over: "{{steps.fetch-emails.messages}}"
    # ... processing steps ...

  - id: mark-processed
    action: GMAIL_ADD_LABEL
    input:
      message_id: "{{currentItem.id}}"
      label_id: "{{steps.ensure-label.id}}"  # resolved ID, not display name
```

The `-label:processed` filter does the dedup work. Each email is processed exactly once. If the workflow goes down for a week, widen to `newer_than:7d` on the next run to backfill.

---

## When to use event triggers

Event triggers are correct when:
- The user explicitly requires sub-minute latency ("alert within 30 seconds", "as soon as", "real-time")
- The workflow is a **side effect** (fire-and-forget notification), not a record-of-truth producer
- Missed events are acceptable (or you have a separate reconciliation job)

---

## Trigger type cheatsheet

| User says | Trigger |
|---|---|
| "process inbound pitch emails" | Schedule (daily) |
| "triage support emails every morning" | Schedule (daily 08:00) |
| "every Monday summarize last week's emails" | Schedule (weekly) |
| "analyze my inbox and create Notion entries" | Schedule (daily) |
| "page oncall within 30s of an escalation email" | App event |
| "create a ticket the moment a customer emails" | App event |
| "run every time a form is submitted" | Webhook |
| "user clicks Run" | Manual |

---

## One-line rule

> Default to Schedule + label-based dedup for email and document intake; use event triggers only when the user explicitly states a latency requirement under one minute.
