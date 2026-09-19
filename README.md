<div align="center">

# rogger-agent-team

**Fable 5.1 decides. Opus 5 builds. Sonnet 5 grinds. Haiku scouts.**<br>
Claude Code plugin. Pay Fable rate for judgment only.

[![Claude Code](https://img.shields.io/badge/Claude_Code-%3E%3D_2.1.255-blueviolet)](https://code.claude.com/docs)
[![Model](https://img.shields.io/badge/Fable-5.1-8A2BE2)](https://www.anthropic.com/claude-fable-and-mythos-5-1)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Sources](https://img.shields.io/badge/sources-2026--09--09-informational)](skills/rogger-agent-team/reference/models.md#sources)

</div>

```mermaid
flowchart LR
    U([task]) --> F

    subgraph FABLE["Fable 5.1 · $10 / $50"]
        F[architect<br/>plan · decide · judge]
    end

    F -. spawn async .-> B & W & S
    B & W & S -- result --> F
    F -. verify .-> V
    V -- PASS / FAIL + evidence --> F
    F --> D([done])

    subgraph LANES["cheap lanes"]
        B[opus-builder<br/>$5 / $25]
        W[sonnet-worker<br/>$2 / $10]
        S[haiku-scout<br/>$1 / $5]
        V[fresh-verifier<br/>Opus, clean context]
    end

    classDef fable fill:#8A2BE2,color:#fff,stroke:none
    classDef lane fill:#2b2d42,color:#fff,stroke:none
    classDef io fill:#edf2f4,color:#2b2d42,stroke:#8d99ae
    class F fable
    class B,W,S,V lane
    class U,D io
```

## Install

Inside Claude Code:

```
/plugin marketplace add roggern7/rogger-agent-team
/plugin install rogger-agent-team@rogger-agent-team
```

Then merge [`settings/settings.example.json`](settings/settings.example.json) (this repo, or the installed copy under `~/.claude/plugins/marketplaces/rogger-agent-team/settings/`) into `~/.claude/settings.json`. Run `/model fable` per session when the task warrants it. Agents appear as `rogger-agent-team:opus-builder` etc.

## Pick wiring

```mermaid
flowchart TD
    Q1{unattended,<br/>checkable end state?} -- yes --> G["/goal &lt;check&gt; on top of the pattern below"]
    Q1 -- no --> Q2
    G --> Q2{same step over<br/>many files?}
    Q2 -- yes --> WF["workflow<br/><i>use a workflow to …</i>"]
    Q2 -- no --> Q3{peers must argue<br/>or own layers?}
    Q3 -- yes --> AT["agent teams<br/>Fable lead · Sonnet peers"]
    Q3 -- no --> Q4{few decisions,<br/>short session, /clear often?}
    Q4 -- yes --> AD["advisor mode<br/>/model sonnet · /advisor fable"]
    Q4 -- no --> AR["architect + delegate<br/>/model fable · lanes build"]

    classDef q fill:#edf2f4,color:#2b2d42,stroke:none
    classDef a fill:#8A2BE2,color:#fff,stroke:none
    class Q1,Q2,Q3,Q4 q
    class G,WF,AT,AD,AR a
```

Advisor re-reads the whole transcript uncached at $10/MTok per consult. Architect reads its own context from cache at $0.25. Cost table in [`wiring.md`](skills/rogger-agent-team/reference/wiring.md#1-advisor-mode).

## Layout

| Path | What |
|---|---|
| [`skills/rogger-agent-team/SKILL.md`](skills/rogger-agent-team/SKILL.md) | Loads on trigger. Lanes, wiring, effort, cost levers, traps, checklist. |
| [`reference/models.md`](skills/rogger-agent-team/reference/models.md) | Facts with sources. Pricing, advisor pairings, 5.1 vs 5 deltas. |
| [`reference/wiring.md`](skills/rogger-agent-team/reference/wiring.md) | Configs for all five patterns. Frontmatter, env, limits, cost table. |
| [`reference/prompt-kit.md`](skills/rogger-agent-team/reference/prompt-kit.md) | 16 paste-in blocks for 5.1, each marked whether Claude Code's Fable system prompt already carries it (checked live). Old-prompt patterns to delete. |
| [`reference/failure-modes.md`](skills/rogger-agent-team/reference/failure-modes.md) | Symptom → cause → fix. |
| [`agents/`](agents) | Five lanes, pinned model + effort + tools. |
| [`settings/`](settings) | `settings.json` examples, `CLAUDE.md` routing block. |

## Four traps

- Cyber-flagged prompt → session **falls back to Opus 4.8** (one transcript notice, then stays there). Bio → Opus 5. `claude --safe-mode` finds the trigger.
- Unpinned subagents **inherit Fable**. `CLAUDE_CODE_SUBAGENT_MODEL=opus`.
- Subagent cache is **5 min**. `subagentPromptCacheTtl: "1h"`.
- A **fork** subagent inherits Fable and the whole transcript, ignoring `CLAUDE_CODE_SUBAGENT_MODEL`. Spawn a named lane instead.

<sub>MIT. Each claim is attributed in <a href="skills/rogger-agent-team/reference/models.md#sources">models.md</a>: Claude Code docs, the Anthropic announcement, or Anthropic's bundled Fable 5.1 migration guidance.</sub>
