---
description: Decompose a roadmap plan into self-contained step docs with cross-step consistency sweep
argument-hint: "<slug>"
---
You are running the **Step decomposition** phase from `~/.pi/agent/AGENTS.md` § 2.

Set thinking level to **medium** per Cross-cutting principles → Thinking levels per phase.

## Resolve active feature (if `$1` missing)

Per Cross-cutting principles → State management: if `$1` is empty, read the slug from `.pi/active.md`. Explicit `$1` overrides.

Slug: $1
Plan: `docs/roadmap/$1/$1.md`
Step docs output dir: `docs/roadmap/$1/`

Produce one step doc per implementation unit at `docs/roadmap/$1/step-<N>-<step-slug>.md`. Each step doc must be **self-contained** — someone reading only that doc with no conversation context can implement it. Required fields per step:

- **Acceptance criteria** — concrete, verifiable
- **Files touched** — explicit paths
- **API/function signatures** — verify they exist in the codebase via `bash read`/`bash grep`; do not assume
- **Verification commands** — exact `bash` lines (tests, typecheck, lint)
- **Invariants to preserve** — what must NOT change (signatures, public APIs, test names, perf characteristics)

Apply the 6-field reasoning template (Decision/Constraints/Second-order/Failure modes/Alternatives/Self-consistency) at the **step level** as well — make implicit intent explicit.

## Mechanical constraint checks (required for defensive-code constraints)

For each constraint that names a specific code pattern (try/catch presence, null checks, type guards, listener cleanup, specific import path, defensive default values, error fallbacks), the step doc must include a `## Mechanical constraint checks` section with a table mapping each constraint to a bash one-liner that exits 0 if satisfied, non-zero if violated.

Per Cross-cutting principles → Mechanical verification for defensive-code constraints, this is non-optional. Constraints with OR clauses ("guard internally or in callers") are explicitly forbidden — split into separate constraints per branch, or pick one branch and remove the alternative.

Format:

```markdown
## Mechanical constraint checks

| # | Constraint (matches Constraints section) | Verification command | Expected exit |
|---|---|---|---|
| N | "<exact text>" | `<bash one-liner>` | 0 |
```

Examples of grep/awk patterns:

- `try/catch in function F`: `awk '/^export function F\b/,/^}/' <file> | grep -q '^[[:space:]]*try'`
- `import path uses .ts extension`: `! grep -E "from '\.\./[^']+\.js'" <file>`
- `listener cleanup present in useEffect`: `awk '/useEffect/,/^}\)/' <file> | grep -q 'removeEventListener'`
- `function not imported from react`: `! grep -q "from 'react'" <file>`

What does NOT require mechanical check (model-judged only): architectural decisions, UX requirements, naming conventions, design rationale, performance budgets without numeric threshold.

## Consistency sweep (required before claiming done)

After writing all step docs, **do not stop**. The doctrine §2 cross-step consistency contract is non-optional. Write the report to `docs/roadmap/$1/_consistency.md` as a separate file in the same directory as the step docs before declaring decomposition complete. Do NOT append the sweep to the last step doc — keep the audit artifact discoverable as its own file.

### 1. Roadmap-to-step traceability

Build a table mapping every numbered requirement in the roadmap's Goal section, every entry in UI Specification (if any), every Invariant, and every Out-of-Scope item to the step doc and acceptance criterion (or invariant) that covers it.

| Roadmap item | Covered by step | Acceptance criterion / invariant |
|---|---|---|
| Goal #1: ... | step-N | "..." (quoted) |

Items with no coverage = silent drops. Either add the missing acceptance criterion to a step doc, or edit the roadmap to mark the item deferred (with a one-line reason). Do not leave gaps.

### 2. Roadmap decision coverage

For each major Decision in the roadmap (1, 2, 3, …): cite the step doc whose implementation reflects it. If the roadmap left any decision deferred ("either X or Y works", "could be A or B"), the step doc that implements it must **resolve** it with a one-sentence reason. List the resolutions explicitly.

| Roadmap decision | Step | Resolution if deferred |
|---|---|---|
| Decision N: ... | step-M | (one-line reason or "no deferral") |

### 3. Internal step consistency — with evidence

