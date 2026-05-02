---
description: Ship a completed feature — rebase feature branch onto main, ff-only merge, remove worktree
argument-hint: "[slug]"
---
You are running the **Finalize** phase from `~/.pi/agent/AGENTS.md` § 8 (Post-ship git operations). Ships a completed feature branch back to main via rebase + ff-only merge, then cleans up the worktree.

Set thinking level to **medium** per Cross-cutting principles → Thinking levels per phase. Mostly mechanical git operations with stop-on-error semantics; thinking is for evaluating pre-flight gate outputs and conflict resolution recommendations.

## Step 0 — Argument resolution

Per Cross-cutting principles → State management: explicit args always override the state file.

**Slug resolution** (`$1`):
- If `$1` is empty: read slug from `.pi/active.md` in the current worktree (`bash sed -n 's/^# Active feature: //p' .pi/active.md`). If file missing, error: *"No active feature in this worktree. Pass slug explicitly or run /finalize from inside the feature's worktree."*
- If `$1` is provided: treat as slug.

Derive paths:

```bash
WORKTREE_PATH=$(pwd)
ORIG_REPO=$(git worktree list --porcelain | awk '/^worktree/{print $2; exit}')
BRANCH="feature/$1"
MAIN_BRANCH="main"   # adjust to "master" if the project uses master
```

`ORIG_REPO` is the FIRST worktree listed by `git worktree list` — the original/main checkout where the feature merges back. The worktree this template runs in is a sibling.

If `ORIG_REPO == WORKTREE_PATH`, error: *"/finalize must be run from inside the feature worktree, not the original repo. cd to the worktree first."*

## Step 0.5 — Auto-commit feature deliverables

The doctrine's "verified changes remain uncommitted by default" principle (Calibration notes, plus user-confirms-before-commit discipline) is for in-flight verification cycles — at /finalize time, the worktree contents ARE the feature, and committing them is part of shipping. Without auto-commit here, every /finalize requires a manual git commit step preceding it, adding HITL friction at scale.

Procedure:

1. Check if worktree has uncommitted changes:

```bash
git -C "$WORKTREE_PATH" status --porcelain
```

If empty (worktree already clean), skip to Step 1 (Pre-flight gates) — likely the user pre-committed manually for fine-grained commit control.

2. If non-empty, auto-commit ALL changes with a templated message. Resolve ADR number(s) from roadmap's `produced-adrs:` frontmatter at `<WORKTREE_PATH>/docs/roadmap/$1/$1.md`:

```bash
ADR_LIST=$(awk '/^produced-adrs:/{getline; gsub(/[\[\]]/,""); print}' "$WORKTREE_PATH/docs/roadmap/$1/$1.md" 2>/dev/null || echo "<none>")
git -C "$WORKTREE_PATH" add -A
git -C "$WORKTREE_PATH" commit -m "feat($1): ship feature

ADR(s): $ADR_LIST
Roadmap: docs/roadmap/$1/$1.md
Step docs: docs/roadmap/$1/
Source + tests: per step docs files-touched sections
Calibration: docs/calibration/trace-$1.md (if exists) + verify outputs (if persisted)"
```

`git add -A` is appropriate here because the worktree exists for THIS feature only — anything in the working tree is feature-related by construction. (If the user has made unrelated changes in the worktree, that's a usage error outside /finalize's responsibility.)

If commit fails (e.g., pre-commit hook rejects the changes), stop and report the hook output verbatim. Do NOT bypass hooks (`--no-verify` is forbidden per the doctrine principle "investigate root causes rather than bypass safety checks").

3. Proceed to Step 1 (Pre-flight gates). Gate 1.1 should now pass.

## Step 1 — Pre-flight gates (any failure stops the operation)

All gates must pass. On any failure, stop and report. Do NOT proceed to step 2 — no partial state.

### Gate 1.1 — Worktree clean (post auto-commit sanity check)

```bash
test -z "$(git -C "$WORKTREE_PATH" status --porcelain)"
```

