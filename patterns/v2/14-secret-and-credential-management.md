# 14 — Secret and credential management: scoping, rotation, and not leaking

**Problem**: Workflows accumulate API keys, OAuth tokens, and webhook secrets in places they shouldn't live — workflow JSON, prompt templates, exported configs, version control. Rotation is a manual scavenger hunt; a leaked key is silently still valid for months because no one knows where it's referenced.

**Why it fails silently**: Credentials in a workflow definition look like config — they paste cleanly, they work on first run, and they never throw an error. The failure modes (key in a public repo, key shared across users who shouldn't share, expired token in a paused workflow) don't surface as workflow errors. They surface as breaches and as customer support tickets six weeks later asking "why did this stop running".

---

## Three scoping levels, picked deliberately

| Scope | Who can use the credential | When to use |
|---|---|---|
| **Workspace** | All workflows and users in the workspace | Shared service accounts (org-wide CRM, shared email) |
| **User** | Only workflows run as / on behalf of one user | Per-user OAuth (the user's Gmail, the user's Google Calendar) |
| **Workflow** | Only this one workflow | Service-specific keys with limited blast radius |

Most leaks happen because the wrong scope was chosen for convenience. A user-scoped Gmail token used at workspace scope means every workflow run can read every user's inbox. A workflow-scoped key set at workspace scope means it appears in every other workflow's autocomplete.

**Default to the narrowest scope that makes the workflow run.** Promote to a wider scope only when there's a concrete reason.

---

## Anti-pattern

```yaml
- id: send-email
  action: gmail.send-email
  input:
    api_key: "ya29.a0AfH6SMC..."           # ❌ raw token in workflow JSON
    body: "Hello {{steps.enrich.name}}"
```

Problems compound:
- Token is checked into version control via the workflow definition
- Anyone with read access to the workflow has the token
- Rotation requires editing every workflow that references it
- Audit log shows zero usage information per workflow

---

## Correct pattern

```yaml
- id: send-email
  action: gmail.send-email
  credential:
    scope: user
    ref: connected_email                   # references a connection, not a value
  input:
    body: "Hello {{steps.enrich.name}}"
```

The workflow references the credential **by name and scope**. The runtime resolves the reference at execution time against an encrypted credential store. The token never appears in the workflow JSON, never in exports, never in logs.

Three properties this gives you:

1. **Rotation in one place** — replace the credential, all workflows using `connected_email` immediately use the new value.
2. **Per-user isolation** — when User A runs the workflow, User A's Gmail is used; User B's token is never accessible.
3. **Auditability** — the credential store logs every resolution: which workflow, which user, when.

---

## OAuth tokens: refresh is mandatory, not optional

OAuth access tokens expire. If your workflow stores only the access token, it will silently start failing when the token rotates. You must:

1. Store the **refresh token** alongside (encrypted at rest).
2. The credential resolver checks expiry on every read; if expired, it refreshes and writes back.
3. If the refresh fails (revoked, scope changed, user disconnected), the next workflow run fails with a clear "credential needs reconnection" — not an opaque 401.

```
read credential → if expires_at < now + 5min → refresh → return access_token
                                              → on refresh fail → mark needs_reconnect → fail step
```

A workflow that ran fine for 90 days and then "mysteriously stops" is almost always an unhandled refresh-token failure.

---

## Webhook secrets: verify, don't trust

Inbound webhooks (Stripe events, GitHub events, custom integrations) include a signature header. **Verifying it is non-optional.** An unverified webhook endpoint is an internet-exposed RCE shaped like a workflow trigger.

```yaml
trigger:
  type: webhook
  verify:
    method: hmac_sha256
    header: X-Stripe-Signature
    secret_ref: stripe_webhook_secret
    on_invalid: reject
```

If your orchestrator doesn't have first-class signature verification, write a verification step before any other workflow logic. Reject with 401 on invalid signature; never let the workflow proceed.

---

## Anti-pattern: secrets in prompt templates

```
prompt: "Use API key sk-abc123 to call OpenAI..."
```

Anything in the prompt is in the LLM call's training-leakable surface area. Even if the model doesn't memorize, the call is logged by the LLM provider and may appear in their backend dashboards.

**Secrets must never appear in prompt templates, AI step inputs, or anywhere the model can echo them back.** If the AI step needs to call an API, the runtime injects the credential into the tool call — the model sees a tool description, not the key.

---

## Anti-pattern: shared workspace credential for "convenience"

```
credential: workspace_default_gmail
```

When User A runs a workflow that "checks the team inbox", does it use User A's view, the inbox owner's view, or the workspace service account? If you can't answer this in one sentence, the credential is over-scoped.

A good rule: any credential that touches user-private data (inbox, calendar, drive, DMs) must be **user-scoped**. Workspace-scoped credentials are appropriate only for shared service accounts (org-wide CRM, public website analytics, shared payment processor).

---

## Rotation as a regular operation

Schedule rotation, don't react to it. For each credential, log:

| Credential | Scope | Last rotated | Rotation cadence | Next due |
|---|---|---|---|---|
| `stripe_secret_key` | workspace | 2026-01-15 | 90d | 2026-04-15 |
| `gmail_oauth_user_x` | user | auto (refresh) | n/a | n/a |
| `webhook_signing_secret` | workflow | 2025-12-01 | 180d | 2026-05-30 |

Rotation that's never scheduled is rotation that only happens after a leak. The cost of rotation is bounded; the cost of a leaked key in active use is unbounded.

---

## Detection: a key was leaked, what now?

1. **Revoke at the provider** first (Stripe dashboard, GitHub token settings) — buys you immediate safety.
2. **Search the credential reference** in your workflow store — every workflow using the rotated credential will fail next run with "credential needs reconnection". That's the signal to update.
3. **Audit-log review** — pull the credential's resolution log for the last 30 days. Anything unexpected (new IP, new workflow, off-hours run) is a finding.

If your runtime can't do step 2 (because keys are inlined in workflow JSON), the leak triages into a manual repo grep across every workflow you've ever written. Don't end up here.

---

## One-line rule

> Reference credentials by name and narrowest-possible scope; never inline secrets in workflow JSON, prompts, or logs, and treat OAuth refresh and webhook signature verification as required, not optional.
