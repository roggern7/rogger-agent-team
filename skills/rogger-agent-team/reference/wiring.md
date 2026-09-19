# Wiring

Five ways to put Fable 5.1 in charge. Pick by section in SKILL.md, configure here.

## 1. Advisor mode

Experimental, Anthropic API only. Cheap model types, Fable steers at decision points (before committing to approach, on recurring errors, before declaring done). Fable reads the whole transcript each call - cost grows with session length.

```
/model fable         # once: accept usage-credit consent if your plan asks (else /advisor fable is refused)
/model sonnet        # or opus
/advisor fable
```

Persistent: `~/.claude/settings.json` -> `"advisorModel": "fable"`. Single session: `claude --model sonnet --advisor fable`.

Force a consult: `consult the advisor before you continue`. No setting caps or forces calls; say it in the prompt.

Subagents inherit the advisor and re-check pairing against their own model, so each subagent's consults re-read that subagent's transcript at Fable rate too. Main on Fable -> advisor must be Fable; an Opus advisor is dropped with a notification.

Cost model. Anthropic's guidance: a cheap main plus a stronger advisor "typically costs less than running the stronger model throughout", because consults happen at decision points, not every turn. The catch: each consult re-reads the whole transcript uncached at $10/MTok, while a Fable architect (pattern 2) reads its own context from cache at $0.25/MTok.

Per decision point, list price, both arms fully costed:

| Transcript | Advisor consult (read + ~1K advice) | Fable architect turn (cached read + ~2K output) |
|---|---|---|
| 20K | $0.25 | $0.105 |
| 100K | $1.05 | $0.125 |
| 300K | $3.05 | $0.175 |

Advisor mode also runs every non-consult turn on Sonnet instead of Fable, which is where its savings come from. So: advisor wins when consults are few and transcripts short; architect wins when Fable judgment is needed often or the session runs long. Neither number includes cache writes, which both arms pay. Cost test: consults x $10 x transcript MTok, vs the same decisions as Fable-architect turns at ~$0.10-0.20 each. One consult on a 100K transcript costs about eight architect turns. `/clear` between tasks keeps advisor mode cheap.

## 2. Architect + delegate

Fable main session. Plans, spawns lanes, verifies with fresh agent. Default for real builds.

```
/model fable
/effort high
```

Lanes ship with the plugin as `rogger-agent-team:<name>`. To customize, copy `agents/*.md` into `.claude/agents/` (project) or `~/.claude/agents/` (global); restart once after creating a new agents dir.

Subagent frontmatter that matters:

```yaml
---
name: opus-builder
description: ...            # Claude reads this to decide delegation. Short.
model: opus                 # sonnet | opus | haiku | fable | inherit | full ID
effort: high                # low | medium | high | xhigh | max
tools: Read, Edit, Write, Bash, Grep, Glob   # allowlist; omit = inherit all
disallowedTools: Agent      # denylist, applied first
maxTurns: 60                # partial output marked incomplete when hit
isolation: worktree         # own git worktree; no file conflicts between parallel builders
memory: project             # .claude/agent-memory/<name>/ persists across sessions
background: true            # stay async even if caller wants foreground
permissionMode: acceptEdits # ignored for plugin-shipped agents (also hooks, mcpServers); works from .claude/agents/
experimental:
  cacheTtl: 1h              # per-agent cache lifetime (v2.1.248+); ignored while drawing on usage credits
---
```

Parallel builders on one repo: add `isolation: worktree` to `opus-builder` so each runs in its own git worktree and merges back.

Read-only lanes (`haiku-scout`, `fresh-verifier`, `fable-architect`) are read-only by prompt; they still hold `Bash`. To enforce, add a permission deny rule for write commands or copy them to `.claude/agents/` with `permissionMode: plan`.

Listing `Agent` in a subagent's `tools` lets it spawn subagents; omitting the whole `tools:` key inherits everything including `Agent`; a `tools:` list without `Agent`, or `disallowedTools: Agent`, blocks spawning. The `Agent(type, type)` allowlist is documented only for an agent running as the main thread via `claude --agent`; don't rely on it inside a subagent file.