After Step 0.5's auto-commit, this should pass. If it does NOT pass, something interfered between Step 0.5 and this gate (uncommitted changes appeared mid-flight, or the auto-commit failed silently). Stop with: *"Worktree still unclean after Step 0.5 auto-commit. Investigate manually: git status output may show files that git add -A did not stage (e.g., gitignored or submodule state)."*

### Gate 1.2 — All step statuses complete

Read `<WORKTREE_PATH>/.pi/active.md`. Every line matching `^- \[` must start with `- [x]` (no unchecked `[ ]`).

On any unchecked: report which step(s) and stop with: *"Step <N> not yet verified. Run /verify first."*

### Gate 1.3 — ADR exists

Read roadmap frontmatter at `<WORKTREE_PATH>/docs/roadmap/$1/$1.md`. Find `produced-adrs:` field. For each ADR number listed, confirm `<WORKTREE_PATH>/docs/adr/<NNNN>-*.md` exists.

On missing ADR or empty `produced-adrs:`: stop with: *"No ADR linked to this feature. Run /distill first."*

### Gate 1.4 — Current branch matches feature branch

```bash
git -C "$WORKTREE_PATH" rev-parse --abbrev-ref HEAD
```

Must equal `feature/$1`. On mismatch: stop with: *"Current branch is <X>, expected feature/$1. Are you in the right worktree?"*

### Gate 1.5 — Original repo is on main

```bash
git -C "$ORIG_REPO" rev-parse --abbrev-ref HEAD
```

Must equal `$MAIN_BRANCH`. On mismatch: stop with: *"Original repo at <ORIG_REPO> is on branch <X>, expected $MAIN_BRANCH. Switch to $MAIN_BRANCH before /finalize: git -C <ORIG_REPO> checkout $MAIN_BRANCH"*

If all 5 gates pass, proceed.

## Step 2 — Rebase feature branch onto origin/main

First, detect if a rebase is already in progress (e.g., from a prior failed `/finalize` or from `pi-orchestrate-conflict`):

```bash
git -C "$WORKTREE_PATH" status
```

If `git status` reports *"rebase in progress"*, skip the `git rebase` command below and jump directly to § 2a — treat any `UU` files as the conflict to resolve. After resolving and continuing, re-run `git status` to confirm the rebase completed; if it still shows *"rebase in progress"*, loop back to § 2a again.

If no rebase is in progress, start one:

```bash
git -C "$WORKTREE_PATH" fetch origin
git -C "$WORKTREE_PATH" rebase "origin/$MAIN_BRANCH"
```

On rebase conflict: capture `git status` output listing conflicted files. For each conflicted file, invoke **autonomous conflict resolution** (see § 2a below). If any conflict is non-textual (binary, deleted-by-us/them, rename/add, etc.) or the model cannot determine a sensible merge, stop with:

> *Unresolvable rebase conflict on files: `<list>`.*
> *Resolve manually inside the worktree:*
> *  `cd <WORKTREE_PATH>`*
> *  `git status` — see conflict details*
> *  Edit files to resolve, then:*
> *  `git add <resolved-files>`*
> *  `GIT_EDITOR=true git rebase --continue`*
> *Then re-run /finalize.*

If all conflicts are resolved by the autonomous procedure, proceed.

### § 2a — Autonomous conflict resolution protocol

When `git rebase` stops with conflicts, execute this protocol for every file reported by `git status --short` as `UU` (or `AA`, `AU`, `UA`):

1. **Read the conflicted file** using the Read tool.
2. **Parse conflict markers** (`<<<<<<< HEAD`, `=======`, `>>>>>>> <branch>`).
3. **Resolve strategy** (choose one per hunk, based on context):
   - If one side is pure deletion and the other is pure addition, keep the addition (the deletion is from the already-merged `HEAD` side; the feature side is the new work).
   - If both sides changed the same line but the changes are semantically independent (e.g., different props, different functions), produce a merged hunk containing **both** changes in a sensible order.
   - If both sides changed the same line in incompatible ways (e.g., different values for the same variable), prefer **theirs** (the feature branch) because `HEAD` already landed and the feature is what we are rebasing onto it.
   - If the conflict is in a generated lockfile (`package-lock.json`, `Cargo.lock`, etc.), prefer **theirs** and plan to regenerate/validate in a later step (`npm install`, etc.).
