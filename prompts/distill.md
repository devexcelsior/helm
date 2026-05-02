---
description: Distill a shipped roadmap into a numbered ADR
argument-hint: "<slug>"
---
You are running the **Distillation** phase from `~/.pi/agent/AGENTS.md` § 7.

**Bump thinking level to high** per Cross-cutting principles → Thinking levels per phase. Synthesis-heavy: rewriting roadmap reasoning into prose, stripping scaffolding, supersession analysis.

## Resolve active feature (if `$1` missing)

Per Cross-cutting principles → State management: if `$1` is empty, read slug from `.pi/active.md`. Explicit `$1` overrides.

Slug: $1
Roadmap: `docs/roadmap/$1/$1.md`

## Pre-flight

1. Confirm the roadmap is **shipped** — verify by checking that all step docs under `docs/roadmap/$1/` are marked complete (acceptance criteria all checked) and the implementing code is on the current branch via `bash git diff main` or equivalent. If anything is in-flight, stop and report.
2. Find the next ADR number: `bash ls docs/adr/ | grep -E '^[0-9]{4}-' | sort | tail -1` — increment by 1, zero-pad to 4 digits.
3. Read `docs/adr/template.md` for the format.

## Distillation contract

ADRs are **decision records**, not work logs. Strip the workflow scaffolding from the roadmap and keep only what a future cold-start reader needs to understand why the code looks the way it does.

**Keep**: Context/problem, decision drivers, options considered (with the load-bearing rejection reason for each), the chosen option, consequences (positive/negative/to-watch), validation signals.

**Drop**: Step lists, chunk-size plans, in-progress notes, "TODO" markers, agent harness mechanics (which model ran which pass, token budgets), the `Responses to critiques` table (revision provenance, not record), anything that was true during the build but isn't true about the system after ship.

**Rewrite**: The 6-field reasoning template entries in the roadmap collapse into prose. Don't preserve the table format — it's planning scaffolding, not record format.

## Output

Write to `docs/adr/NNNN-<adr-slug>.md` where:
- `NNNN` is the next sequential number from pre-flight step 2
- `<adr-slug>` is kebab-case past-tense decision form (e.g. `use-tursodb-for-edge-cache`, not `tursodb-decision`). Note: ADR slug is decided here, not the same as the roadmap slug.

Frontmatter:
- `status: accepted` (use `proposed` only if shipping but not yet validated; `superseded-by:` if this replaces an earlier ADR)
- `date:` today's date (YYYY-MM-DD)
- `distilled-from: docs/roadmap/$1/$1.md`
- `supersedes:` any ADR numbers this replaces (check existing ADRs for related topics; if you replace one, add `superseded-by:` to the old ADR in the same change)

## After writing the ADR

1. Update `docs/roadmap/$1/$1.md` frontmatter (or add a footer if no frontmatter): `produced-adrs: [NNNN]` so the back-link exists.
2. If this ADR supersedes an earlier one, edit the older ADR's frontmatter to set `superseded-by: NNNN`.
3. Report: ADR number, slug, what it supersedes (if any), and a one-sentence summary of the decision.

Do not delete or archive the roadmap — leave that to the user. The ADR is the durable record; the roadmap is provenance.

## State file lifecycle note

Per Cross-cutting principles → State management, `.pi/active.md` describes the feature-in-flight state and is **NOT deleted by /distill**. After the ADR is successfully written and the roadmap's `produced-adrs` frontmatter is updated, append `distill` to active.md's `Phases completed:` line (alongside earlier phases like `plan, critique-..., revise, decompose`).

Active.md persists through `/calibrate` (if run) and is deleted by `/finalize` as part of worktree removal after the merge to main is complete (per §8 — Post-ship git operations).

If the ADR write failed for any reason, do NOT update active.md — leave its state intact so the user can re-run.

## Next-step recommendation (output format — required after ADR written + roadmap frontmatter updated + active.md phase appended)

After the ADR is written, the roadmap's `produced-adrs:` frontmatter is updated, and `distill` has been appended to active.md's `Phases completed:` line, append exactly this block.

```markdown
---
**✅ ADR-NNNN written — feature distilled — ready for /calibrate**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/calibrate` | **high** (already correct from /distill, no Shift+Tab needed) |
```

`/calibrate` resolves the most recent ADR via `bash ls -t docs/adr/*.md \| head -1`. The slug is derived from the ADR's `distilled-from:` frontmatter (independent of active.md state — works whether active.md is present or not). To pin to a specific ADR explicitly, use `/calibrate <NNNN>` (e.g., `/calibrate 0003`).

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands. Output `/calibrate` — never `/calibrate <slug> <NNNN>`.
