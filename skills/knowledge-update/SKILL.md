---
name: knowledge-update
description: "Extract domain knowledge from the current session and persist it to Lovable project memory. Detail files live under .memory/<domain>/<slug>.md on disk (mem:// detail files are not reliably persisted by the Lovable runtime), and mem://index.md is the always-in-context pointer. Saves facts, tracks hypotheses with confirmation counts, promotes hypotheses to Core rules at 5+ confirmations, and keeps mem://index.md in sync. Run at the end of a work session."
---

# Knowledge Update

Extract what was learned in this session and persist it to Lovable project memory.

## Storage model

This skill is the Lovable-native version of the classic Claude-Code knowledge-update flow. There is **no `~/.claude/projects/...` filesystem** in this environment.

`mem://` detail-file writes are not reliable in the Lovable runtime: a write can report success and then be gone in a later session, with no error surfaced at write time. Only `mem://index.md` itself persists reliably. So this skill uses two storage layers instead of one:

- **`mem://index.md`** — always in context, persists across sessions. Keep it short: one-liner Core rules and links to `.memory/...` detail files.
- **`.memory/<domain>/<slug>.md`** — canonical detail files that live on disk, commit with the project's repo, and are readable via `code--view`. These are the source of truth for any non-trivial fact — never rely on a `mem://<domain>/<slug>.md` write alone.

Domains: `.memory/design/`, `.memory/features/`, `.memory/constraints/`, `.memory/preferences/`, `.memory/reference/`, `.memory/security/`, `.memory/integrations/`, `.memory/architecture/`, `.memory/auth/`, `.memory/seo/`, `.memory/hypotheses/`.

When running a knowledge update, write the detail file to disk first, then sync `mem://index.md` to point at the disk path — the two writes can happen in parallel (no data dependency between them).

## When to Use

Run at the end of a session with `/knowledge-update` when:

- You just completed a non-trivial task
- You encountered a surprising bug, API quirk, or pattern
- Something worked differently than expected
- A previous hypothesis was confirmed or contradicted

## How It Works

### Step 1 — Identify the domain

Look at what was worked on. Check `mem://index.md` `## Memories` for a matching entry. If one exists, open the `.memory/...` file it links to and update in place — never duplicate.

### Step 2 — Extract insights

Review the conversation and classify:

**Facts** (certain, repeatable): API behaviors, schema details, confirmed constraints → write to `.memory/<domain>/<slug>.md`.

**Hypotheses** (seen once or twice, not confirmed): `.memory/hypotheses/<slug>.md` with a confirmation count and "last seen" date. Use the table format below.

**Confirmations** (a hypothesis proved true again): bump the confirmation count in the matching hypothesis file.

**Contradictions** (a rule proved wrong): remove the Core line from `mem://index.md` and move the body back into `.memory/hypotheses/<slug>.md` with a note about what contradicted it.

Every fact you classify gets a file. "Just implementation detail" is not a reason to skip writing it — that question only matters in Step 3 (whether it *also* earns a Core line). Most won't; they still get a file.

### Step 3 — Apply promotion logic

- Hypothesis with **5+ confirmations** → promote: add a Core line in `mem://index.md` and keep the detail file under `.memory/constraints/` or `.memory/features/`. Delete or shrink the hypothesis file.
- Rule **contradicted by new data** → demote per above.

### Step 4 — Write the memory file

Standard frontmatter:

```
---
name: <descriptive name>
description: <specific one-liner — drives relevance matching>
type: design | constraint | preference | feature | reference
---
<content>
```

Hypothesis file:

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

The index is always in context. Since `.memory/` files are NOT auto-injected, index bullets must be descriptive enough to signal when to open the file.

1. **Core** — one-liners applied to every action. Add only for promoted rules or universal constraints. Keep under ~150 chars each. Remove demoted rules.
2. **Memories** — bullets linking to `.memory/<domain>/<slug>.md` paths. Add/update whenever you create or rename a file.

Do the detail-file write and the `mem://index.md` update in **parallel** (no data dependency).

Replace, don't append the index: re-read `mem://index.md`, edit with `code--line_replace` or rewrite with `code--write`.

## What NOT to store

- Information already obvious from reading the current files
- File paths, component names, or other things a search would find instantly — **this does not cover behavioral or UX patterns**. "Settings accordion uses hash deep-linking" is not findable by grep; it is a design decision with intent behind it and belongs in `.memory/design/` or `.memory/features/`. The test: could a future session accidentally redo this decision without knowing it exists? If yes, write the file.
- Step-by-step how-to guides (those belong in a skill, not in memory)
- Ephemeral task details from this session only
- Lists of open or unresolved bugs

## Example

After a session adding a bank-import Undo/Move flow:

> **Created** `.memory/features/bank-import-batch-management.md` — `import_batch_id` schema, banner, bulk Move to Account.
> **Updated** `.memory/design/dashboard-floating-glass-tab-bar.md` — added contextual action-button pattern.
> **Updated** `mem://index.md` — new Memories bullets linking to both `.memory/...` paths.
> No promotions or demotions this round.
