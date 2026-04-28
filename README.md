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
| [08-composed-email-approval](patterns/v1/08-composed-email-approval.md) | Composed email with approval gate | One AI step generates + sends; never separate draft + send actions. |
| [09-reports-and-knowledge-storage](patterns/v1/09-reports-and-knowledge-storage.md) | Report rendering + KG persistence | Always render reports via a config layout; persist structured results to the KG. |
| [10-person-research-ladder](patterns/v1/10-person-research-ladder.md) | Person research: signal-based lookup + fallback ladder | Pick the lookup by the strongest input signal; fall down the ladder, not across. |
| [11-company-research-ladder](patterns/v1/11-company-research-ladder.md) | Company research: match source to question | LinkedIn for people, website for positioning, directories for financials. |

### v2 — Production hardening

| File | Pattern | One-line rule |
|------|---------|---------------|
| [10-observability](patterns/v2/10-observability.md) | Structured logging, execution tracing, alerting on silent failures | Declare a business-outcome assertion and emit count signals at every fetch, loop, and write. |
| [11-human-in-the-loop](patterns/v2/11-human-in-the-loop.md) | Approval gates, async review, timeout and escalation | Every gate needs notification, preview, timeout, and escalation; route to a role, default to the safe action. |
| [12-idempotency](patterns/v2/12-idempotency.md) | Safe retries, dedup keys, exactly-once at step level | Every side-effect step needs a deterministic idempotency key derived from execution + inputs. |
| [13-multi-agent-handoff](patterns/v2/13-multi-agent-handoff.md) | Passing context between agents without prompt drift | Pass typed structured payloads and always include the original input as an immutable reference. |
| [14-secret-and-credential-management](patterns/v2/14-secret-and-credential-management.md) | Env vars, rotation, per-user vs per-workspace scoping | Reference credentials by name and narrowest scope; never inline secrets in workflow JSON or prompts. |

### v3 — Anti-patterns library

| File | Anti-pattern | One-line rule |
|------|-------------|---------------|
| [anti-pattern-prompt-in-loop](patterns/v3/anti-pattern-prompt-in-loop.md) | Calling LLM per iteration when batch works | Default to a single batch prompt with array output; loop only for distinct per-item tool calls. |
| [anti-pattern-fire-and-forget](patterns/v3/anti-pattern-fire-and-forget.md) | Async steps with no completion signal | Async dispatch is not async completion; wait on the completion event, not the dispatch. |
| [anti-pattern-god-workflow](patterns/v3/anti-pattern-god-workflow.md) | 30-step monoliths vs composable child workflows | If a workflow has > ~15 steps or reusable slices, split into orchestrator + typed children. |
| [anti-pattern-llm-as-router](patterns/v3/anti-pattern-llm-as-router.md) | AI for binary decisions a condition handles for free | Use AI for structured output, use conditions to branch on it. |
| [anti-pattern-missing-dedup](patterns/v3/anti-pattern-missing-dedup.md) | Polling workflows without a dedup gate (the cost math) | Polling without dedup burns credits proportional to source size × poll frequency. |
| [anti-pattern-event-for-intake](patterns/v3/anti-pattern-event-for-intake.md) | App-event triggers where polling + label dedup is correct | Default to scheduled polling with label-based dedup for email/document intake. |

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
