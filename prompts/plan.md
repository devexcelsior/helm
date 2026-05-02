---
description: Planning pass with 6-field reasoning template + cross-cutting concerns check
argument-hint: "<slug> <topic-description>"
---
You are running the **Planning** phase from `~/.pi/agent/AGENTS.md` § 1.

Slug: $1
Topic: ${@:2}

Set thinking level to **medium** per Cross-cutting principles → Thinking levels per phase. Initial planning is structured output (decomposing requirements into decisions); high thinking is reserved for the revision pass via `/revise`.

## Branch + worktree setup (per workflow doctrine §8 — Post-ship git operations)

Run before writing the roadmap. Sets up a feature branch + sibling git worktree so this feature can be worked on in parallel with other features without git state collision.

### 1. Detect git repo

```bash
git rev-parse --is-inside-work-tree 2>/dev/null
```

If not a git repo → skip steps 2–4, fall back to in-place behavior: roadmap goes to `docs/roadmap/$1/$1.md` in current dir, active.md to `.pi/active.md` in current dir, with `Branch: <none>` and `Worktree: $(pwd)`.

If git repo detected, continue.

### 2. Pre-flight collision check (atomic — refuse before any mutation)

Resolve paths:

```bash
REPO_PATH=$(git rev-parse --show-toplevel)
WORKTREE_PATH="$REPO_PATH/../$(basename "$REPO_PATH")-$1"
```

Refuse if either exists:

- `git branch --list "feature/$1" | grep -q .` exits 0 → branch already exists. Stop with error: *"Branch `feature/$1` already exists. Either pick a different slug, or `git branch -D feature/$1` to remove the existing branch first."*
- `test -d "$WORKTREE_PATH"` exits 0 → worktree path already exists. Stop with error: *"Worktree path `$WORKTREE_PATH` already exists. Either pick a different slug, or `rm -rf "$WORKTREE_PATH"` to clear the path first."*

If either check trips, do NOT proceed to step 3 — no partial state.

### 3. Create branch + worktree

```bash
git worktree add "$WORKTREE_PATH" -b "feature/$1"
```

Atomic: creates the branch and a working directory rooted at HEAD of the current branch.

### 4. Write roadmap and active.md INSIDE the worktree

Output paths shift to the worktree (NOT the original repo):

- Roadmap: `$WORKTREE_PATH/docs/roadmap/$1/$1.md`
- State file: `$WORKTREE_PATH/.pi/active.md`

The original repo's working tree on main is not modified. The roadmap exists only in the feature branch's worktree until `/finalize` merges back to main.

### 5. Print user instructions to switch pi sessions

After files are written, output exactly:

```
Worktree created: <WORKTREE_PATH absolute>
Roadmap: <WORKTREE_PATH>/docs/roadmap/$1/$1.md
State: <WORKTREE_PATH>/.pi/active.md

Open a new pi session in the worktree to continue:
  cd <WORKTREE_PATH>
  pi

Then run /critique <perspective-tag> to start the critique cycle.
Do not continue this feature in the original repo's pi session — its .pi/active.md is for whichever feature (if any) was active there before, not for $1.
```

After printing, stop. Do not run additional commands or write further files in the original repo.

---

Roadmap output path is determined above: `$WORKTREE_PATH/docs/roadmap/$1/$1.md` if a worktree was created, otherwise `docs/roadmap/$1/$1.md` in the current dir. Create directories as needed.

## Reasoning template (per major decision)

For each major decision in the plan, output:

1. **Decision**: what is being decided
2. **Constraints**: what must hold true; cite specific files/code with line numbers (per Cross-cutting principles → Evidence over claims)
3. **Second-order effects**: what else changes downstream; which other steps does this affect; what new constraints does it create
4. **Failure modes**: what could go wrong. **Name the deceptively obvious wrong answer and why it's wrong** — pause before the obvious choice. Real wrong answer, not strawman.
5. **Alternatives considered**: at least 2; explicit reason for rejecting each
6. **Self-consistency check**: does this contradict any earlier decision in this plan? Quote the prior decision before answering. Resolve contradictions here — don't defer.

Apply at the **phase level** for high-level planning. The structure makes implicit reasoning explicit — readable, diffable, regression-testable.

## Cross-cutting concerns (mandatory section in the plan)

Before finalizing, the plan must address each cross-cutting concern below. Either resolve with a Decision, mark out-of-scope with rationale, or accept default with rationale. Missing decisions are implicit decisions someone else will make later.

- Testing strategy — apply Cross-cutting principles → Testing discipline. Specifically: (a) selector preference `findByRole({ name: ... })` over `findByText(...)` to avoid substring-match defects; (b) mock state construction via entity-adapter API only (`postsAdapter.setAll(...)`), no hand-rolled `{ ids, entities }`; (c) brittleness avoidance — no snapshot tests, no arbitrary-timeout `waitFor`, no internal-state assertions; (d) negative-permutation discipline — tests must fail against deliberately-broken implementation permutations. Acceptance criteria must be structured as user-observable outcomes (so /decompose can produce per-AC test mapping per §2 rule 8).
- Accessibility (ARIA, keyboard, contrast, screen-reader)
- Error handling (boundaries, fallbacks, user-facing messaging)
- Observability (logging, telemetry, error reporting)
- Deployment (build artifacts, hosting, CDN)
- Security (CSP, input validation, secrets, auth)
- Performance budgets (bundle size, render time, network)

