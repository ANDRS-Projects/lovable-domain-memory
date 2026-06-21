---
name: knowledge-update
description: Extract domain knowledge from the current session and persist it to Lovable project memory under mem://. Saves facts, tracks hypotheses with confirmation counts, promotes hypotheses to Core rules at 5+ confirmations, and keeps mem://index.md in sync. Run at the end of a work session.
---

# Knowledge Update

Extract what was learned in this session and persist it to Lovable project memory at `mem://`.

This skill is the Lovable-native version of the classic Claude-Code knowledge-update flow. There is **no `~/.claude/projects/...` filesystem** in this environment — all persistent project knowledge lives under `mem://` and is read/written with the normal file tools (`code--view`, `code--write`, `code--line_replace`). The always-on index is `mem://index.md`.

## When to Use

Run at the end of a session with `/knowledge-update` when:
- You just completed a non-trivial task
- You encountered a surprising bug, API quirk, or pattern
- Something worked differently than expected
- A previous hypothesis was confirmed or contradicted

## How It Works

### Step 1 — Identify the domain

Look at what was worked on. Map it to an existing memory entry or decide a new one is needed.

Existing entries are listed in `mem://index.md` under `## Memories`. Open the ones whose description matches the session's topic before writing anything new — never duplicate.

A "domain" in `mem://` is just a folder convention. Use:
- `mem://constraints/<slug>.md` — things that must/must not happen
- `mem://features/<slug>.md` — how a feature is required to behave
- `mem://design/<slug>.md` — visual/design decisions
- `mem://preferences/<slug>.md` — how the user wants you to work
- `mem://reference/<slug>.md` — pointers to external resources
- `mem://hypotheses/<slug>.md` — patterns not yet confirmed (see Step 3)

### Step 2 — Extract insights

Review the conversation and classify:

**Facts** (certain, repeatable): API behaviors, schema details, confirmed constraints. Write directly into the appropriate `mem://` file.

**Hypotheses** (seen once or twice, not confirmed): record in `mem://hypotheses/<slug>.md` with a confirmation count and "last seen" date. Use the table format below.

**Confirmations** (a hypothesis proved true again): bump the confirmation count in the matching hypothesis file.

**Contradictions** (a rule proved wrong): demote — remove from Core in `mem://index.md` and move the body back into `mem://hypotheses/<slug>.md` with a note about what contradicted it.

### Step 3 — Apply promotion logic

- Hypothesis with **5+ confirmations** → promote: write the rule as a Core line in `mem://index.md` and (if it deserves detail) keep a full file under `mem://constraints/` or `mem://features/`. Delete or shrink the hypothesis file.
- Rule **contradicted by new data** → demote per above.

### Step 4 — Write/update the memory file

Use the standard Lovable memory frontmatter:

```
---
name: <descriptive name>
description: <specific one-liner — drives relevance matching>
type: design | constraint | preference | feature | reference
---
<content>
```

For hypothesis files:

```
---
name: <hypothesis>
description: Unconfirmed pattern about <topic>
type: reference
---
| Hypothesis | Confirmations | Last seen |
|---|---|---|
| <claim> | 2 | 2026-06-19 |

Notes: <evidence, where seen, what would confirm or refute>
```

### Step 5 — Sync `mem://index.md`

The index is always in context, so it's where enforcement happens. After any write:

1. **Core** — one-liner rules applied to every action. Add a line here only for promoted rules or universal constraints. Keep each under ~150 chars. Remove demoted rules.
2. **Memories** — bullet list of links to detail files. Add/update the bullet whenever you create or rename a file. Each bullet needs a description specific enough to judge relevance without opening the file.

Replace, don't append: re-read `mem://index.md`, edit in place with `code--line_replace` or rewrite with `code--write`.

## What NOT to store

- Information already obvious from reading the current files
- File paths, component names, or other things a search would find instantly
- Step-by-step how-to guides (those belong in a skill, not in memory)
- Ephemeral task details from this session only
- Lists of open or unresolved bugs

## Example

After a session fixing the editor "jump back to Content tab + lost images" bug:

> **Updated** `mem://constraints/strip-data-urls.md` — added the sessionStorage cache requirement so image fields survive editor remounts.
> **Created** `mem://features/editor-tab-persistence.md` — Tabs must be controlled + persisted to `sessionStorage` key `pdk:edit-active-tab`.
> **Updated** `mem://index.md` — added both as Core lines and Memories bullets.
> No promotions or demotions this round.
