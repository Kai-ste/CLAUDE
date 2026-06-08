# `~/.claude` — Claude Code user configuration

This is the **user-level** (global) configuration directory for [Claude Code].
Everything here applies to *every* project you open, unless a project-local
`.claude/` overrides it.

This repo tracks only the **curated, hand-authored config** (agents, skills,
commands, output styles, conventions, settings). All the runtime state Claude
Code generates — session logs, caches, credentials, history — is ignored via
`.gitignore`. See [What's tracked vs ignored](#whats-tracked-vs-ignored).

---

## How Claude Code discovers things here

Claude Code scans a few **fixed folder names** at this level. The folder name is
the contract — a file only "works" if it sits at the right path:

| Path | What it is | Who triggers it |
|------|-----------|-----------------|
| `agents/*.md` | Subagents | Claude (auto-delegates) or you ("use the X agent") |
| `skills/<name>/SKILL.md` | Skills | Claude (auto, when task matches description) |
| `commands/*.md` | Slash commands | You (`/name`) |
| `output-styles/*.md` | Output styles | You (`/output-style name`) |
| `settings.json` | Shared settings | — (always active) |
| `settings.local.json` | Machine-local settings | — (always active, **not committed**) |
| `CLAUDE.md` | Global memory / always-on context | — (auto-loaded every session) |
| `conventions/` | **Not** a native feature — plain data read by some skills | the skills themselves |

**Precedence:** project-local `./.claude/` > this user-level `~/.claude/`. So a
project can add or shadow anything defined here.

---

## Agents (`agents/`)

A **subagent** is a specialized assistant that runs in its **own separate
context window** and reports a summary back to the main conversation. Use them to
keep a focused task (and its noisy output) out of the main thread.

**File format** — one `.md` per agent, with YAML frontmatter:

```markdown
---
name: writing-reviewer
description: Reviews prose quality, clarity, conciseness, grammar, academic tone
tools: Read, Glob, Grep        # restricts what this agent may call
model: opus                    # which model runs it
---
You are a Writing Quality Reviewer ...   # the agent's system prompt
```

**How it's triggered — two ways:**
1. **Automatic delegation.** When your request matches an agent's `description`,
   Claude may hand the task off on its own. *The `description` is the routing
   signal* — write it as "when to use me," not just "what I am."
2. **Explicit.** You say "use the `karen` agent to…" and Claude invokes it.

`tools:` is a safety/scoping fence — a review agent given only `Read, Glob, Grep`
literally cannot edit files. `model:` lets you run cheap agents on a smaller
model and heavy ones on Opus.

**What's here:** an academic-writing toolkit (`writing-reviewer`,
`prose-polisher`, `section-drafter`, `bibliography-auditor`, `technical-reviewer`,
`logic-reviewer`, `consistency-checker`, `latex-figure-specialist`,
`latex-layout-auditor`, `research-analyst`, `paper-crawler`, `brainstormer`),
general dev/QA agents (`karen`, `Jenny`, `task-completion-validator`,
`code-quality-pragmatist`, `claude-md-compliance-checker`, `ultrathink-debugger`,
`ui-comprehensive-tester`), and a software-workflow set from the solatis config
(`architect`, `developer`, `debugger`, `quality-reviewer`, `technical-writer`).

---

## Skills (`skills/`)

A **skill** is a packaged capability: a folder containing a `SKILL.md` plus,
optionally, scripts and reference files. Skills use **progressive disclosure** —
only the short `SKILL.md` description sits in context until the skill is
triggered, at which point its full instructions (and any bundled scripts) load.

**File format** — `skills/<name>/SKILL.md`:

```markdown
---
name: deepthink
description: Invoke IMMEDIATELY when the user requests structured reasoning for
  open-ended analytical questions. Do NOT explore first.
---
# DeepThink
...instructions, often invoking a bundled script...
```

**How it's triggered:** **model-invoked.** Claude continuously matches the task
against every skill's `description` and activates one when it fits — you don't
type a command. (This is the main difference from commands, which *you* trigger.)
Some skills are also exposed as slash commands.

