# 08 — Composed email with approval gate: outreach that ships safely

**Problem**: Agents build outreach by chaining a "draft email" AI step with a separate "gmail send" app action. This skips the user-review step, sends unreviewed drafts, and can't display the email in the pending-approval UI.

**Why it fails silently**: A raw draft-then-send pipeline has no concept of "email" as a first-class artifact. The orchestrator's approval UI won't render a preview, the `schedule-email` action can't fire, and there's no audit trail of what was sent to whom. The workflow looks correct at authoring time and may even work in testing — until a real recipient gets an unreviewed draft.

---

## The composed-email shape

An email step is a specialized `aiAction` with four coupled pieces:

1. **Prompt type**: `pipelineStepPrompt.type: "email"` — tells the orchestrator this is an email step (not a generic AI step).
2. **Renderer**: `renderer: { type: "Email", config: { fromContextKey: "outreachProfile" } }` — tells the UI to render the result as an email preview, pulling sender info from a context input page.
3. **Integrations declaration**: declares that the workflow needs a connected email account (Gmail or Outlook).
4. **Approval + send action**: `onApproval: { action: "schedule-email" }` + `next.conditions.approvalRequired: true` — the step waits for user approval, then the `schedule-email` action actually sends the email.

All four must be present. Omit any one and the step silently degrades: no preview, no approval gate, or no send.

## Required input page: outreachProfile

Sender identity lives on a workflow-level `inputPages` entry with `contextKey: "outreachProfile"`. Without this page, the `{{context.outreachProfile.fromEmail}}` template resolves to empty and the email has no sender.

```json
{
  "title": "Outreach Profile",
  "pathname": "outreach-profile",
  "configuration": {
    "contextKey": "outreachProfile",
    "shortDescriptionFields": ["name", "fromEmail"],
    "fields": [
      { "name": "name", "label": "Sender name", "type": "text", "required": true },
      { "name": "fromEmailLabel", "label": "From name", "type": "text", "required": true },
      { "name": "fromEmail", "label": "From email", "type": "connected_emails_selector_multiple", "required": true },
      { "name": "replyToEmail", "label": "Reply-to (optional)", "type": "text" }
    ]
  }
}
```

## Full composed-email step (copy-paste)

```json
{
  "id": "send-outreach-email",
  "type": "aiAction",
  "name": "Send Outreach Email",
  "pipelineStepPrompt": {
    "type": "email",
    "template": "Draft a personalized email to {{steps.enrich.contact.fullName}} at {{steps.enrich.company.name}} introducing our offering. Keep it warm, concise, and specific to their role.",
    "responseStructure": {
      "email": {
        "from": "{{context.outreachProfile.fromEmail}}",
        "to": "{{steps.enrich.contact.email}}",
        "subject": "",
        "body": "",
        "bodyType": "html"
      }
    }
  },
  "renderer": {
    "type": "Email",
    "config": { "fromContextKey": "outreachProfile" }
  },
  "integrations": [
    {
      "type": "oneOf",
      "label": "Email",
      "connectorType": "email",
      "options": [
        { "name": "Gmail", "url": "https://gmail.com", "isUserAccountConnectionRequired": true },
        { "name": "Outlook", "url": "https://outlook.com", "isUserAccountConnectionRequired": true }
      ],
      "selectionHint": "preferConnected"
    }
  ],
  "onApproval": {
    "executedText": "Email sent by {{name}} at {{date}}",
    "failedText": "Email failed to send.",
    "action": "schedule-email"
  },
  "creditCost": 5,
  "next": { "stepId": "done", "conditions": { "approvalRequired": true } }
}
```

## Anti-patterns

**Draft + Gmail send (no approval):**

```
❌ draft-email (aiAction) → gmail.send-email (appAction) → done
```

No preview, no user approval, and the drafted body is written as a plain string instead of the structured `email` object — Gmail's send-email action expects specific fields (`to`, `subject`, `html`) that the draft prompt has no way to produce reliably.

**Missing `pipelineStepPrompt.type: "email"`:**

```yaml
pipelineStepPrompt:
  template: "..."
  responseStructure:
    email: { from, to, subject, body, bodyType }
# ❌ No `type: "email"` — the renderer falls back to a generic JSON view,
# and the orchestrator treats this as a regular aiAction (no schedule-email).
```

**Email body as plain text when `bodyType: "html"`:**

Email clients render HTML bodies literally if the wrapping element is missing. Always generate email-safe HTML (`<p>`, `<br>`, `<a>`, `<strong>` — no CSS blocks, no `<script>`). Don't use markdown — the approval UI won't convert it.

**No `outreachProfile` input page:**

`{{context.outreachProfile.fromEmail}}` resolves to an empty string. The email has no sender. It may still send (from the default connected account) but you lose per-workflow sender control.

## Checklist

Before wiring a composed-email step into a workflow:

- [ ] `context.inputPages[]` contains the `outreachProfile` page (pattern above)
- [ ] `pipelineStepPrompt.type: "email"` is set
- [ ] `renderer: { type: "Email", config: { fromContextKey: "outreachProfile" } }` is present
- [ ] `integrations[]` declares the email connector (Gmail / Outlook)
- [ ] `onApproval.action: "schedule-email"` is set
- [ ] `next.conditions.approvalRequired: true` on the outgoing edge
- [ ] Recipient and subject are derived from prior step output, not hardcoded

## When to use a generic approval gate instead

For non-email approvals (sending a Slack post, firing a webhook), use a plain `aiAction` + `next.conditions.approvalRequired: true` without the email renderer. The approval gate is a general-purpose feature; the email shape is only required when the approved artifact is an email that needs to be sent through the user's connected mailbox.
