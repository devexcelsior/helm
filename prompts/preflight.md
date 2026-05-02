---
description: Hallucination defense pre-flight; writes a persisted report for /implement to read
argument-hint: "<slug> [<step-N>]"
---
You are running the **Pre-flight** phase from `~/.pi/agent/AGENTS.md` § 6.

Set thinking level to **minimal** per Cross-cutting principles → Thinking levels per phase. Mechanical verification, not synthesis.

## Step -1 — Resolve active feature (if `$1` missing)

Per Cross-cutting principles → State management: if `$1` is empty, read slug from `.pi/active.md` (`bash sed -n 's/^# Active feature: //p' .pi/active.md`). Standard error if missing.

If `$1` is missing AND `$2` is missing AND `.pi/active.md` exists with a non-null `Current step`: also resolve `$2` to that current step. This makes `/preflight` (no args) auto-target the next-undone step. Explicit args always override.

## Step 0 — Mode determination (DO THIS FIRST, before any other action)

Determine mode by checking whether `$2` is set (after Step -1 resolution).

- If `$2` is set (e.g., `/preflight dark-mode-toggle 2` or resolved from state): **mode = focused**.
- If `$2` is empty (e.g., `/preflight dark-mode-toggle` and no Current step in state): **mode = holistic**.

**State your mode determination at the top of your response before doing anything else:** `Mode: focused (step $2)` or `Mode: holistic (all steps)`.

The two modes have DIFFERENT inputs, DIFFERENT output paths, and DIFFERENT report content. Do not mix them. Do not default to holistic when `$2` is provided — that's a template violation.

---

## If mode = focused (typical, run before each `/implement`)

**Inputs**:
- Resolve THE single step doc: `bash ls docs/roadmap/$1/step-$2-*.md`
- Expect exactly one match. If zero or multiple matches, stop and report.
- Read ONLY that one step doc. Do NOT read other step docs.

**Output path**: `docs/roadmap/$1/step-$2.preflight.md`

**Report frontmatter**:
```
**Date**: YYYY-MM-DD
**Mode**: focused (step $2)
**Step doc**: docs/roadmap/$1/step-$2-<resolved-slug>.md
```

**Report scope**: only the files, APIs, imports, and constraints relevant to THIS ONE step. Do not list step docs other than the one you're verifying.

---

## If mode = holistic (rare, before the loop starts as a sanity check)

**Inputs**:
- List all step docs: `bash ls docs/roadmap/$1/step-*.md`
- Read each.

**Output path**: `docs/roadmap/$1/_preflight.md`

**Report frontmatter**:
```
**Date**: YYYY-MM-DD
**Mode**: holistic (all steps)
**Step docs**:
  - docs/roadmap/$1/step-1-...md
  - docs/roadmap/$1/step-2-...md
  - ...
```

**Report scope**: union of all steps' files, APIs, imports, and constraints.

---

## Pre-flight checks (apply within whichever mode you selected)

For the relevant step doc(s):

1. **Files to touch.** Confirm each path exists with `bash ls <path>` or `bash test -e <path>`. Quote the output. Mark new files (will be created) explicitly.
2. **Functions / APIs to call.** `bash grep -n '<symbol>' <file>` for each one to confirm it exists at the path you think. Quote the matching line.
3. **Imports to add.** `bash grep -n '"<package>"' package.json` (or `requirements.txt` / `Cargo.toml` / etc.) to confirm each package is declared. Quote the matching line and version.
4. **Project constraints affecting implementation.** Surface relevant tsconfig / lint / framework settings that impose patterns (e.g., `verbatimModuleSyntax: true` → must use `import type`, `noUnusedLocals: true` → no dead vars).

## Required report sections (after frontmatter)

```markdown
## 1. Files to touch — existence verified

| File | Status | Notes |
|---|---|---|
| <path> | ✅ Exists / ⚠️ New file | <one-line context> |

## 2. Functions / APIs to call — existence verified

| API | Source | Status | Evidence |
|---|---|---|---|
| <symbol> | <package or DOM> | ✅ | <file:line from grep> |

## 3. Imports to add — package availability verified

| Import | Package | In manifest? | Notes |
|---|---|---|---|
| <symbol> | <package> | ✅ ^<version> | <one-line context> |

## 4. Project constraints affecting implementation

| Setting | Value | Impact |
|---|---|---|
| <name> | <value> | <one-line implication for code style> |

## 5. Verdict

Pass: "ready to /implement <slug> <step-N>" with any caveats.
Fail: explicit list of what's missing, where, and what to do about it.
```

Per Cross-cutting principles → Evidence over claims, every ✅ must have file:line evidence quoted from actual `grep`/`ls` output. "I checked and it's fine" is not evidence — quote the line.

## Hard rules

- **Mode determination is non-negotiable.** If `$2` is set, you are in focused mode. Do not produce holistic output. Do not write to `_preflight.md`.
- **Output path matches mode.** Focused → `step-<N>.preflight.md`. Holistic → `_preflight.md`. No exceptions.
- **If any check fails, stop and re-plan.** Do not invent APIs to fill the gap. Do not write the report claiming Pass when items failed — write Fail with the specific gaps named.

## Next-step recommendation (output format — required after preflight Pass)

After preflight reports Pass, append exactly this block. **Bare commands only** — slug + step are persisted in `.pi/active.md`.

```markdown
---
**✅ Preflight Pass — ready for /implement**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/implement` | **medium** — Shift+Tab up from minimal |
| Then | `/verify` | **minimal** |
| Then (calibration mode) | `/assess` | **medium** |
```

`/assess` is recommended for calibration-mode runs; skippable for production. The persisted preflight at `step-<N>.preflight.md` is read by `/implement` as cold-start context.

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands wherever the state file resolves all required args. Output `/implement` — never `/implement <slug> <N>`.
