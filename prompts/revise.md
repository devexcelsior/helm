---
description: Revise a plan against critiques, producing a per-point response table
argument-hint: "<slug>"
---
You are running the **Plan revision** phase (Pass 3) from `~/.pi/agent/AGENTS.md` § 1.

**Bump thinking level to high before invoking** — synthesis-heavy task per Cross-cutting principles → Thinking levels per phase.

## Resolve active feature (if `$1` missing)

Per Cross-cutting principles → State management: if `$1` is empty, read the slug from `.pi/active.md` via `bash sed -n 's/^# Active feature: //p' .pi/active.md`. If file missing, stop with the standard error. Explicit `$1` overrides.

Slug: $1

Plan: `docs/roadmap/$1/$1.md`
Critiques: discover via `bash ls docs/roadmap/$1/$1.critique-*.md` — read all of them.

## Read everything

1. Read `docs/roadmap/$1/$1.md` end to end.
2. List critique docs: `bash ls docs/roadmap/$1/$1.critique-*.md` — read each.
3. Note all six-field findings AND all discipline-axis findings from each critique.

Count the total critique points across all input docs. The response table at the bottom of the revised plan must have exactly that many rows. No silent drops.

## Revise the plan

For each critique point — including discipline-axis findings — make one of three choices:

- **Incorporate**: edit the relevant decision/section. Cite where the change landed (section name or decision number).
- **Reject**: leave unchanged with a one-line reason. "We don't think it's important" is not a reason. "Out of scope per Goal section line N" or "the cited concern is mitigated by Decision M's constraint X" both qualify.
- **Partial**: incorporate part, reject part. Both halves explicit in the response table.

Edit `docs/roadmap/$1/$1.md` in place. Do not produce a separate "revised plan" file.

If a critique surfaced a missing decision on a cross-cutting concern (test strategy, accessibility, error handling, observability, deployment, security, performance), add it as a new Decision N with the full 6-field reasoning template. Do not add a paragraph in passing — add a Decision so it survives `/decompose` and `/distill`.

## Required output: response table

Append at the bottom of the revised plan:

```markdown
## Responses to critiques

### <perspective-tag-1>

| # | Point | Response |
|---|---|---|
| 1 | <one-line summary of finding> | **Incorporated**: <where in plan>. |
| 2 | ... | **Rejected**: <one-line reason>. |
| 3 | ... | **Partial**: kept <X>, dropped <Y> because <Z>. |

### <perspective-tag-2>
(same structure)

### Discipline-axis findings (cross-critique)

| Axis | Finding | Response |
|---|---|---|
| Deferred decisions | <quoted construction with line> | **Resolved**: <one-line reason>. |
```

One row per critique point. Per Cross-cutting principles → Evidence over claims, "incorporated" responses must cite the section name or decision number that changed.

## Self-audit before finalizing — hedge-language watch

Scan your own revised plan for these patterns. Each is a silent drop dressed as a resolution:

- **"We will consider X" / "may be added later" / "could be evaluated"** — deferral dressed as resolution. If incorporating, commit. If rejecting, reject.
- **"Either X or Y is fine"** — same pattern as the discipline-axis check. Pick one. Resolving previously-flagged instances does not prevent new ones one level down.
- **Constraint paragraphs added without changing the Decision** — if a critique surfaced a real gap, the Decision text changes, not just the prose around it.
- **"This is implied by Decision M"** — "implied" is not coverage; either make explicit or reject with reason.

Fix any of the above before finalizing.

## Hard rules

- Every critique point must appear in the response table. Count rows: must equal critique-point total across input docs.
- Incorporated changes must be locatable. "Various places" is not a citation.
- Discipline-axis findings get their own table section.
- Do not silently refactor the plan structure. If you reorder decisions or merge sections, note it as its own row.

## Next-step recommendation (output format — required after revised plan + response table written)

After the revised plan is written and the response-to-critiques table is appended, append exactly this block. **Bare commands only** — slug is in `.pi/active.md`.

```markdown
---
**✅ Revised plan written — response table appended — ready for /decompose**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/decompose` | **medium** — Shift+Tab down from high |
| Then | `/preflight` (step 1) | **minimal** |
| Then | `/implement` | **medium** |
| Then | `/verify` | **minimal** |
```

The per-step pipeline (preflight → implement → verify → assess) starts after /decompose produces step docs and updates `.pi/active.md`.

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands. Output `/decompose` — never `/decompose <slug>`.
