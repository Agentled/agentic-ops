# v2 patterns

| File | Pattern | One-line rule |
|------|---------|---------------|
| [10-observability](10-observability.md) | Structured logging, execution tracing, alerting on silent failures | Every workflow must declare a business-outcome assertion and emit a structured count signal at every fetch, loop, and write. |
| [11-human-in-the-loop](11-human-in-the-loop.md) | Approval gates, async review, timeout and escalation | Every gate must have notification, preview, timeout, and escalation; route to a role not a person, default to the safe action on timeout. |
| [12-idempotency](12-idempotency.md) | Safe retries, dedup keys, exactly-once at step level | Every external side-effect step needs a deterministic idempotency key derived from execution + inputs, never from time. |
| [13-multi-agent-handoff](13-multi-agent-handoff.md) | Passing context between agents without prompt drift | Pass typed structured payloads between agents and always include the original input as an immutable reference. |
| [14-secret-and-credential-management](14-secret-and-credential-management.md) | Env vars, rotation, per-user vs per-workspace scoping | Reference credentials by name and narrowest-possible scope; never inline secrets in workflow JSON, prompts, or logs. |

Want to contribute a pattern? Open a [new pattern issue](../../.github/ISSUE_TEMPLATE/new-pattern.md).