For each step doc, scan Context, Decision, Constraints, and Implementation sections for prose contradictions. **Per Cross-cutting principles → Evidence over claims**, the sweep must produce evidence even when no contradiction is found. For each step, quote one Context/Decision/Implementation triplet showing alignment:

```
step-N:
  Context (line X): "..."
  Decision (line Y): "..."
  Implementation (line Z): "..."
  Verdict: aligned (or: contradiction — Context says A, Decision says B)
```

"No contradictions found" without quoted text is rubber-stamping; the sweep must show its work.

### 3b. Sketch-vs-constraint mechanical validation (positive + negative)

For each step doc with a `## Mechanical constraint checks` section, run a two-phase validation per Cross-cutting principles → Mechanical verification rules 4 and 5. **Both phases are required.** Either phase failing blocks the step doc from shipping.

**Phase 1 — Positive validation (sketch satisfies checks).**

1. Extract the implementation sketch fenced code block to a temp file.
2. Run each mechanical check against the temp file.
3. Every check must exit 0. If any check fails: sketch is defective — the step doc claims a constraint that the sketch itself doesn't satisfy. Fix the sketch (or the constraint) and re-run.

**Phase 2 — Negative validation (checks actually catch violations).**

For each check, produce a deliberately-broken permutation of the sketch that violates the specific constraint the check is verifying. Run the check against the broken permutation — it must exit non-zero. If the check passes on broken code, the check itself is buggy (false-positive risk) and must be fixed before shipping.

Example permutations:
- Check verifies `try/catch around X`: mutate sketch to remove the `try { ... } catch { ... }` wrapping `X`.
- Check verifies `import path uses .ts extension`: mutate sketch to change `from '../foo.ts'` to `from '../foo.js'`.
- Check verifies `listener cleanup in useEffect`: mutate sketch to remove the `return () => removeEventListener(...)` line.
- Check verifies `function not imported from react`: mutate sketch to add `import { foo } from 'react'`.

The mutation should be the minimal change that violates the constraint. Use sed, awk, or manual edits — the goal is producing a single broken file per check, running the check, capturing the non-zero exit.

**Report format**:

```
Check N: <constraint summary>
  Positive (sketch as-is): exit code 0 ✓
  Negative (sketch with <mutation>): exit code <non-zero> ✓
```

If any check shows positive fail, negative pass, or both, the step doc is not ready to ship. Fix and re-run.

This catches two distinct failure modes:
- **Sketch defects** (Phase 1): the canonical defensive-code OR-clause failure where a sketch claims compliance via prose but doesn't show the code pattern. Validated on dark-mode-toggle Step 2 — sketch had no try/catch despite the constraint requiring it; OR clause hid the contradiction.
- **Check defects** (Phase 2): silent bugs in the check command itself that always pass or always fail regardless of code. Validated on dark-mode-toggle Step 2 retrofit — `awk '/.../\b...'` patterns were silently broken because gawk interprets `\b` as backspace, not word boundary; checks would have false-failed correct implementations.

Model-based scans miss both. Mechanical execution against the sketch + a broken permutation surfaces them deterministically.

### 4. Generalized deferred-decision scan

Scan all step docs (every section, not just the explicit decisions) for ANY of these constructions:
- "either X or Y works"
- "X is fine; Y is also fine"
- "could be A or B"
- "inside or outside [...] either way"
- "may be placed [...] or [...]"

Quote each instance with file and line number. Resolve each with a one-sentence reason or mark as a genuine equivalence (rare). **Pattern recognition is not pattern elimination**: resolving the previously-flagged instance does not prevent a new one from appearing one level down (validated on dark-mode-toggle).

### 5. Step-list-shape match

Compare the produced step list to the roadmap's "Step-level decomposition" section (or equivalent step outline). Any merge/split must be explicitly called out and justified. Silent restructuring hides the change from anyone reading the roadmap.

| Roadmap step | Produced step | Notes if merged/split |
|---|---|---|
| Roadmap step 7: Verification | (merged into step-6 manual criteria) | "Verification absorbed into step-6 because [reason]" |

### 6. File-level conflict check

List every step doc with its Files-touched section. Flag any case where two steps modify the same file with potentially incompatible changes (e.g., both rewrite `src/main.tsx` rather than one editing what the other added).

### 7. Test mapping per AC (§2 rule 8)