Skills can ship code. Several skills here (`planner`, `refactor`,
`codebase-analysis`, …) call into the shared **`skills/scripts/`** Python package
rather than reasoning inline — the `SKILL.md` tells Claude to run the script and
the script *is* the workflow.

**What's here:** `deepthink`, `planner`, `refactor`, `codebase-analysis`,
`problem-analysis`, `decision-critic`, `incoherence`, `doc-sync`,
`prompt-engineer`, `arxiv-to-md`, `cc-history`, and the shared `scripts/`
package. (These come from the solatis config — see `README_SOLATIS.md`.)

---

## Commands (`commands/`)

A **slash command** is a reusable prompt **you** trigger by typing `/name`. Think
of it as a saved, parameterized instruction.

**File format** — `commands/<name>.md`:

```markdown
---
name: academic
description: Academic writing multi-agent orchestrator. TRIGGER when ...
allowed-tools: Agent, Read, Glob, Grep, Edit, Write, Bash, WebSearch, WebFetch
argument-hint: [task-description]
---
The prompt body. Use $ARGUMENTS for everything after the command,
or $1, $2 for positional args.
```

**Commands vs. skills:** commands are **user-triggered** (you type `/academic`);
skills are **model-triggered** (Claude decides). A command body is essentially a
prompt template; `allowed-tools` scopes what it may use while running.

**What's here:** `academic` and `academic-writing` (orchestrators for the
academic agent suite).

---

## Output styles (`output-styles/`)

An **output style** swaps how Claude communicates by editing its system prompt.
**Only one is active at a time**; switch with `/output-style <name>`. Unlike a
command (a one-shot prompt) or an agent (a delegated task), a style changes the
*default voice* of every reply until you switch back.

**File format** — `output-styles/<name>.md`:

```markdown
---
name: Direct
description: Direct, fact-focused communication. Minimal explanation.
---
# Technical Directness
...behavioral instructions...
```

**What's here:** `direct.md` — terse, no-hedging, fact-first style.

---

## Conventions (`conventions/`)

**Not a Claude Code feature.** This is a plain reference library (markdown + a
`REGISTRY.yaml` index) that some **skills read from disk** — the `planner` and
`refactor` skills load `conventions/severity.md`, `conventions/diff-format.md`,
`conventions/code-quality/`, etc. by path. It lives at this top level (rather than
inside a skill) because those skills reference it as `.claude/conventions/`.

---

## Settings (`settings.json`, `settings.local.json`)

- **`settings.json`** — shared, committed config: model, theme, effort level,
  permission rules, hooks, env vars. Safe to version.
- **`settings.local.json`** — machine-/checkout-specific overrides (e.g. one-off
  permission grants). **Not committed** — it's in `.gitignore`. Claude Code
  auto-writes to this file as you approve permissions.

Edit these via the `update-config` skill or the `/config` command rather than by
hand when possible.

---

## What's tracked vs ignored

**Tracked** (the curated config): `agents/`, `commands/`, `skills/`,
`output-styles/`, `conventions/`, `settings.json`, `CLAUDE.md`,
`keybindings.json`, the README files, and `.gitignore`.

**Ignored** (runtime state, caches, secrets — regenerated by Claude Code):
`projects/`, `sessions/`, `session-env/`, `file-history/`, `shell-snapshots/`,
`telemetry/`, `cache/`, `backups/`, `plugins/`, `history.jsonl`,
`.credentials.json`, `settings.local.json`, and more. The `.gitignore` uses an
**allowlist** ("ignore everything, then un-ignore the curated folders") so new
runtime junk never leaks into the repo.

---

## Provenance

- The **canonical academic-writing principle set** lives in
  `commands/academic-writing.md` (30 principles, codes A1–F); the academic agents
  reference it at `~/.claude/commands/academic-writing.md`.
- Parts of this config originate from
  [`solatis/claude-config`](https://github.com/solatis/claude-config) — see
  `README_SOLATIS.md` for its own docs. The academic agents/commands are a
  separate authored suite.

[Claude Code]: https://claude.com/claude-code
