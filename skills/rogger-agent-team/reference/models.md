# Models (checked 2026-09-09)

## Claude Fable 5.1

- ID `claude-fable-5-1`. Alias `fable`. Released September 2026. Successor to Fable 5 (`claude-fable-5`, still served).
- 1M context (default and max). 128K max output. Knowledge cutoff June 2026.
- Thinking always on, adaptive. Cannot disable. Effort `low` / `medium` / `high` / `xhigh` / `max` is the only depth control.
- $10 input / $50 output per MTok. Cache read $0.25 per MTok, down from $1 on Fable 5 (75% cut, per the Anthropic announcement). That is 2.5% of input price; Opus 5 and Sonnet 5 cache reads stay at the standard 10% ($0.50 and $0.20). Anthropic: ~25% cheaper than Fable 5 on typical workloads, up to ~45% on heavy agentic loops.
- Terminal-Bench 4.0 at high effort: 55.8% (Fable 5: 42.0%, Opus 5: 52.3%).
- Claude Code >= v2.1.255. Not default on any plan. Select with `/model fable` or `claude --model fable`. `best` alias = Fable where available, else Opus.
- Some plans bill Fable to usage credits. Interactive sessions show a one-time consent prompt; until accepted, `/advisor fable` is refused. `-p` / SDK sessions bill without asking (model-config doc). While drawing on usage credits the main-session cache TTL drops from 1h to 5m unless `promptCacheTtl: "1h"` is set.
- Requires 30-day data retention; zero-data-retention orgs get `400 invalid_request_error` unless Anthropic expressly authorizes ZDR (Anthropic migration guidance; https://code.claude.com/docs/en/zero-data-retention#model-availability-under-zdr).
- Safety classifiers on cyber and bio content. Fewer false positives than Fable 5 (~60% fewer cyber interventions, ~85% fewer bio false positives). Vulnerability discovery in source allowed; exploit generation not.
- Mythos 5.1 = same model, fewer safeguards, restricted access programs only.

## Cheaper lanes

| Model | ID | Alias | $ in/out | Context | Effort levels |
|---|---|---|---|---|---|
| Opus 5 | `claude-opus-5` | `opus` | 5 / 25 | 1M | low-max |
| Opus 4.8 | `claude-opus-4-8` | pin via `ANTHROPIC_DEFAULT_OPUS_MODEL` | 5 / 25 | 1M | low-max (legacy; cyber fallback target) |
| Sonnet 5 | `claude-sonnet-5` | `sonnet` | 2 / 10 | 1M | low-max |
| Haiku 4.5 | `claude-haiku-4-5` (dated: `claude-haiku-4-5-20251001`) | `haiku` | 1 / 5 | 200K | fixed thinking budget, no effort |

Opus 5: thinking on by default. Opus 4.8: thinking off unless set. Sonnet 5 uses the current tokenizer (introduced with Opus 4.7): roughly a third more tokens than 4.6-generation models, still cheaper per task.

## Fable 5.1 vs Fable 5 behavior deltas

Source: Anthropic's Fable 5.1 migration guidance, bundled in Claude Code as the `/claude-api` skill (`shared/model-migration.md`). Prompt-tunable, not API-breaking. Matters when migrating skills.

| Delta | Fix |
|---|---|
| Batches implied tool calls less in long loops | Nudge: "First privately list what you need next; then request every item that doesn't depend on another's result in this one response." |
| Narrates less between tool calls | Ask for a one-line preamble + standalone recap (prompt kit: progress updates) |
| Answers from memory more at `low` | Raise effort for turns naming products/models, or add name-verification line |
| Rewrites whole files where an edit would do | Targeted-edit line |
| Extra tests, fixes nearby bugs unasked | Scope + test-coverage block |
| Denser prose, fewer headers/bullets | Remove anti-formatting rules; add "remove mannered prose" if needed |
| Instruction given once persists better (reported, unconfirmed at launch) | Drop per-turn reminder repetition inherited from Fable 5 prompts and re-test |

## Effort semantics

Source: same migration guidance.

- Level names do not map to the same thinking across models. Re-sweep when changing model.
- Fable `low` often exceeds prior models at `xhigh`. Fable `medium` roughly matches Fable 5 at lower cost.
- `xhigh` / `max`: long deliverables get drafted in thinking then written again. ~2x output tokens. Use `high` unless measured.
- `ultracode` = valid `/effort` value: `xhigh` + automatic workflow orchestration for every substantive task. Session-wide. Expensive. Not valid in subagent `effort:`.
- Set per model in settings: `"modelSettings": {"claude-fable-5-1": {"effortLevel": "high"}}` (inner key is `effortLevel`). Settings keys accept `low`-`xhigh`; `max` is rejected there and only reachable via `/effort`. Subagent frontmatter `effort:` overrides. `${CLAUDE_EFFORT}` readable inside skills (skills doc).

## Advisor pairings

Advisor must be at least as capable as main model.

| Main | Accepted advisors |
|---|---|
| Haiku 4.5 | Fable, Opus, Sonnet |
| Sonnet 4.6 | Fable, Opus, Sonnet |
| Sonnet 5 | Fable, Opus, Sonnet 5 (Sonnet 4.6 rejected) |
| Opus 4.6 | Fable, Opus, Sonnet 5 |
| Opus 4.7 / 4.8 / 5 | Fable, Opus 4.7 or later |
| Fable 5.1 | Fable 5.1 only |

Opus or Sonnet advisor on a Fable main is not attached; `/advisor` output and a notification say so. Advisor reads the full transcript uncached on every call. Experimental; Anthropic API only (not Amazon Bedrock, Claude Platform on AWS, Google Cloud Agent Platform, Microsoft Foundry); needs feature-flag fetching (off under `DISABLE_TELEMETRY`). Toggling advisor does not break main-model prompt cache.

## Sources

Live check, 2026-09-09, Claude Code 2.1.266, Fable 5.1 main session: system-prompt overlap per prompt-kit block (`prompt-kit.md`), Agent tool schema (`fork` inherits parent model and full context; `isolation: worktree`; `SendMessage` resume; model enum `sonnet|opus|haiku|fable`), Workflow tool schema (`workflowSizeGuideline` medium = under 15 agents; triggers: `ultracode`, "use a workflow", skill instruction, named workflow), 1h main-session cache TTL dropping to 5m under usage overage. Subagent system prompt probed with a Haiku agent: none of the main-session behavior lines present.


- Anthropic announcement: https://www.anthropic.com/claude-fable-and-mythos-5-1
- Model overview + pricing: https://platform.claude.com/docs/en/about-claude/models/overview , https://platform.claude.com/docs/en/about-claude/pricing
- Claude Code model config (Fable section, fallback, effort, aliases): https://code.claude.com/docs/en/model-config
- Advisor: https://code.claude.com/docs/en/advisor
- Subagents: https://code.claude.com/docs/en/sub-agents
- Agent teams: https://code.claude.com/docs/en/agent-teams
- Workflows: https://code.claude.com/docs/en/workflows
- /goal: https://code.claude.com/docs/en/goal
- Costs: https://code.claude.com/docs/en/costs
- Behavior deltas, effort semantics, prompt snippets: Anthropic Fable 5.1 migration guidance, bundled in Claude Code (`/claude-api` skill, `shared/model-migration.md`)