Env:

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "opus",
    "CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS": "20",
    "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH": "2"
  },
  "promptCacheTtl": "1h",
  "subagentPromptCacheTtl": "1h"
}
```

`CLAUDE_CODE_SUBAGENT_MODEL` = default for unpinned subagents (else they inherit Fable). `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` (v2.1.257+) overrides even pinned ones except forks and `model: inherit` skills. Defaults: concurrency 20, depth 3 (example config sets depth 2).

Async pattern: Fable spawns builders, keeps planning next phase, receives results as they land. Resume a finished agent with `SendMessage` by name - full history kept, no re-establishing context. Named long-lived agents + 1h cache TTL beats spawn-per-subtask.

Validate agent files: `claude plugin validate .claude/agents`. Watch running agents + their model/effort: `/tasks`.

## 3. Agent teams

Peers with own context windows, shared task list, direct messaging. Experimental.

```json
{ "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" } }
```

Prompt shape:

```
Spawn three teammates on Sonnet: one investigates hypothesis A, one B, one C.
Have them try to disprove each other. Write consensus to FINDINGS.md.
```

Model per teammate: name it in the spawn prompt, else definition `model:`, else `CLAUDE_CODE_SUBAGENT_MODEL`, else lead model. Teammates inherit lead effort. Reuse `agents/` definitions by name: `Spawn a teammate using the fresh-verifier agent type`.

Rules: 3-5 teammates. Each owns different files. Shut down when done - idle teammates keep costing. ~7x tokens of a solo session in plan mode. No nested teams. In-process teammates can't spawn background subagents. Interactive sessions only: under `-p` or the SDK, named subagents run as plain subagents. With teams on, any subagent Fable names becomes a teammate - set env to `0` for plain delegation.

## 4. Workflows

Script orchestrates dozens-hundreds of agents; Fable context holds only the final result. Resumable. Saveable as `/command`.

Trigger: `use a workflow to ...` or `ultracode: ...` in a prompt you type. Not from `-p`, the SDK, scheduled tasks, or relayed comments - for unattended runs save the workflow first and invoke `/<name>`. Session-wide: `/effort ultracode` (xhigh + auto workflows for everything - expensive, drop back with `/effort high`). Pro plans enable workflows in `/config`; `disableWorkflows` / `CLAUDE_CODE_DISABLE_WORKFLOWS=1` turn them off.

Good fits: audit N files for same issue, migrate a tree in isolated copies, fix-until-check-passes, per-file review then merge findings, cross-checked research (`/deep-research` bundled).

Limits: 16 concurrent agents, 1000 per run, 4096 items per `parallel()` / `pipeline()`. No mid-run user input. Size guideline `workflowSizeGuideline`: `small` (<5), `medium` (<15, default), `large` (<50), `unrestricted`. A settings-file value overrides `/config`. Warning at over 25 agents or 1.5M projected tokens.

Model per stage: ask in prompt ("use Sonnet for the per-file pass, Opus for the merge"). Unassigned agents run on session model - on Fable, that's Fable. Say the lane.

Save: `/workflows` -> select -> `s` -> `.claude/workflows/` (shared) or `~/.claude/workflows/` (personal). Edit with `/workflow-authoring` skill loaded, then `/reload-skills`.

## 5. /goal

Completion condition evaluated by Haiku after every turn. Fable keeps going until met, judged impossible, or unrecoverable error.

```
/goal npm test exits 0 and tsc --noEmit is clean; no test file outside src/auth modified; or stop after 25 turns
```

Condition needs: one measurable end state, the check command, constraints, a stop clause. Up to 4000 chars. Evaluator only sees what Fable surfaced in transcript - Fable must run the check and show output.

Unattended: run in auto permission mode (`claude --permission-mode auto`; a classifier approves tool calls) or permission prompts block. Leave `switchModelsOnFlag` unset (default `true`) for unattended runs - `false` pauses on a flag and nobody is there to answer. The shipped `settings.example.json` leaves it unset for this reason; set it `false` yourself for interactive work. Background subagents defer evaluation; check-ins every 30 min, backing off. Headless: `claude -p "/goal ..." --output-format stream-json --verbose`.

`/goal` = status. `/goal clear` = stop. Survives `--resume`.

## Combining

- Build: 2 + 5. Fable architect, lanes build, goal holds it to green.
- Bulk: 4 with Sonnet stages, Fable session only reads final report.
- Cheap steering: 1 with Sonnet or Opus main. Escalate single hard steps to `fable-architect` subagent (plan only, returns text). Pointless from a Fable main - it already is the architect.
- Debug unknown cause: 3 with Sonnet peers arguing, Fable lead adjudicates.