4. **Write the resolved file** using the Edit tool (or Write for multi-hunk rewrites), removing all conflict markers.
5. **Stage the file**: `git add <file>`.
6. **Continue the rebase**: `GIT_EDITOR=true git rebase --continue`.

After continuing, the rebase may stop again with more conflicts (e.g., stacked commits). Loop back to Step 1 until `git status` shows no conflicted files.

**Stop and escalate** if:
- The file is binary (cannot Read as text).
- The conflict markers are malformed or nested in a way that automated parsing fails.
- The resolution would delete user-visible functionality (not just boilerplate/imports).
- `git rebase --continue` fails for a reason other than new conflicts (e.g., commit-message editor, pre-commit hook failure).

On clean rebase, proceed.

## Step 2.5 — ADR collision resolution (after rebase, before merge)

Parallel features may have claimed the same ADR number during `/distill`. After rebase, check if the feature's ADR(s) collide with existing ADRs on main.

```bash
# List ADR numbers in the feature branch that are NEW (not on main)
FEATURE_ADRS="$(git -C "$WORKTREE_PATH" diff --name-only origin/$MAIN_BRANCH -- docs/adr/ | grep -oE '^docs/adr/[0-9]{4}-' | sed 's/docs.adr.//' | sed 's/-$//')"

# List ADR numbers already on main
MAIN_ADRS="$(git -C "$ORIG_REPO" ls-tree -r --name-only origin/$MAIN_BRANCH -- docs/adr/ 2>/dev/null | grep -oE '^docs/adr/[0-9]{4}-' | sed 's/docs.adr.//' | sed 's/-$//')"

for adr in $FEATURE_ADRS; do
  if echo "$MAIN_ADRS" | grep -q "^${adr}$"; then
    # Collision: find next available number
    NEXT="$(echo "$MAIN_ADRS" | sort -n | tail -1)"
    NEXT=$((NEXT + 1))
    NEXT_PADDED=$(printf "%04d" $NEXT)
    
    # Rename the ADR file in the feature branch
    OLD_FILE="$(git -C "$WORKTREE_PATH" diff --name-only origin/$MAIN_BRANCH -- docs/adr/ | grep "^docs/adr/${adr}-")"
    NEW_FILE="${OLD_FILE/${adr}/${NEXT_PADDED}}"
    git -C "$WORKTREE_PATH" mv "$OLD_FILE" "$NEW_FILE"
    
    # Update the roadmap reference
    ROADMAP="$WORKTREE_PATH/docs/roadmap/$1/$1.md"
    # Replace the old ADR number with the new one in the roadmap frontmatter/footer
    sed -i "s/${adr}/${NEXT_PADDED}/g" "$ROADMAP"
    git -C "$WORKTREE_PATH" add "$ROADMAP" "$NEW_FILE"
    
    # Amend the last commit (the distill commit)
    git -C "$WORKTREE_PATH" commit --amend --no-edit
    
    echo "ADR collision resolved: ${adr} → ${NEXT_PADDED}"
  fi
done
```

If any collision was resolved, the rebase history has been amended. The merge will proceed with the corrected ADR numbers.

## Step 3 — Fast-forward merge to main

```bash
git -C "$ORIG_REPO" merge --ff-only "feature/$1"
```

`--ff-only` enforces clean linear history. If it fails, main moved during steps 2-3 (rare). Stop with: *"Fast-forward merge failed — main moved during finalize. Re-rebase: cd <WORKTREE_PATH>; git fetch; git rebase origin/$MAIN_BRANCH; then re-run /finalize."*

On success, proceed.

## Step 4 — Cleanup

**CRITICAL ORDER:** The worktree removal (4c) MUST be the last bash command. After the worktree directory is deleted, the pi harness cannot execute further bash commands because its current working directory no longer exists. All other git operations (branch deletion, footer commit) must run BEFORE removing the worktree.

### 4a — Branch deletion (auto)

