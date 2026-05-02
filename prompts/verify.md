---
description: Invariant gate before claiming a change is done
argument-hint: "[<slug> <step-N>]"
---
You are running the **Verification gate** phase from `~/.pi/agent/AGENTS.md` § 5.

Set thinking level to **minimal** per Cross-cutting principles → Thinking levels per phase. Mechanical check.

## Resolve active feature

Per Cross-cutting principles → State management: if `$1` is empty, read slug from `.pi/active.md`. If `$2` is empty, read `Current step:` from same file (default to 1 if null). Explicit args override.

If args resolved successfully:
- Read the step doc: `bash ls docs/roadmap/$1/step-$2-*.md` (glob; expect exactly one match). Report the matching path.

If args unresolvable (no state file, no explicit args), use the ambient context (most recently implemented step) and proceed without auto-advance.

## Mechanical constraint checks (run FIRST — binding answer)

If the step doc has a `## Mechanical constraint checks` section: run every check command via `bash`, capture exit code, and report verbatim.

| # | Check command | Exit code | Verdict |
|---|---|---|---|
| 1 | `<command from step doc>` | 0 / non-zero | Pass / Fail |

Per Cross-cutting principles → Mechanical verification for defensive-code constraints, **mechanical checks are the binding answer**. Any non-zero exit fails the entire verification. Do not proceed to model-based gates if any mechanical check fails — fail-closed and report.

Failed mechanical check means: implementation does not satisfy a constraint the step doc explicitly required (verified mechanically). Either the implementation is wrong (re-run `/implement`) or the constraint/check is wrong (update step doc and re-run `/decompose` consistency sweep). Do NOT relax the check to make verification pass.

If the step doc has no `## Mechanical constraint checks` section, that is itself a step-doc defect when defensive-code constraints exist. Flag and proceed to model-based gates with a noted gap.

## Constraint gate (run AFTER mechanical checks pass)

For constraints that do NOT have mechanical checks (architectural, UX, naming, etc. — see Cross-cutting principles), produce the model-based gate table:

| # | Constraint (quoted from step doc) | Satisfied? | Evidence (file:line range) |
|---|---|---|---|
| 1 | "..." | Yes / No | `<file>:<line-range>` quoted from actual code |

Per Cross-cutting principles → Evidence over claims, "Yes" requires the file:line that demonstrates satisfaction. If any constraint shows "No" or "Not found" — **fail verification**.

This gate handles constraints that mechanical checks cannot enforce (architectural shape, UX, naming conventions). The mechanical-first ordering ensures that the easy-to-falsify constraints are caught by deterministic execution before the model-based ones are reached.

## Invariant gate

List the invariants the change must preserve, drawn from the step doc's "Invariants to preserve" section:
- Function signatures
- Public APIs
- Existing test names
- Side-effect contracts
- Performance characteristics
- Anything else the step doc names

Per Cross-cutting principles → Evidence over claims, for each invariant state explicitly **how you verified** it's still preserved:
- Which file you re-read (with line)
- Which test you ran (with output)
- Which `bash grep` returned what

Do not proceed without naming each. "Verified" without evidence is theater.

## Verification commands

Run every verification command listed in the step doc's "Verification commands" section. Report results verbatim, including failures.

## Hard rules

- Do not claim pass without command output.
- If any check fails, stop and report. Do not modify code to make verification pass — if verification reveals a real bug, that's a step-doc or implementation issue; the next step is `/implement $1 $2` rerun, not patch-and-claim.

## Auto-advance state file (on PASS only)

Per Cross-cutting principles → State management, when verification PASSES across all gates AND `$1` and `$2` were resolved (either explicit or from state file):

1. Edit `.pi/active.md`:
   - In `## Step status`, change `- [ ] step-$2-<slug>` to `- [x] step-$2-<slug> — verified <YYYY-MM-DD HH:MM>` (use today's date and current time)
   - Update `**Current step**: $2` to `**Current step**: <$2 + 1>` (incremented), or to `complete` if `$2 == total_steps`
2. Use the Edit tool for these changes. Do not rewrite the whole file; surgical edits only.

If verification FAILED on any gate: do NOT update `.pi/active.md`. Report the failure and stop.

If `$1`/`$2` were not resolvable (ambient-context fallback): skip auto-advance. State updates require explicit step identity.

## Next-step recommendation (output format — required on PASS)

After /verify PASS and state-file auto-advance, append exactly this block to your output. **Bare commands only** — slug and step are persisted in `.pi/active.md`; do NOT include redundant args.

If `Current step` was advanced to a number ≤ `Total steps` (mid-feature):

```markdown
---
**✅ Step $2 verified — `Current step` advanced to <$2 + 1> in `.pi/active.md`**

| Phase | Command | Thinking |
|---|---|---|
| **Now** (calibration mode) | `/assess` | **medium** — Shift+Tab up from minimal |
| Then | `/preflight` | **minimal** — Shift+Tab down from medium |
| Then | `/implement` | **medium** |
| Then | `/verify` | **minimal** |
```

`/assess` is recommended for calibration-mode runs (baby projects validating the harness); skippable for production runs where ceremony per step is unwanted. If skipping, omit the first row.

If `$2` was the final step (`Current step` now `complete`):

```markdown
---
**✅ Step $2 verified — feature complete**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | manual smoke test in browser | n/a |
| Then | `/distill` | **high** |
| Then | `/calibrate` | **high** |
```

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands wherever the state file resolves all required args. Output like `/preflight routing-skeleton 4` is wrong — slug and step are in `.pi/active.md`. Output `/preflight` (thinking: **minimal**) instead. Including the args defeats the state-file-driven invocation.