For each step doc that ships runtime code (i.e., not configuration-only or static-asset-only steps), verify the step doc includes a `## Test mapping` section per Cross-cutting principles → Testing discipline + §2 rule 8. The section must:

1. List every acceptance criterion from the step doc
2. Pair each AC with a named test (describe block + it block)
3. Specify test type per AC (unit | integration | e2e)
4. Specify the file path where the test lives

Steps that modify only configuration, documentation, or static assets may omit the section with an explicit one-line justification (e.g., "No runtime code — test mapping N/A").

Format check:

```markdown
## Test mapping
| AC | Test name | Test type | File |
|---|---|---|---|
| <AC text> | `<describe > it>` | unit \| integration \| e2e | `src/.../X.test.ts` |
```

ACs with no test mapping = silent ACs that cannot be verified at /verify gate 5. Either add the missing test mapping, or re-examine whether the AC is actually testable as written (per Testing discipline, ACs should be user-observable outcomes; "feature X works" is unmappable). Do not push unmapped ACs to the implementer.

Per Testing discipline → Negative-permutation discipline: each named test in the mapping must be reviewable against the criterion "does this test fail against a deliberately-broken implementation permutation?" The mapping doesn't enforce this at /decompose time (that's /assess's job per question 6), but the test names should be specific enough that the question is answerable (e.g., `selectPostsByTags > returns OR-union for multi-tag input`, not `posts > works correctly`).

### 8. Report block

Write to `docs/roadmap/$1/_consistency.md` (separate file, not appended to a step doc):

```
## Consistency sweep report

- Roadmap requirements with no step coverage: [list or "none"]
- Roadmap decisions with no step implementation: [list or "none"]
- Roadmap deferrals propagated into step docs: [list or "none"]
- Step doc internal consistency evidence: [link to §3 above]
- Deferred-decision constructions found: [list with file:line, or "none"]
- Step-list-shape deviations from roadmap: [list with justification, or "matches"]
- Files modified by multiple steps with potential conflicts: [list or "none"]
- Step docs missing test mapping (per §2 rule 8): [list of step docs that ship runtime code without a `## Test mapping` section, or "none — all runtime-code steps have test mapping"]
- ACs unmapped to specific tests: [list of "step-N AC: <text>" entries with no paired test, or "none — all ACs map to named tests"]
```

If any field is non-empty (other than "none"), **fix the step docs (or the roadmap) before stopping**. Do not push unresolved gaps to the implementer.

## Hard rules

- Do not implement. Output only the step docs and the consistency sweep report.
- Do not silently scope a roadmap requirement down. If you find one is impractical, edit the roadmap explicitly and note the change in the report.
- Do not propagate deferred decisions. The implementer should not be picking between equally-weighted options the planner left undecided.
- Do not produce "no contradictions found" without quoted evidence per step. The check is theater otherwise.

## After consistency sweep — update state file

Per Cross-cutting principles → State management, update `.pi/active.md` with the decomposition results. Edit the file to set:

- `**Total steps**: <N>` (the actual number of step docs produced)
- `**Current step**: 1` (set to first step; auto-advance via `/verify`)
- Append `decompose` to `**Phases completed**:` (also append any critique/revise phases that ran prior)
- Replace the `## Step status` section with a checklist of every step doc produced:
  ```markdown
  ## Step status
  - [ ] step-1-<slug>
  - [ ] step-2-<slug>
  ...
  ```

Use the actual filenames of the step docs you produced.

## Next-step recommendation (output format — required after `_consistency.md` written + active.md updated)

After `docs/roadmap/$1/_consistency.md` is written and `.pi/active.md` is updated with the step list, append exactly this block. **Bare commands only** — slug + step are persisted in `.pi/active.md`.

```markdown
---
**✅ Decomposition complete — `_consistency.md` written, `.pi/active.md` updated with N steps**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/preflight` | **minimal** — Shift+Tab down from medium |
| Then | `/implement` | **medium** |
| Then | `/verify` | **minimal** |
| Then (calibration mode) | `/assess` | **medium** |
```

The cycle repeats per step: preflight → implement → verify → assess (optional). State auto-advances on /verify Pass.

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands. Output `/preflight` — never `/preflight <slug> 1`.
