# agentic-ops — Production patterns for agentic workflows

You have the agentic-ops skill loaded. When building, debugging, or reviewing agentic workflows, apply these patterns. They encode hard-won production knowledge — not theory.

## Core rules (apply always)

1. **Trigger default**: For email/document intake → Schedule trigger + label-based dedup. Only use event triggers when latency < 1 min is explicitly required.
2. **Label IDs, not names**: Gmail API requires label IDs (`Label_XXXXXXXXXX`), not display names. Always resolve via a create-or-get-label step first.
3. **Retry, don't restart**: When a workflow step fails, retry from that step. Never start a full new execution to debug — it wastes credits and reprocesses already-done work.
4. **Dedup everything**: Without a dedup gate, any polling or webhook workflow will reprocess records. Label dedup for email; idempotency keys for webhooks.
5. **Typed return contracts**: Child workflows use `return` step (not `milestone`) and must declare their output fields explicitly. The calling workflow depends on this contract.
6. **Conditions field names**: Use `criteria` (not `conditions`) and `variable` (not `field`). Wrong names silently skip steps with no error.
7. **Loop completion**: Always wait for loop completion before consuming loop results. Use a `loop_completion` entry condition on the step that follows a loop.

## When to use which error behavior

| Scenario | Use |
|---|---|
| Optional data — step can be skipped if missing | `onCriteriaFail: "skip"` |
| Hard prerequisite — nothing downstream is valid without it | `onCriteriaFail: "stop"` |
| Async dependency — waiting for a loop or group to finish | `onCriteriaFail: "wait"` |

## Trigger decision (fast lookup)

| User says | Trigger |
|---|---|
| "process inbound emails", "triage inbox", "review pitches" | Schedule (daily) |
| "every Monday", "daily at 9am", "weekly digest" | Schedule |
| "as soon as", "real-time", "within 30 seconds" | App event |
| "when a form is submitted", "on new webhook" | Webhook |

## Gmail label dedup (copy-paste pattern)

```
Step 1: GMAIL_CREATE_LABEL { name: "processed" }        → returns { id: "Label_XXXX" }
Step 2: GMAIL_FETCH_EMAILS { query: "-label:processed newer_than:1d" }
Step 3: Loop over messages → process each
Step 4: GMAIL_ADD_LABEL { message_id: "{{item.id}}", label_id: "{{steps.step1.id}}" }
```

**Never**: `label_id: "processed"` — display names cause `400: Invalid label`.

## Credit efficiency rules

- Fix the issue → retry from the failed step → verify. One cycle.
- Use `test_ai_action` / `test_code_action` to test steps in isolation before wiring them.
- Mock downstream steps with saved output from prior runs — don't re-run expensive upstream steps.

## Full pattern library

### v1 — Core
- [00-why-agentic-ops](patterns/v1/00-why-agentic-ops.md)
- [01-trigger-design](patterns/v1/01-trigger-design.md)
- [02-dedup-gates](patterns/v1/02-dedup-gates.md)
- [03-credit-efficiency](patterns/v1/03-credit-efficiency.md)
- [04-loop-patterns](patterns/v1/04-loop-patterns.md)
- [05-child-workflow-contracts](patterns/v1/05-child-workflow-contracts.md)
- [06-conditional-routing](patterns/v1/06-conditional-routing.md)
- [07-error-handling](patterns/v1/07-error-handling.md)
- [08-composed-email-approval](patterns/v1/08-composed-email-approval.md)
- [09-reports-and-knowledge-storage](patterns/v1/09-reports-and-knowledge-storage.md)
- [10-person-research-ladder](patterns/v1/10-person-research-ladder.md)
- [11-company-research-ladder](patterns/v1/11-company-research-ladder.md)

### v2 — Production hardening
- [10-observability](patterns/v2/10-observability.md)
- [11-human-in-the-loop](patterns/v2/11-human-in-the-loop.md)
- [12-idempotency](patterns/v2/12-idempotency.md)
- [13-multi-agent-handoff](patterns/v2/13-multi-agent-handoff.md)
- [14-secret-and-credential-management](patterns/v2/14-secret-and-credential-management.md)

### v3 — Anti-patterns
- [anti-pattern-prompt-in-loop](patterns/v3/anti-pattern-prompt-in-loop.md)
- [anti-pattern-fire-and-forget](patterns/v3/anti-pattern-fire-and-forget.md)
- [anti-pattern-god-workflow](patterns/v3/anti-pattern-god-workflow.md)
- [anti-pattern-llm-as-router](patterns/v3/anti-pattern-llm-as-router.md)
- [anti-pattern-missing-dedup](patterns/v3/anti-pattern-missing-dedup.md)
- [anti-pattern-event-for-intake](patterns/v3/anti-pattern-event-for-intake.md)
