---
description: Critical post-phase assessment of step or phase output, written to calibration trace for /calibrate to consume
argument-hint: "[<slug> <step-N | phase-name>]"
---
You are running the **Assessment** phase. Operates in one of two modes:

- **Step-level** (default, post-`/verify` per-step capture): reviews a single step's output. `$2` is a numeric step number.
- **Phase-level** (post-phase capture): reviews a workflow-phase artifact across the whole feature. `$2` is a phase name (`plan`, `critique`, `revise`, `decompose`, `distill`, `calibrate`).

Set thinking level to **medium**. Light synthesis: critically review what just shipped, capture observations for `/calibrate` to consume at end-of-feature.

## Step 0 — Argument resolution and mode detection

Per Cross-cutting principles → State management: explicit args always override the state file.

**Slug resolution** (`$1`):
- If `$1` is empty: read slug from `.pi/active.md`. If file missing or no header, error: *"No active feature. Run `/plan <slug> <topic>` first, or pass `<slug>` explicitly."*
- For phase-level invocation against `calibrate` (which runs after `/distill` deleted `.pi/active.md`): if `$1` is empty AND `.pi/active.md` is missing, error: *"`.pi/active.md` deleted by /distill — pass `<slug>` explicitly for phase-level /assess on calibrate."*

**Mode detection** (`$2`):
- If `$2` is empty: default to step-level mode. Read most recently completed step from `.pi/active.md`'s `## Step status` (last `[x]`-marked step, NOT current_step which has already advanced past it).
- Else if `$2` matches `^[0-9]+$` or `^step-[0-9]+$`: **step-level mode**. Strip `step-` prefix if present.
- Else if `$2` ∈ {`plan`, `critique`, `revise`, `decompose`, `distill`, `calibrate`}: **phase-level mode**.
- Else: error *"Unrecognized $2 — expected numeric step or phase name (plan|critique|revise|decompose|distill|calibrate)."*

Once mode is determined, follow only the matching mode's sub-sections below. Findings classification and Hard rules apply to both modes.

---

## Step-level mode

### Read context

