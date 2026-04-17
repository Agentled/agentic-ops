# agentic-ops

**Production patterns for agentic workflows — the ops discipline AI agents are missing.**

Maintained by the team at [Agentled.ai](https://agentled.ai) — we built this because we kept explaining the same hard-won patterns to every developer who hit the wall. All content is platform-agnostic. PRs from any platform welcome.

---

## Install as a Claude Code skill

Add to your project's `CLAUDE.md`:

```markdown
# Load agentic-ops patterns
See: https://github.com/agentled/agentic-ops
```

Or reference directly in your Claude Code session:

```bash
# Fetch the skill and append to your CLAUDE.md
curl -sL https://raw.githubusercontent.com/agentled/agentic-ops/main/CLAUDE.md >> CLAUDE.md
```

---

## Quick reference — the decision you'll make most often

**Should I use a scheduled trigger or a real-time event trigger for email intake?**

| | Schedule (polling) | App Event (real-time) |
|---|---|---|
| Latency | minutes–hours | seconds |
| Idempotency | trivial — label marks processed | must dedupe on messageId; re-deliveries happen |
| Backfill | widen the query window | doesn't exist |
| Replay after outage | automatic on next run | events can be permanently lost |
| Debugging | read last execution log | subscription + delivery + filter + dedupe all need checking |

**Rule: default to Schedule + label-based dedup for email/document intake. Use an event trigger only when the user explicitly needs < 1 minute latency.**

Copy-paste Gmail polling query: `-label:processed newer_than:1d`

---

## Patterns

### v1 — Core

| File | Pattern | One-line rule |
|------|---------|---------------|
| [00-why-agentic-ops](patterns/v1/00-why-agentic-ops.md) | Why structured workflows beat ad-hoc prompting | Ad-hoc prompting doesn't scale; agentic-ops is the ops layer that makes AI agents production-ready. |
| [01-trigger-design](patterns/v1/01-trigger-design.md) | Polling vs event triggers | Default to schedule for intake; use events only when latency < 1 min is a hard requirement. |
| [02-dedup-gates](patterns/v1/02-dedup-gates.md) | Idempotency and dedup | Always resolve label IDs before use — never pass display names to Gmail API. |
| [03-credit-efficiency](patterns/v1/03-credit-efficiency.md) | Not burning money while debugging | Fix → retry from failed step → verify. Never start a new execution to debug a failed one. |
| [04-loop-patterns](patterns/v1/04-loop-patterns.md) | Iterating without N+1 or data loss | Always wait for loop completion before consuming results; never assume order. |
| [05-child-workflow-contracts](patterns/v1/05-child-workflow-contracts.md) | Composable workflows with typed return contracts | Child workflows use `return`, not `milestone`; always define a typed return contract. |
| [06-conditional-routing](patterns/v1/06-conditional-routing.md) | Conditions that actually fire | Use `criteria`/`variable` (not `conditions`/`field`) — wrong field names silently skip steps. |
| [07-error-handling](patterns/v1/07-error-handling.md) | skip vs stop vs wait | `skip` for optional data, `stop` for hard prerequisites, `wait` for async completion. |

### v2 — Coming soon

- `08-observability` — Structured logging, execution tracing, alerting on silent failures
- `09-human-in-the-loop` — Approval gates, async review, timeout and escalation
- `10-idempotency` — Safe retries, dedup keys, exactly-once guarantees at step level
- `11-multi-agent-handoff` — Passing context between agents without data loss
- `12-secret-and-credential-management` — Env vars, rotation, per-user vs per-workspace scoping

### v3 — Anti-patterns library

- `anti-pattern-prompt-in-loop` — Calling LLM per iteration when batch works
- `anti-pattern-fire-and-forget` — Async steps with no completion signal
- `anti-pattern-god-workflow` — 30-step monoliths vs composable child workflows
- `anti-pattern-llm-as-router` — AI for binary decisions a condition handles for free

---

## Pattern format

Every pattern follows the same structure so they're fast to scan:

```
## Pattern Name

**Problem**: One sentence — what goes wrong without this.
**Why it fails silently**: The specific failure mode developers don't see coming.

### Anti-pattern
[the wrong way]

### Correct pattern
[the right way]

### One-line rule
> Always X, never Y. Reason in one clause.
```

---

## Contributing

Patterns are most valuable when they come from real production failures — not theoretical advice.

1. Open an issue using the [new pattern template](.github/ISSUE_TEMPLATE/new-pattern.md)
2. Or submit a PR adding a file to `patterns/v1/` following the format above
3. Patterns must include a "why it fails silently" section — that's the hard-won knowledge

See [MAINTAINERS.md](MAINTAINERS.md) for the people who review PRs.

---

## Feedback

Hit a pattern that didn't work? Found something missing?
→ [Open a feedback issue](.github/ISSUE_TEMPLATE/feedback.md) — it directly shapes what gets built in v2.

---

## License

MIT — use freely, attribution appreciated.