After successful ff-only merge, the feature branch is redundant — all commits are on main. Delete it automatically from inside the worktree (worktrees share the original repo's object database):

```bash
git -C "$ORIG_REPO" branch -d "feature/$1"
```

If deletion fails (e.g., branch was not fully merged due to an unexpected state), stop and report. This is safe because `--ff-only` was verified in Step 3.

### 4b — Append shipped footer to roadmap and commit

In the original repo on main, append to `<ORIG_REPO>/docs/roadmap/$1/$1.md` (the file is now in main via the merge). Use the Edit tool to add at the end of the file:

```
---

**Shipped**: <today YYYY-MM-DD> | **Branch**: feature/$1 | **ADRs**: <comma-separated NNNN list from frontmatter>
```

Then commit the footer automatically:

```bash
git -C <ORIG_REPO> add docs/roadmap/$1/$1.md && git -C <ORIG_REPO> commit -m "docs($1): shipped footer"
```

The footer is generated bookkeeping (timestamp, branch, ADR list) — low-risk, auto-committed by default. If code quality degrades, re-enable HITL review of footers.

### 4c — Remove worktree and delete branch (terminal step)

The shell's current working directory is inside the worktree. After the worktree directory is deleted, the pi harness cannot spawn new processes from a deleted CWD. **All remaining git operations must therefore run from `$ORIG_REPO` in a single bash command.**

```bash
cd "$ORIG_REPO" && git worktree remove "$WORKTREE_PATH" && git branch -d "feature/$1"
```

This single command:
1. Moves the shell to the original repo (valid CWD).
2. Removes the worktree directory and its git metadata.
3. Deletes the now-redundant feature branch (deletable because the worktree no longer holds it).

After execution, the pi session is still in `$ORIG_REPO` — a valid directory — so any follow-up bash commands (e.g., verification, pushing) can continue. The feature worktree is gone.

## Step 5 — Print success summary

```
✅ Feature $1 shipped.

Branch: feature/$1 (deleted)
Worktree: removed (was at <WORKTREE_PATH>)
Roadmap: <ORIG_REPO>/docs/roadmap/$1/$1.md (footer committed)
ADR(s): <comma-separated list>

Optional next step:
  git -C <ORIG_REPO> push origin $MAIN_BRANCH    # push to remote when ready
```

## Hard rules

- **No auto-push.** Per doctrine §8, /finalize stops after the local merge. User confirms before push.
- **Conflict auto-resolution is attempted for textual `UU` files.** Binary, structural, or ambiguous conflicts stop for manual resolution.
- **Footer is auto-committed.** Generated bookkeeping (timestamp, branch, ADR list) — low risk. Re-enable HITL review if code quality degrades.
- **Branch is auto-deleted after ff-only merge.** Redundant once merged; recoverable via reflog if needed. Deletion happens inside the same bash command as worktree removal, after `cd "$ORIG_REPO"`, to avoid a deleted CWD.
- **Pre-flight gates are non-negotiable.** Any failure halts the entire operation. Do not skip gates "to save time."
- **`--ff-only` is non-negotiable.** Any merge that would require a merge commit means the rebase missed upstream changes. Re-rebase rather than abandon ff-only.
- **Run from inside the worktree.** /finalize uses `pwd` to determine the worktree path. If run from elsewhere, error per Step 0.

## Next-step recommendation (output format — required after success summary)

After the success summary is printed, append exactly this block. /finalize is terminal in the workflow — there is no automatic next phase per feature.

```markdown
---
**✅ Feature $1 shipped — worktree removed, footer committed**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | (optional) `git push origin main` | n/a |
| Then | `/plan <new-slug> "<topic>"` — start the next feature in a new worktree | **medium** |

The current pi session in this worktree is no longer connected — the worktree was just removed. Open a new pi session in the original repo or in a new worktree directory.
```

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands wherever the state file resolves all required args. /finalize defaults to slug-from-active.md; pass `<slug>` explicitly only when needed (e.g., active.md is corrupted). Output `/finalize` — never `/finalize <slug>` when the state file already resolves it.