## Discipline-axis self-check (before declaring done)

Scan your own plan for the four failures `/critique` will surface anyway. Catching them in self-check saves a critique pass.

1. **Deferred decisions**: any "either X or Y works" / "may be inside or outside" / "could be A or B" construction → pick one with a one-sentence reason.
2. **Unjustified specifics**: any concrete choice without defense (storage key namespace, timeout value, library version pin) → add the defense or remove the specificity.
3. **Circular rejection reasoning**: alternatives rejected because of a constraint you set yourself → tighten or replace the rejection reason.
4. **Missing cross-cutting concerns**: per the section above. Each concern present in the plan or explicitly out-of-scope.

If you catch any in self-check, fix before output. If `/critique` catches them anyway, that's calibration data.

## After writing the plan — initialize state file

Per Cross-cutting principles → State management (per-worktree), write `.pi/active.md` inside the resolved location (worktree if step 3 above ran; current dir otherwise) with this content:

```markdown
# Active feature: $1

- **Started**: <today YYYY-MM-DD>
- **Branch**: <feature/$1 if worktree was created, else "<none>">
- **Worktree**: <absolute WORKTREE_PATH if created, else $(pwd)>
- **Status**: in_progress
- **Current step**: null
- **Total steps**: null
- **Phases completed**: plan

## Step status

(Populated by `/decompose`.)
```

Total steps and step status remain null until `/decompose` runs. The state file is per-worktree, enabling parallel feature work without state collision per doctrine §8.

## Hard rules

- Do not implement. Planning only.
- Per Cross-cutting principles → Evidence over claims, every constraint citing the codebase must include the file path and line number, verified via `bash read`/`bash grep`.
- Do not skip the state file write — subsequent commands depend on it for argument-less invocation.
- **Acceptance criteria must be test-mappable.** Per §2 rule 8 (Test mapping), `/decompose` will require each AC to pair with a named test. ACs phrased as "feature X works" or "the implementation is correct" are unmappable. Phrase ACs as user-observable outcomes (e.g., "when user clicks tag chip, the URL updates to `?tags=<id>` and the post list filters to posts containing that tag"). If you cannot phrase an AC as a user-observable outcome, the AC is too abstract — split it or restate.
- **Apply Cross-cutting principles → Read discipline.** Readable scope for `/plan`: the doctrine, codebase under `src/`, project config (`package.json`, tsconfig, vitest.config.ts), `docs/adr/*`. Forbidden: `docs/roadmap/*` (other than your output), `docs/calibration/*`, any sibling artifact of `$1` (`.dsv4`, `.k2p6`, `.claude-cold` and similarly-suffixed files). If filesystem exploration surfaces a forbidden file, treat it as absent — do not open it. The user does NOT need to paste a "READ DISCIPLINE" preamble into the topic; this rule is the forcing function.
- **Apply Cross-cutting principles → Read discipline → Granularity discipline.** Decompose at whatever step count fits this feature's native complexity. Do NOT anchor step count or decomposition shape to any precedent established by prior features (whether visible to you or not). If your native judgment is that the feature requires 3 steps, produce 3; if 12, produce 12. The §3 chunk cap is a per-chunk guard, not a per-feature target.

## Next-step recommendation (output format — required after plan written + state file initialized)

After the plan is written and `.pi/active.md` is created, append exactly this block to your output. **Bare commands only** — slug is in `.pi/active.md`; perspective tag is NOT in state and must be passed explicitly.

```markdown
---
**✅ Plan written to `docs/roadmap/$1/$1.md` — `.pi/active.md` initialized**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/critique <perspective-tag>` | **high** — Shift+Tab to set |
| Then | `/critique <other-perspective-tag>` (rotated) | **high** |
| Then | `/revise` | **high** |

**Available perspective tags** (pick two non-overlapping for passes 1 and 2; perspective rotation is the load-bearing rule per §1):

| Tag | Lens |
|---|---|
| `senior-engineer-risk` | senior engineer reviewing for risk and edge cases |
| `pm-requirements` | PM checking requirements coverage and acceptance criteria |
| `security-perf` | security / performance reviewer |

**Recommended first pass**: `senior-engineer-risk` (broadest defect surface for typical features).
```

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands wherever the state file resolves all required args. Slug is in `.pi/active.md`; perspective tag is NOT in state and must be passed explicitly. Output `/critique senior-engineer-risk` — never `/critique <slug> senior-engineer-risk` (redundant slug) and never `/critique` alone (no default perspective; would error).
