---
description: Critique pass with perspective rotation + discipline-axis check
argument-hint: "<slug> <perspective-tag>"
---
You are running the **Critique** phase from `~/.pi/agent/AGENTS.md` § 1. Default model is K2.6 with explicit role swap (perspective rotation is the load-bearing signal driver). Cross-model critique (DSV4 Pro Think High preferred; MiniMax M2.7 fallback) is recommended after perspective rotation passes — produces unique catches not surfaced by perspective rotation alone (validated feature 4).

**Bump thinking level to high before invoking** — synthesis-heavy task per Cross-cutting principles → Thinking levels per phase.

## Step 0 — Argument resolution (smart-parse)

`/critique` accepts two invocation forms:

- **Bare with perspective only** (preferred when state exists): `/critique <perspective-tag>`. Slug resolves from `.pi/active.md`.
- **Explicit slug + perspective**: `/critique <slug> <perspective-tag>`. Both arguments explicit.

Apply this resolution:

1. **If `$1` matches one of `senior-engineer-risk`, `pm-requirements`, `security-perf` AND `$2` is empty**: treat `$1` as the perspective tag. Read slug from `.pi/active.md` (`bash sed -n 's/^# Active feature: //p' .pi/active.md`). Use the perspective tag as `$2` going forward.
2. **Else**: treat `$1` as the slug, `$2` as the perspective tag (legacy form).
3. **If neither form yields a slug**: error with *"No active feature. Run `/plan <slug> <topic>` first, or pass `<slug> <perspective>` explicitly."*

Slug: <resolved>
Perspective tag: <resolved> (one of: `senior-engineer-risk` | `pm-requirements` | `security-perf`)

Plan to critique: `docs/roadmap/$1/$1.md`
Output path: `docs/roadmap/$1/$1.critique-$2.md`

Perspective expansion (apply per the tag):
- `senior-engineer-risk` → "senior engineer reviewing for risk and edge cases"
- `pm-requirements` → "PM checking requirements coverage and acceptance criteria"
- `security-perf` → "security/perf reviewer"

## Output frontmatter

The reviewer field MUST be one of these exact strings (no variations, no additions):

- `Moonshot Kimi K2.6 thinking`
- `DeepSeek V4 Pro Think High`
- `DeepSeek V4 Pro Think Medium`
- `MiniMax M2.7`
- `<unknown — user to fill>`

If you cannot determine your actual model identity from runtime context, write `<unknown — user to fill>` literally. **Confabulating an identity invalidates the audit trail of cross-model studies.** Validated failure mode: DSV4 Pro Think High wrote "Kimi K2.6 thinking mode high" in 3 consecutive critique runs across 2026-04-29 because it disregarded the prior "do not guess" instruction. The constrained list above eliminates the guessing surface — pick from the list or use `<unknown>`. Do not invent variations.

Thinking-level qualifier in the model string must match your actual runtime thinking level (`High`, `Medium`, `minimal`). Do NOT default to `High` if your runtime is medium.

```
---
reviewer: <one of the strings above>
perspective: $2
plan: docs/roadmap/$1/$1.md
date: <YYYY-MM-DD>
---
```

## Pass 1 — Six-field scoring per decision

Read `docs/roadmap/$1/$1.md`. For each major decision, score it against the six reasoning template fields. Per Cross-cutting principles → Evidence over claims, every ✓/⚠/❌ must cite the line(s) of the plan it scored — not just "Decision ✓" but "Decision ✓ (line 30: clear, names exact API)".

1. Decision — clear?
2. Constraints — specific, cited, complete?
3. Second-order effects — has the planner thought past the local change?
4. Failure modes — was the deceptively obvious wrong answer named? Real, not strawman?
5. Alternatives considered — at least 2 with real rejection reasons?
6. Self-consistency — any contradictions with prior decisions?

If any field is empty or shallow, that's the gap. Quote the relevant lines.

## Pass 2 — Discipline-axis check (required, regardless of perspective)

Six-field scoring catches gaps in stated decisions. It systematically misses gaps in *how* decisions were made. Scan the entire plan (Goal, Out-of-scope, UI Specification, Invariants, every section) for these four failures. Each surfaced finding is a critique gap.

1. **Deferred decisions** — any "either X or Y works" / "X is fine; Y is also fine" / "could be A or B" / "may be placed inside or outside" construction. Quote each instance with line number. Resolution required.
2. **Unjustified specifics** — concrete choices stated without defense (specific naming conventions, numeric thresholds, library pins). Quote with line number; ask for justification.
3. **Circular rejection reasoning** — alternatives rejected because of a constraint the planner set themselves. Quote both the rejection and the supporting constraint.
4. **Missing decisions on cross-cutting concerns** — explicitly check whether the plan covers: testing strategy, accessibility, error handling, observability, deployment, security, performance budgets. List the cross-cutting concerns the plan does NOT address.
5. **OR clauses in defensive-code constraints** — scan every constraint that names a defensive-code pattern (try/catch, null check, type guard, listener cleanup, specific import path, etc.) for OR/escape-hatch phrasing ("guard internally or in callers", "X or Y is acceptable", "unless Z"). Per Cross-cutting principles → Mechanical verification for defensive-code constraints, OR clauses make the constraint un-mechanically-checkable and let the model claim compliance via the loophole branch. Quote each OR-clause constraint and recommend either: split into separate constraints per branch (each with its own mechanical check), or pick one branch and remove the alternative.

Scan the **entire artifact**, not just sections previously flagged. Pattern recognition is not pattern elimination — the model resolves named instances and reproduces unnamed ones one level down.

## Output structure

Write `docs/roadmap/$1/$1.critique-$2.md` with:

1. **Frontmatter** — fill in the actual reviewer model.
2. **Pass 1 — Six-field scoring** with quoted evidence per ✓/⚠/❌.
3. **Pass 2 — Discipline-axis findings** with quoted evidence per axis. If an axis surfaces no finding, state "no instances found" — never silently omit.
4. **Risk summary table** — Severity | Likelihood | Mitigation, listing top findings only.
5. **Verdict** — count of substantive findings, single most important to address.

## Cap reminder

Critique-pass sweet spot is 2 (3 if cross-cutting infra). Beyond that, you fabricate problems to satisfy the prompt. Stop at 2 unless plan covers genuinely unfamiliar terrain.

## Next-step recommendation (output format — required after critique written)

After the critique file is written, append exactly this block. **Bare command for /revise; perspective tag explicit for /critique** (perspective is NOT in state).

If this is critique pass 1 (only one critique file exists for the slug at `docs/roadmap/$1/$1.critique-*.md`):

```markdown
---
**✅ Critique pass 1 complete (`<perspective>`) — ready for pass 2**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/critique <other-perspective-tag>` (rotated) | **high** |
| Then | `/revise` | **high** |
```

If this is critique pass 2 (two critique files now exist for the slug):

```markdown
---
**✅ Critique pass 2 complete (`<perspective>`) — ready for /revise**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/revise` | **high** |
| Then | `/decompose` | **medium** |
```

**Available perspective tags** (pick one not yet used for this slug):

| Tag | Lens |
|---|---|
| `senior-engineer-risk` | senior engineer reviewing for risk and edge cases |
| `pm-requirements` | PM checking requirements coverage and acceptance criteria |
| `security-perf` | security / performance reviewer |

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands wherever the state file resolves all required args. Slug is in `.pi/active.md`. Perspective tag is NOT in state — must be passed explicitly. Output `/critique pm-requirements` — never `/critique <slug> pm-requirements`.