1. The step doc: `bash ls docs/roadmap/$1/step-$2-*.md` (one match)
2. The implementation diff. Strategy: try `bash git diff -- <files-touched-from-step-doc>` first (working tree vs HEAD — captures uncommitted verified changes, the default state per the workflow's manual-commit discipline). If that returns empty, the step has already been committed; fall back to `bash git diff HEAD~1 -- <files-touched-from-step-doc>`. If still empty, use `git diff main -- ...`. Per Calibration notes → harness friction, `git diff HEAD~1` is unreliable as primary because verified changes default to uncommitted on this workflow.
3. The most recent `/verify` output (terminal scrollback or wherever you can find it; if not available, note the gap)

### Anti-flattery questions (step-level)

The same model that produced the work is now reviewing it. Without explicit prompting it'll mostly say "looks good." Each question below requires either a substantive answer or quoted evidence that you actually looked.

#### 1. Implementation drift
**Q**: What did `/implement` produce that wasn't explicit in the step doc?
- Even minor additions (a helper variable, a logger call, a defensive check the step doc didn't mandate, a comment that wasn't there).
- If "nothing", quote a specific section of the step doc to demonstrate you compared line-by-line, not vibe-checked.

#### 2. Verify-gate evidence quality
**Q**: Did any `/verify` gate pass without quoted file:line evidence?
- Mechanical checks have command output (binary pass/fail) — these are inherently evidenced.
- Constraint gate, invariant gate, acceptance criteria require text quotes per Cross-cutting principles → Evidence over claims.
- Identify any row that read like "Yes" or "verified" without a citation.

#### 3. Phase budget observation
**Q**: What in this step took longer or shorter than expected, and why?
- Look at tool-call counts, file sizes, time-to-pass.
- Compare to prior steps if any.
- Calibration data on whether the doctrine's chunk caps and thinking levels match real workload.

#### 4. Cross-step pattern recognition
**Q**: What pattern did this step share with prior steps that wasn't anticipated?
- Positive pattern recognition — tracks whether the workflow is reproducing known-good behavior across step complexity.
- Or: did this step deviate from a prior pattern? Note both directions.

#### 5. Surprise
**Q**: What surprised you about this step (positive or negative)?
- Catches things outside the explicit questions above.
- Includes harness friction, model-behavior oddities, anything you noticed that the questions don't cover.

#### 6. Test quality
**Q**: For each test added in this step (if any), would it FAIL against a deliberately-broken permutation of the implementation it claims to verify?
- Specifically: for AND-filter tests, flip to OR semantics in the implementation and rerun — the test must fail. For selector-output tests, mutate the selector to return the wrong shape — the test must fail. For component-rendering tests, return `null` from the component — the test must fail.
- Per Cross-cutting principles → Testing discipline → Negative-permutation discipline. Tests that pass against both correct AND broken implementations are rubber-stamps and surface zero signal at scale.
- Also note: did any test use `findByText` substring-match where `findByRole` would be more specific? Did any test assert internal state (Redux shape, hook internals) instead of user-observable outcome?
- If no tests were added in this step, write "No tests added — step is N/A" and quote the step doc's `## Test mapping` section to verify the AC-to-existing-test linkage held (per §2 rule 8).

---

## Phase-level mode

### Read context

Read the phase's primary artifact(s) per `$2`:

| Phase | Primary artifact(s) |
|---|---|
| `plan` | `docs/roadmap/$1/$1.md` (latest revised version) |
| `critique` | All files matching `docs/roadmap/$1/$1.critique-*.md` |
| `revise` | `docs/roadmap/$1/$1.md` (revised version) + the response-to-critiques table at the bottom |
| `decompose` | All files matching `docs/roadmap/$1/step-*.md` |
| `distill` | The ADR matched via roadmap's `produced-adrs:` frontmatter, found at `docs/adr/NNNN-*.md` |
| `calibrate` | Most recent calibration output: `bash ls -t docs/calibration/[0-9]*-$1.md \| head -1` |

Also read for context:
- `~/.pi/agent/coding-agent-workflow.md` — the doctrine claim for this phase. Relevant section per phase: §1 for plan/critique/revise, §2 for decompose, §7 for distill, "Calibration buckets" + Calibration notes for calibrate.
- `docs/calibration/trace-$1.md` — for cross-phase pattern recognition (does this phase share patterns with prior phases?).

### Anti-flattery questions (phase-level)

The same model that produced the phase output is now reviewing it. Each question requires substantive answer or quoted evidence.

#### 1. Phase output substance
**Q**: What did the phase produce that wasn't derivable from input artifacts?
- The phase reads inputs (e.g., `/calibrate` reads roadmap + critiques + step docs + ADR + trace + concerns) and produces an output. What in the output is *new analysis* vs *restatement of input*?
- If "everything is novel synthesis", quote 2-3 specific output passages and identify the input passages they derive from. Demonstrate the synthesis happened.
- If the output reads like a summarization of inputs, that's a phase-broken signal — flag it explicitly.

#### 2. Evidence per claim/recommendation
**Q**: Did the phase's output cite evidence per claim or recommendation?
- For phases that produce structured output (`/critique` scoring, `/decompose` constraints, `/calibrate` edits): each scored item / constraint / edit must cite evidence — file:line, command output, or quoted text.
- Identify any item that read like a claim without a citation. Per Cross-cutting principles → Evidence over claims, this is the load-bearing requirement.

#### 3. Phase budget
**Q**: How long did the phase take? Was the thinking level appropriate?
- Tool-call counts, time-to-completion, output length.
- Compare to the per-phase thinking-level guidance in Cross-cutting principles → Thinking levels per phase. Note any phase where the default felt under- or over-powered.

#### 4. Cross-phase pattern
**Q**: What pattern did this phase share with prior phases (per phase, not per step)?
- Did the phase reproduce a known-good pattern (perspective rotation produces non-overlapping findings, mechanical-first verification fail-closed)?
- Did the phase exhibit a known failure mode (rubber-stamp consistency sweep, accumulation bias in calibration recommendations, silent merging of step-list shape)?
- Note both directions — positive validation and negative regression.

#### 5. Surprise
**Q**: What surprised you about this phase output (positive or negative)?
- Things outside the explicit questions above. Harness friction, model-behavior oddities, anything noticed that the questions don't cover.

---

## Findings classification (both modes)

For each finding produced above, classify into one of three buckets:

| Bucket | Definition | Examples |
|---|---|---|
| **portable** | Applies regardless of model/harness | Evidence-over-claims, perspective rotation produces non-overlapping findings, response-table-prevents-silent-drops |
| **model-specific** | Tied to current model behavior; changes when model changes | Chunk cap of 15-20, thinking-level guidance, K2.6's deferred-decision tendency, K2.6's silent-improvement-mid-write tendency |
| **harness-specific** | Tied to pi version or workflow tooling; changes on pi upgrade | pi 0.70.2 GLM `enable_thinking` bug, `/reload` cache, prompt template format, gawk `\b` interpretation |

If unsure, mark with `[?]` and let `/calibrate` classify at end-of-feature.

---

## Output — append to calibration trace

Write to `docs/calibration/trace-$1.md`. If file doesn't exist, create it with header `# Calibration trace: $1`. F-numbering is continuous across the file — start from `(last F-number in file) + 1`.

### Step-level entry format

```markdown
## Step $2 — <step-slug-from-filename>

**Verified**: <YYYY-MM-DD HH:MM>
**Phases run**: /preflight (mode), /implement, /verify
**Verdict**: PASS | FAIL

### Anti-flattery scan

1. **Implementation drift**: <substantive answer or quoted comparison evidence>
2. **Verify-gate evidence**: <which gates had citations, which read like vibes>
3. **Phase budget**: <observation about effort/time>
4. **Cross-step pattern**: <observation>
5. **Surprise**: <observation>
6. **Test quality**: <permutation results, selector preference compliance, observed-state-vs-internal-state assertions; or "No tests added — N/A" with cited test-mapping linkage>

### Findings (classified)

- F<N> [portable | model-specific | harness-specific | ?]: <one-line finding>
- F<N> [...]: <...>
```

### Phase-level entry format

```markdown
## Phase: $2 — <phase-name with brief context, e.g. "calibrate — v1 against ADR-0001">

**Reviewed**: <YYYY-MM-DD HH:MM>
**Artifact(s) read**: <file paths>
**Verdict (one-line)**: <did the phase earn its weight on this feature?>

### Anti-flattery scan (phase-level)

1. **Phase output substance**: <what was novel synthesis vs restatement; quote 2-3 passages with derivation>
2. **Evidence per claim**: <which items had citations, which read like vibes>
3. **Phase budget**: <effort/time/thinking-level fit>
4. **Cross-phase pattern**: <known-good reproduction or known failure-mode regression; note both>
5. **Surprise**: <observation>

### Findings (classified)

- F<N> [portable | model-specific | harness-specific | ?]: <one-line finding>
- F<N> [...]: <...>
```

Do not auto-update `.pi/active.md` — that's `/verify`'s job (step-level only) and was already done. Do not summarize the whole feature — just this step or phase.

`/calibrate` reads this trace at end-of-feature; don't pre-summarize or pre-bucket conclusions across steps/phases. Per-step and per-phase granularity is the value.

---

## Hard rules

- Per Cross-cutting principles → Evidence over claims: "no findings" answers require quoted evidence that you actually looked.
- Don't be diplomatic. The point of `/assess` is to surface things that would otherwise be lost in the smooth flow of a clean run.
- Findings classified as `[?]` are fine — `/calibrate` will resolve. Don't guess wrong to look decisive.

## Next-step recommendation (output format — required after trace entry written)

After the trace entry is appended successfully, output exactly this block. **Bare commands only** — slug and step persist in `.pi/active.md`; do NOT include redundant args.

### Step-level mode

If `Current step` is a number ≤ `Total steps` (mid-feature, more steps remain):

```markdown
---
**✅ Trace entry appended for step $2 — <N> findings classified**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/preflight` | **minimal** — Shift+Tab down from medium |
| Then | `/implement` | **medium** |
| Then | `/verify` | **minimal** |
| Then | `/assess` | **medium** (calibration mode) |
```

If `$2` was the final step (`Current step` now `complete`):

```markdown
---
**✅ Trace entry appended for step $2 (final step) — <N> findings classified**

| Phase | Command | Thinking |
|---|---|---|
| **Now** | manual smoke test in browser | n/a |
| Then | `/distill` | **high** |
| Then | `/calibrate` | **high** |
```

### Phase-level mode

```markdown
---
**✅ Phase-level trace entry appended for `$2` — <N> findings classified**

No automatic next phase — phase-level /assess feeds /calibrate synthesis at end-of-feature. The user decides next action based on findings (e.g., extend a template, retract a recommendation, propose a doctrine candidate).
```

**Hard rule — command form**: per `/status` template's "Hard rules — command form" section, recommend BARE commands wherever the state file resolves all required args. Output like `/preflight routing-skeleton 4` is wrong — slug and step are in `.pi/active.md`. Output `/preflight` (thinking: **minimal**) instead.
