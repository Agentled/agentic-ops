# 02 — Dedup gates: idempotency for agentic workflows

**Problem**: Without a dedup gate, every record in a polling or webhook workflow gets processed multiple times — silently, expensively, and often with conflicting writes.

**Why it fails silently**: The first few runs look correct. Duplicates only surface when you notice your CRM has 3 entries for the same company, your enrichment bill is 2× expected, or your outreach tool sent the same email twice. By then, the damage is done.

---

## The Gmail label-ID bug (the most common dedup failure)

This is the one that wastes 2 hours and isn't documented anywhere.

You build an email polling workflow. You add a step to mark each email as processed. You pass the label name:

```json
{
  "action": "GMAIL_ADD_LABEL",
  "input": {
    "message_id": "{{currentItem.id}}",
    "label_id": "processed"
  }
}
```

Result: `400 Bad Request: Invalid label: processed`

The Gmail API does not accept label **display names**. It requires internal label **IDs** — strings that look like `Label_3456789012345678`. The display name "processed" is what you see in Gmail's UI. The ID is what the API needs.

Same bug with any user-created label: `"agentled"`, `"reviewed"`, `"done"` — all invalid.

---

## Anti-pattern

```json
// Wrong: passing label display name
{
  "action": "GMAIL_ADD_LABEL",
  "input": {
    "message_id": "{{currentItem.id}}",
    "label_id": "processed"
  }
}
// → 400: Invalid label: processed
```

---

## Correct pattern

Always resolve the label ID first using a create-or-get-label step:

```json
// Step 1: create label if it doesn't exist, or get existing (idempotent)
{
  "id": "ensure-label",
  "action": "GMAIL_CREATE_LABEL",
  "input": { "name": "processed" }
}
// Returns: { "id": "Label_3456789012345678", "name": "processed" }

// Step 2: fetch unprocessed emails
{
  "id": "fetch-emails",
  "action": "GMAIL_FETCH_EMAILS",
  "input": {
    "query": "-label:processed newer_than:1d",
    "max_results": 50
  }
}

// Step 3 (inside loop): mark each email processed using the resolved ID
{
  "id": "mark-processed",
  "action": "GMAIL_ADD_LABEL",
  "input": {
    "message_id": "{{currentItem.id}}",
    "label_id": "{{steps.ensure-label.id}}"  // ← resolved ID, not display name
  }
}
```

`GMAIL_CREATE_LABEL` is idempotent — if the label already exists, it returns the existing label's ID. Run it every time with no side effects.

---

## How label-based dedup works

The `-label:processed` filter in the fetch query does the dedup work:

1. First run: fetches 50 emails. Processes each. Adds `processed` label to each.
2. Second run: fetches emails without `processed` label. Those 50 are now excluded. Only new emails are returned.
3. Outage for 3 days: widen to `newer_than:7d` on the next run. All unprocessed emails in the window are caught. Processed ones are excluded.

This gives you **exactly-once processing** with no database, no external state store, and no coordination overhead.

---

## Dedup for webhook triggers

Webhooks re-deliver. Always. Your endpoint will receive the same event 2–5× under normal conditions (retries on timeout, delivery confirmation failures). Without dedup:

```
Webhook fires → workflow starts → enrichment call × 3 duplicates → 3 CRM entries
```

The fix: use a unique event ID as an idempotency key and check before processing:

```javascript
// Code step at workflow entry
const eventId = input.webhookPayload.id;  // or messageId, leadId, etc.
const alreadyProcessed = await kv.get(`processed:${eventId}`);
if (alreadyProcessed) {
  return { skipped: true, reason: "duplicate" };
}
await kv.set(`processed:${eventId}`, true, { ttl: 86400 });
```

---

## Dedup patterns by source

| Source | Dedup mechanism |
|---|---|
| Gmail polling | `-label:processed` query + `GMAIL_ADD_LABEL` after processing |
| Webhook | Idempotency key from event ID, stored in KV or DB |
| Scheduled API poll | Cursor / `since_id` / `updated_at` timestamp stored in persistent memory |
| File/S3 intake | Move to `processed/` prefix after reading |
| Form submissions | Unique submission ID checked before processing |

---

## One-line rule

> Always resolve label IDs before passing them to the Gmail API — display names cause a silent 400 error — and always add the processed label as the final step in every email intake loop.
