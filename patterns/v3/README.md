# v3 — Anti-patterns library

The negative space is where production knowledge lives. These are planned for v3.

| File | Anti-pattern |
|------|-------------|
| `anti-pattern-prompt-in-loop.md` | Calling LLM once per iteration when a batch prompt works |
| `anti-pattern-fire-and-forget.md` | Async steps with no completion signal |
| `anti-pattern-god-workflow.md` | 30-step monoliths vs composable child workflows |
| `anti-pattern-llm-as-router.md` | Using AI for binary decisions a simple condition handles for free |
| `anti-pattern-missing-dedup.md` | Running polling workflows without a dedup gate (the cost math) |
| `anti-pattern-event-for-intake.md` | Using app_event triggers where polling + label dedup is correct |

Want to contribute one? Open a [new pattern issue](../../.github/ISSUE_TEMPLATE/new-pattern.md).
