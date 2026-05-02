---
description: Cold-start implementation from a step doc
argument-hint: "<slug> <step-N>"
---
You are running the **Implementation** phase from `~/.pi/agent/AGENTS.md` § 3-4. Treat this as a **cold start** — file system is the memory; do not assume conversation context.

Set thinking level to **medium** per Cross-cutting principles → Thinking levels per phase. Mostly transcription from the step doc; thinking helps but synthesis demand is low.

## Resolve active feature (if args missing)

Per Cross-cutting principles → State management: if `$1` is empty, read slug from `.pi/active.md`. If `$2` is empty, read `Current step:` from same file (default to 1 if null). Explicit args override.

Slug: $1
Step number: $2

Step doc: discover via `bash ls docs/roadmap/$1/step-$2-*.md` — should match exactly one file. Report the matching path before reading it.

## Cold-start procedure

1. Read the step doc end to end.
2. Read any pre-flight reports that exist for context:
   - `docs/roadmap/$1/_preflight.md` (holistic — covers all steps)
   - `docs/roadmap/$1/step-$2.preflight.md` (focused — covers this step specifically)
   These are the "what's already verified" reference. Trust them unless something in the step doc or codebase contradicts. Don't re-verify what's already documented; do flag any contradictions you find.
3. Read every file the step doc lists under "Files touched" and "API/function signatures".
4. Run `bash git diff main` (or `bash git status` if not a git repo) and note state so far.
5. **Constraints contract (BEFORE writing any code).** Output a table covering every constraint from the step doc's "Constraints" section. Map each constraint to the snippet that should satisfy it:

   | # | Constraint (quoted from step doc) | Satisfied by snippet? | Where in snippet (line range or "missing") |
   |---|---|---|---|
   | 1 | "..." | Yes / No | lines N-M of step doc |

   If any constraint shows "No" or "missing" — **STOP and report**. The snippet does not satisfy its own constraints; the step doc has a bug. **Do not silently fix the snippet.** Flag the gap for the user to update the step doc and re-run.

6. Implement the step. Transcribe the snippet — same logic, same defensive code, same comments where present. Do not "clean up" or simplify.

7. **Constraint coverage check (AFTER writing, BEFORE claiming done).** Recompute the table against the produced code:

   | # | Constraint | Satisfied by output? | Evidence (file:line range from your written code) |
   |---|---|---|---|
   | 1 | "..." | Yes / No | `src/lib/theme.ts:8-12` (quote the relevant lines) |

   If any constraint shows "No" — you violated the step doc. **STOP and report.** Do not claim success.

## Hard rules

- After each file change: run the verification commands listed in the step doc and report results verbatim.
- Per Cross-cutting principles → Evidence over claims: do not claim success until verification passes. If a check fails, stop and report — do not "helpfully" press on.
- **Invariants check before claiming done**: list each invariant from the step doc and state explicitly how you verified it's still preserved. Quote the file/line/command output that proves preservation.
- **Do not "improve" code mid-write.** Steps 5 and 7 of the cold-start procedure are the enforcement mechanism. If the step doc has an implementation snippet, transcribe it. Defensive code (try/catch, null checks, fallbacks), comments, and verbose patterns are intentional — they exist because the constraints required them. If you find yourself simplifying, you are removing constraint coverage. **Stop and report.**

## Chunk cap

Aim for 15-20 tool calls before stopping to recompose. If you hit the wall (repeating earlier work, inventing APIs, refactoring code you wrote earlier in the session), stop and tell the user to decompose further.

## Next-step recommendation (output format — required after /implement)

After /implement completes (whether through STOP-and-report or successful sketch transcription), append exactly this block. **Bare commands only** — slug + step are persisted in `.pi/active.md`.

```markdown
---
**✅ Implement complete — ready for /verify**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/verify` | **minimal** — Shift+Tab down from medium; auto-advances on PASS |
| Then (calibration mode) | `/assess` | **medium** |
```

If /implement STOPped due to lint failure, sketch defect, or /verify pre-condition failure, follow the STOP-and-report path — do NOT advance to /verify until the underlying defect is resolved (re-run /implement after step doc revision OR after fix-and-retry).

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands. Output `/verify` — never `/verify <slug> <N>`.
