---
description: Print the active feature status and recommend the next command with thinking level
---
You are running the `/status` command.

## Step 1 — Print active.md

If `.pi/active.md` does not exist:
- Report: *"No active feature. Run `/plan <slug> <topic>` to start one. (thinking: **medium**)"*
- Stop.

If `.pi/active.md` exists: read it and print its contents verbatim.

## Step 2 — Compute and append next-command recommendation

After printing the file, append a `## Next` section computed from the state. Apply the following logic in order; the first matching condition determines the next command.

Per Cross-cutting principles → Thinking levels per phase, the recommended thinking level for each phase is:

| Phase | Thinking |
|---|---|
| `/plan` initial | medium |
| `/critique` (each pass) | high |
| `/revise` | high |
| `/decompose` | medium |
| `/preflight` | minimal |
| `/implement` | medium |
| `/verify` | minimal |
| `/distill` | high |
| `/calibrate` | high |

### Decision tree

Read these fields from `.pi/active.md`: `Status`, `Current step`, `Total steps`, `Phases completed`.

1. **`Status: complete` AND ADR exists in `produced-adrs:` of the roadmap**:
   - Next: `/calibrate` (thinking: **high**) — bare form; resolves to most recent ADR. Or `/calibrate <NNNN>` to be explicit about which ADR.
   - That's the final step; no further commands recommended.

2. **`Phases completed` includes `decompose` AND `Current step` is a number ≤ `Total steps`**:
   - Per-step sequence:
     - Next: `/preflight` (thinking: **minimal**)
     - Then: `/implement` (thinking: **medium**)
     - Then: `/verify` (thinking: **minimal** — auto-advances on PASS)
   - State will auto-update; user can re-run `/status` to confirm advance.

3. **`Phases completed` includes `decompose` AND `Current step` is `complete` (or > `Total steps`)**:
   - Next: manual smoke test in browser (per roadmap §7 / runbook §7)
   - Then: `/distill` (thinking: **high**)

4. **`Phases completed` includes `revise` but not `decompose`**:
   - Next: `/decompose` (thinking: **medium**)

5. **`Phases completed` includes one critique pass but not `revise` and not a second critique**:
   - Next: `/critique <slug> <other-perspective-tag>` (thinking: **high**) — pick whichever of `senior-engineer-risk` / `pm-requirements` / `security-perf` hasn't run yet
   - After 2nd critique: `/revise` (thinking: **high**)

6. **`Phases completed` includes `plan` only**:
   - Next: `/critique <slug> senior-engineer-risk` (thinking: **high**)
   - Then: a 2nd critique with a different perspective tag, then `/revise`.

### Output format

Append after the active.md contents:

```markdown
---

## Next

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `<command>` | **<level>** — Shift+Tab to set if not already there |
| Then | `<command>` | **<level>** |
| Then | `<command>` | **<level>** |

Per `~/.pi/agent/AGENTS.md` Cross-cutting principles → Thinking levels per phase.
```

The `Now` row is the immediate next command. `Then` rows show the subsequent expected sequence so the user can pre-set thinking level for the chain. If only one command is recommended (e.g., `/calibrate` after feature complete), produce a single-row table with just `Now`.

### Hard rules — command form

**Recommend BARE commands wherever the state file resolves all required args.** Including the slug or step number in the recommendation defeats the entire purpose of the state file. The user has explicitly chosen state-file-driven invocation.

| Command | Recommended form | Why |
|---|---|---|
| `/preflight` | `/preflight` | slug + step from state |
| `/implement` | `/implement` | slug + step from state |
| `/verify` | `/verify` | slug + step from state |
| `/decompose` | `/decompose` | slug from state |
| `/revise` | `/revise` | slug from state |
| `/distill` | `/distill` | slug from state |
| `/critique` | `/critique <perspective-tag>` | slug from state; perspective NOT in state, must be explicit |
| `/calibrate` | `/calibrate` (no args; resolves most recent ADR) or `/calibrate <ADR-NNNN>` to pin a specific ADR | slug derived from ADR's `distilled-from:` frontmatter, not from state file (which `/distill` already deleted) |
| `/plan` | `/plan <slug> <topic>` | slug starts the loop; can't be from state (no state yet) |

Do NOT produce recommendations like `/preflight dark-mode-toggle 3` — the user will not type the redundant args. If a command works bare, recommend it bare. If a command needs a non-slug arg (perspective for critique, ADR number for calibrate), recommend the single-arg form, not the two-arg form.

If you find yourself about to suggest two args for a command that has state coverage, stop — that's the bug this rule was added to prevent.
