# lovable-domain-memory

[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-support-FFDD00?style=flat&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/dare2create)

A lightweight knowledge accumulation system for [Lovable](https://lovable.dev) projects. Captures what you learn while working — API quirks, patterns, confirmed rules — and makes it available in every future session automatically.

This is the **Lovable-native** counterpart to the Claude Code `claude-domain-memory` skill. Lovable has no `~/.claude/projects/...` filesystem, so this variant stores project memory inside the project's own repo instead.

## The problem it solves

Lovable projects accumulate context over many sessions, but there's no built-in way to track patterns that emerge over time — API quirks, design decisions, constraints — beyond what's already in code or chat history. You rediscover the same quirks. Hypotheses never get confirmed or discarded. Knowledge stays in the conversation instead of the project.

This system adds a structured memory convention that accumulates knowledge across sessions — with a clear lifecycle from observation to hypothesis to confirmed rule.

## How it works

`mem://` detail-file writes are not reliably persisted by the Lovable runtime — a write can report success and then be gone in a later session. So this skill uses a **dual-storage** model:

- **`.memory/<domain>/<slug>.md`** — every detail file lives on disk, in the project's own repo. This is the durable storage layer.
- **`mem://index.md`** — the always-in-context pointer. This file *does* persist reliably, so it's where enforcement happens. Its `## Memories` bullets link to `.memory/...` paths.

Domains:

- `.memory/constraints/<slug>.md` — things that must/must not happen
- `.memory/features/<slug>.md` — how a feature is required to behave
- `.memory/design/<slug>.md` — visual/design decisions
- `.memory/preferences/<slug>.md` — how the user wants you to work
- `.memory/reference/<slug>.md` — pointers to external resources
- `.memory/security/<slug>.md` — security-relevant constraints and decisions
- `.memory/integrations/<slug>.md` — third-party service quirks and contracts
- `.memory/architecture/<slug>.md` — structural decisions and their rationale
- `.memory/auth/<slug>.md` — authentication/authorization behavior
- `.memory/seo/<slug>.md` — SEO-relevant constraints and decisions
- `.memory/hypotheses/<slug>.md` — patterns seen but not yet confirmed

The always-on index, `mem://index.md`, holds:

- **Core** — promoted rules and universal constraints, applied to every action
- **Memories** — a bullet list linking to the `.memory/...` detail files above

At the end of a session, run `/knowledge-update`. Lovable's assistant reviews what was worked on and updates the relevant memory files — adding facts, tracking hypothesis confirmation counts, and promoting anything that crosses the threshold (5+ confirmations) into a Core rule in `mem://index.md`.

## Installation

### 1. Install the skill

Copy `skills/knowledge-update/SKILL.md` into your Lovable project's skills setup (wherever your Lovable workspace expects custom skills/commands).

### 2. Seed `mem://index.md`

If it doesn't exist yet, create it with two sections:

```markdown
## Core

## Memories
```

### 3. Use the templates

`templates/domain_template.md` and `templates/rules_template.md` are reference layouts for hypothesis files and rule blocks — adapt the frontmatter to the format described in the skill (`type: design | constraint | preference | feature | reference`).

## Usage

At the end of any session where you learned something non-obvious:

```
/knowledge-update
```

The assistant will:

1. Identify which domain(s) the session touched
2. Extract facts, new hypotheses, and confirmations
3. Update the relevant `.memory/<domain>/<slug>.md` files
4. Promote any hypothesis that reached 5+ confirmations to a Core rule
5. Sync `mem://index.md` so rules and memory links stay current

## What to store

**Good candidates:**
- API quirks not obvious from documentation
- Constraints that caused bugs when assumed wrong
- Patterns confirmed across multiple sessions
- Rules that should apply by default going forward

**Skip:**
- Information already obvious from reading the current files
- File paths, component names, or anything a search would find instantly
- Step-by-step how-to guides (those belong in a skill, not in memory)
- Ephemeral task details from a single session
- Lists of open or unresolved bugs

## File structure

```
mem://
  index.md                  ← Core rules + Memories index, always in context
                               (the only mem:// file this skill relies on)

.memory/                    ← durable storage, lives in the project's own repo
  constraints/<slug>.md
  features/<slug>.md
  design/<slug>.md
  preferences/<slug>.md
  reference/<slug>.md
  security/<slug>.md
  integrations/<slug>.md
  architecture/<slug>.md
  auth/<slug>.md
  seo/<slug>.md
  hypotheses/<slug>.md      ← unconfirmed patterns with a confirmation count
```

## Example hypothesis file

```markdown
---
name: editor-tab-persistence
description: Unconfirmed pattern about tab state surviving editor remounts
type: reference
---
| Hypothesis | Confirmations | Last seen |
|---|---|---|
| Tabs must be controlled + persisted to sessionStorage to survive remounts | 2 | 2026-06-19 |

Notes: seen twice when the Content tab reset after an editor remount.
```

## Memory entries are personal

The skill and templates are generic. The `.memory/` entries themselves contain knowledge specific to your project — they're not meant to be shared. Each project builds its own through use.

## License

MIT — see [LICENSE](LICENSE).
