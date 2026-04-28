# v3 — Anti-patterns library

The negative space is where production knowledge lives.

| File | Anti-pattern | One-line rule |
|------|-------------|---------------|
| [anti-pattern-prompt-in-loop](anti-pattern-prompt-in-loop.md) | Calling LLM once per iteration when a batch prompt works | Default to a single batch prompt with array output; loop the LLM only when each item needs distinct tool calls. |
| [anti-pattern-fire-and-forget](anti-pattern-fire-and-forget.md) | Async steps with no completion signal | Async dispatch is not async completion; always wait on the completion event, not the dispatch. |
| [anti-pattern-god-workflow](anti-pattern-god-workflow.md) | 30-step monoliths vs composable child workflows | If a workflow has > ~15 steps or contains reusable slices, split into orchestrator + typed child workflows. |
| [anti-pattern-llm-as-router](anti-pattern-llm-as-router.md) | Using AI for binary decisions a condition handles for free | Use AI to produce structured output, use conditions to branch on it. |
| [anti-pattern-missing-dedup](anti-pattern-missing-dedup.md) | Running polling workflows without a dedup gate (the cost math) | Polling without dedup burns credits proportional to source size × poll frequency on work already done. |
| [anti-pattern-event-for-intake](anti-pattern-event-for-intake.md) | Using app_event triggers where polling + label dedup is correct | For email and document intake, default to scheduled polling with label-based dedup. |

Want to contribute an anti-pattern? Open a [new pattern issue](../../.github/ISSUE_TEMPLATE/new-pattern.md).
