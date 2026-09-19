# Failure modes

Symptom -> cause -> fix.

## Quality dropped mid-session, nobody changed model

Cause: safety classifier flagged a request. Claude Code swapped model and stayed there. Cyber-flagged -> Opus 4.8. Bio-flagged -> Opus 5.

Check: `/model` shows current model. `/usage` shows per-model token split.

Fix: `/model fable` to return. Prevent: settings `"switchModelsOnFlag": false` -> flagged request pauses, you choose switch or edit prompt.

## Fell back on the very first message

Cause: first request carries CLAUDE.md, skills, git status, directory names. Security-tooling repo, pentest notes, or bio data in the workspace trips it before you type.

Diagnose: `claude --safe-mode` disables CLAUDE.md, skills, MCP, hooks. Still fell back -> git status / dir names. Worked -> a customization. Bisect.

Fix: move offending material out of CLAUDE.md into an on-demand skill; rename dirs; or accept Opus for that repo.

## Refusal with no obvious cyber/bio content

Cause candidates (from Anthropic migration guidance): prompt asks to expose raw reasoning (API refusal category `reasoning_extraction`), compile-check phrasing ("does this compile without errors" - ask "are there bugs" instead), obscure language with no context, tool returning base64 blobs into context.

Fix: audit prompts for reasoning-echo requests; rephrase; give language docs; strip base64 tools.

## Fable over-plans, re-derives, surveys options

Cause: ambiguous task at high effort, or old scaffolding prompts. Fix: act-don't-overplan block. Give the why. Drop effort one level.

## Unrequested refactors, extra tests, nearby bug fixes

Cause: higher effort + open-ended task. Fix: no-tidying block + scope/test-coverage block. Verification scripts outside the repo.

## Turn ends with "I'll now run X" and nothing runs

Cause: early-stop behavior deep into long sessions. Fix: autonomous-run block for unattended; reply `continue` interactively; `/goal` for anything with a checkable end state.

## Fable proposes a new session, trims its own work

Cause: context anxiety, usually triggered by a visible remaining-token count. Fix: don't surface countdowns; context-anxiety block.

## Subagents all running on Fable

Cause: unpinned subagent inherits main model. Fix: `CLAUDE_CODE_SUBAGENT_MODEL=opus` in env, or `model:` in every agent file. Check `/tasks` - shows model and effort per agent.

## One subagent cost as much as the whole session

Cause: it was a fork (`subagent_type: "fork"`). A fork inherits the full conversation and always runs on the parent model; `CLAUDE_CODE_SUBAGENT_MODEL` and `_FORCE` do not apply. On a Fable main that is a Fable cache write of the entire transcript plus Fable output. Fix: fork only when the task needs the whole conversation; otherwise spawn a named lane with a self-contained brief. Check `/usage` for a Fable cache-write spike right after the spawn.

## Builder rewrote a 2000-line file to change 3 lines

Cause: 5.1 leans to whole-file rewrites. Fix: targeted-edits line.

## Fable waits on every subagent before continuing

Cause: spawn-and-block habit. Fix: delegate-async block. Name agents, resume with `SendMessage` instead of respawning.

## Advisor never fires

Cause: pairing rejected - Fable main with Opus/Sonnet advisor; or `DISABLE_TELEMETRY`-class env var blocks feature flags; or org `availableModels` excludes it; or Fable usage-credit consent not given yet (`/model fable` once to accept).

## Subagent file ignored

Cause: no `name`, no `description`, `---` not first line, `:` in name, bad YAML, or first file in a brand-new agents dir (needs restart). Check `claude plugin validate .claude/agents`.

## Named subagents keep turning into teammates

Cause: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Set to `0` in user settings; re-read on next spawn, no restart.

## Costs climb while idle

Cause: idle teammates, scheduled `/loop`, goal check-ins, cross-session messages, long uncleared context. Fix: shut teammates down, `/clear` between tasks, `CLAUDE_CODE_GOAL_CHECKIN_MINUTES=0` if check-ins unwanted.

## Cache misses on every subagent turn

Cause: subagent cache TTL 5 min; agents idle longer than that. Fix: `subagentPromptCacheTtl: "1h"`. `/usage` prompt-cache line names the likely cause of the last miss (v2.1.260+).

## Workflow agents landed on Fable

Cause: script didn't name a model per stage; session model is Fable. Fix: say lanes in the prompt or edit saved script.
