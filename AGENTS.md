# Coding Agent Workflow — Kimi K2.6 primary, perspective-switching critique (cross-model when available)

Tuned for self-host trajectory; current bridge is Fireworks-hosted weights.
Core principle: **model is a fast executor over deltas; file system carries state.**

**K2.6 (thinking mode)** ties or beats Opus 4.6 on most agentic/coding benchmarks (SWE-Bench Pro +5, Terminal-Bench, LiveCodeBench, HLE-w/-tools, DeepSearchQA) and ties or beats GLM 5.1 on every directly-comparable benchmark including the reasoning-heavy ones (HMMT +10, GPQA +4, Toolathlon +9). Per Fireworks's deployment, thinking mode is on by default — verified by curl. The primary signal driver in critique is **perspective rotation** (same model, different role: senior eng → PM → security/perf) — validated on dark-mode-toggle and on prior Opus-era runs as producing near-zero overlap between passes. **Cross-model critique** is high-value when the critic model is reachable; on initial pi+Fireworks data it underperformed perspective rotation, and the harness reliably driving a second model is itself a constraint (see Calibration notes for the pi/GLM `enable_thinking` bug). Use **DeepSeek V4 Pro Think High** as the primary cross-model critic (validated on feature 4 — promote rule met with 3 unique catches per pass not produced by perspective rotation alone). Fall back to **MiniMax M2.7** if DSV4 unreachable, then same-model perspective rotation. **GLM 5.1 demoted** — pi 0.70.2 `enable_thinking` bug + now superseded by DSV4 on quality.

The workflow's chunk-size cap, hallucination defense, and verification gates were originally calibrated against assumed instruct-mode behavior — they may be over-engineered for K2.6 thinking mode. Treat current numbers as starting points to verify against real workload, not measured truth.

---

## Model assignment

| Phase | Model | Why |
|---|---|---|
| Planning / step decomposition | **K2.6 (thinking)** | Wins or ties GLM 5.1 on all directly-comparable reasoning + coding benchmarks; thinking mode on by default |
| Implementation | **K2.6 (thinking)** | Same model as planner — no plan-to-impl translation friction; one model to fine-tune later (LoRA) |
| Critique — perspective rotation (primary) | **K2.6 (thinking)** with explicit role swap (senior eng / PM / security-perf) | Same model + different role flips attention; produces non-overlapping findings across passes; doesn't depend on critic-model availability or harness compat |
| Critique — cross-model | **DSV4 Pro Think High** preferred; **MiniMax M2.7** fallback | Different training data + different failure modes. Validated on feature 4 (posts-tags-filter): DSV4 cross-model produced 3 unique catches K2.6 perspective rotation likely misses (junction-key delimiter ambiguity, OR-clause in focus management, RouteErrorBoundary technical inaccuracy). Doctrine's promote rule met. Re-evaluate at n=2-3 cross-feature. |

---

## Cross-cutting principles

These apply to every phase. Phase contracts (§1–§7) lean on them as bedrock.

### Evidence over claims

Every assertion of completeness, consistency, or correctness in any workflow phase must cite evidence — file paths with line numbers, quoted text, command output, or grep results. Phases that produce structured assessments must produce structured evidence per assessment.

- `/critique` scoring: each ✓/⚠/❌ must reference the line(s) it scored.
- `/decompose` consistency sweep: "no contradictions found" requires quoting one Context/Decision/Implementation triplet per step as proof the check happened.
- `/verify`: invariant claims require the file/line that proves preservation, not just "verified".
- `/distill`: ADR's Validation section signals must be objectively measurable.

"The model said it was fine" is not evidence. "Quoted line N of file F" is.

### Thinking levels per phase

Pi's thinking-level setting (`Shift+Tab` cycles, or `/settings`) materially affects synthesis quality. Defaults below; deviate when the task signals otherwise.

| Phase | Default | Why |
|---|---|---|
| `/plan` initial | medium | Structured output from requirements; decomposition-shaped |
| `/plan` revision against critiques (`/revise`) | **high** | Synthesis-heavy: track 15+ critique points without silent drops, write Decision N additions, maintain self-consistency across the revised corpus |
| `/critique` | **high** | Synthesis-heavy: 6-field scoring + discipline-axis check across all decisions |
| `/decompose` | medium | Mechanical translation from plan to step docs, plus consistency sweep |
| `/preflight` | minimal | Mechanical: list files, grep, check imports |
| `/implement` | medium | Mostly transcription from step doc; thinking helps but synthesis demand is low |
| `/verify` | minimal | Mechanical check against acceptance criteria |
| `/distill` | **high** | Synthesis: rewrite roadmap reasoning into prose, strip scaffolding, supersession analysis |
| `/calibrate` | **high** | Synthesis across full corpus: roadmap, critiques, step docs, ADR, run telemetry |

The cost asymmetry favors high on synthesis tasks: a few cents of thinking tokens vs shipping a plan with silent drops or an ADR that reads like a roadmap copy.

### Pre-flight (before any workflow loop)

Before invoking `/plan` on a real feature, run a smoke `/critique` on a throwaway 10-line doc with the intended critic model. If it errors (e.g., the pi 0.70.2 GLM `enable_thinking` bug noted in Calibration notes), switch to the fallback critic before starting the real loop. Don't burn `/plan` effort discovering a harness incompatibility that would have surfaced in 60 seconds of pre-flight.

### Mechanical verification for defensive-code constraints

Model-based verification fails on constraints with OR clauses ("guard internally **or** in callers"), escape hatches ("unless X is satisfied"), or compositional requirements ("X applies if Y"). The model finds the loophole and produces plausible evidence pointing at one branch without verifying the full conditions. Both `/implement` self-check AND `/verify` external check fall to this pattern.

Validated on dark-mode-toggle Step 2: the `/decompose` pass produced an implementation sketch with no `try/catch` while the constraint required it (with an "or in callers" alternative). `/critique`, `/decompose`'s consistency sweep, `/implement`'s constraint contract, and `/verify`'s constraint gate all rubber-stamped at the model layer. The OR clause was the loophole; every layer found it and reasoned compliance from it without checking that all callers actually guard.

For **defensive-code constraints** (try/catch presence, null checks, type guards, specific import patterns, defensive default values, error fallbacks, listener cleanup, etc.), the doctrine baseline requires:

1. **Constraint phrasing without OR clauses.** Prefer "X is required in function F" over "X or Y is required." If alternatives are genuinely allowed, list each as a separate constraint with its own mechanical check.
2. **Mechanical verification commands.** Step docs include a `## Mechanical constraint checks` section listing bash one-liners (grep, awk, ripgrep, AST tools) that exit 0 if the constraint is satisfied in the produced code, non-zero otherwise.
3. **Sketch self-check.** The implementation sketch in the step doc must itself satisfy every mechanical check. `/decompose`'s consistency sweep validates this — the sketch IS code, and it must pass the same checks the implementation will face.
4. **Mechanical-first verification.** `/verify` runs all mechanical checks BEFORE any model-based assessment. Model gates (constraint, invariant, acceptance criteria) only run if all mechanical checks pass. A failed mechanical check fails the entire verification, regardless of what the model gates would say.
5. **Check self-validation.** Every mechanical check command must be validated before being shipped in a step doc. Validation procedure: (a) run the check against the implementation sketch — must exit 0 (sketch satisfies the constraint by design); (b) run the check against a deliberately-broken permutation of the sketch (e.g., remove the try/catch the check is verifying) — must exit non-zero. If either case fails, the check is broken and must be fixed before the step doc ships. Hand-written and model-generated checks both require this validation. Validated on dark-mode-toggle Step 2 retrofit: an `awk '/.../\b...'` pattern was silently broken because gawk interprets `\b` as a literal backspace (0x08), not a word boundary — the false FAILs would have blocked correct implementations indefinitely. False-fails block correct code; false-passes rubber-stamp violations; both undermine the entire mechanical-verification baseline.

Mechanical checks complement (do not replace) model-based scoring. The model still judges *whether the implementation is correct in spirit*; the script judges *whether specific patterns are present in the bytes*. The combination catches what either alone misses, and the script is the binding answer when they disagree.

### State management — explicit args, with state-file fallback

The pi workflow takes explicit `<slug>` and (where applicable) `<step-N>` arguments. To reduce per-invocation typing, a status file at `.pi/active.md` records the active feature; when args are omitted, commands fall back to the file. **Explicit args always override.** Empty args + no file = clear error, never silent fallback.

**File format** (created by `/plan`, updated by `/decompose` and `/verify`, deleted by `/distill`):

```markdown
# Active feature: <slug>

- **Started**: YYYY-MM-DD
- **Branch**: feature/<slug>
- **Worktree**: <absolute path>
- **Status**: in_progress | complete
- **Current step**: <N or null>
- **Total steps**: <N or null>
- **Phases completed**: plan, critique-..., revise, decompose

## Step status
- [x] step-1-<slug> — verified YYYY-MM-DD HH:MM
- [ ] step-2-<slug>
- [ ] step-3-<slug>
```

**Resolution procedure** (every template applies this when args missing):

1. If `$1` (slug) is empty: `bash test -f .pi/active.md` — file must exist. Extract slug via `bash sed -n 's/^# Active feature: //p' .pi/active.md`. Use as `$1`. If file missing or no header line: stop with error *"No active feature. Run `/plan <slug> <topic>` first, or pass `<slug>` explicitly."*
2. If `$2` (step number) is empty (only for step-aware commands — `/preflight`, `/implement`, `/verify`): extract step via `bash sed -n 's/^- \*\*Current step\*\*: //p' .pi/active.md`. If "null" or empty: default to 1 (first step). Use as `$2`.
3. Explicit args bypass this entirely.

**Auto-advance**: `/verify` on PASS marks step N's checkbox `[x]` with timestamp and increments `Current step:` in `.pi/active.md`. `/decompose` populates the initial step status list. `/distill` appends `distill` to the `Phases completed:` line; it does NOT delete active.md (state describes feature-in-flight; the file persists through /calibrate). **`/finalize` deletes `.pi/active.md` as part of worktree removal** after the merge to main is complete (per §8). Through the full lifecycle — /plan creates active.md → /verify auto-advances → /decompose populates step list → /distill marks distill complete → /finalize ships and cleans up — the state file describes feature-in-flight state. No archive needed: git history + ADR + roadmap captures everything durable.

**State file protection**: `.pi/active.md` is the workflow's own state file and must be excluded from any repository-wide strip, delete, or `git clean` operations. If `.pi/` is in the removal list (e.g., because it is agent personal configuration), the state file must be backed up before `.pi/` deletion and restored immediately after. Alternatively, store `.pi/active.md` outside the repository directory.

**Backward compatibility**: explicit-arg invocations work unchanged. The state file is purely a default; correctness doesn't depend on it.

**Per-worktree state (parallel feature support)**: per §8 (Post-ship git operations), `.pi/active.md` is per-worktree. `/plan` creates a feature branch + sibling worktree at `<repo-parent>/<repo-name>-<slug>`; each worktree has its own state file. Multiple pi sessions in different worktrees do not see each other's state, enabling parallel in-flight features without state collision. This is the harness-side enablement layer for 5+ concurrent features.

### Template tail format

Every prompt template (`~/.pi/agent/prompts/*.md`) that recommends a next-step command must use the canonical Phase/Command/Thinking table format with a referenced Hard Rule on command form. Prose-embedded "After this: /command (thinking: level)" tails fail empirically — the model produces verbose-with-redundant-args output instead of bare commands.

**Canonical format**:

```
| Phase | Command | Thinking |
|---|---|---|
| **Now** | `<bare-command>` | **<level>** — Shift+Tab to set if needed |
| Then | `<bare-command>` | **<level>** |
```

**Hard rule**: bare commands wherever the state file resolves all required args (slug, step). Explicit args only when the state cannot resolve them (perspective tag for `/critique`, ADR number for `/calibrate`). Each template's tail must include a reference to `/status` template's "Hard rules — command form" table as the canonical command-form mapping.

Validated empirically: 10 prompt templates within feature 3 (rtk-normalized-posts) tightened from prose tails → table format. Prose-embedded next-step prescriptions produced ~50% miss rate on bare-command discipline (cluster F1 + F7 + F11 across feature 2). Table format with explicit Hard Rule reference produced ~100% bare-command output on first invocation across both K2.6 thinking and DSV4-Pro Think High (cross-model validated). Bucket: portable.

### Calibration buckets — portable / model-specific / harness-specific

Every claim in this doctrine belongs to one of three buckets. `/calibrate` classifies findings into these buckets so doctrine updates land correctly when models or harnesses change. Auto-pilot doctrine application (not yet implemented; tracked as future `/apply` phase) requires this classification as a prerequisite.

| Bucket | Definition | Survives across | Examples in current doctrine |
|---|---|---|---|
| **portable** | Applies regardless of model or harness; baseline mechanics of the workflow | Model swaps (K2.6 → K2.7 → self-host) AND harness swaps (pi → other) | Evidence-over-claims; perspective rotation; response-table revision; mechanical verification with self-validation; cross-step consistency contract; §1 Discipline-axis check; §7 Distillation contract; State management protocol |
| **model-specific** | Tied to current model behavior; revisited when the model changes | Harness swaps (pi version changes) but NOT model swaps | Thinking levels per phase (specific levels); §3 chunk cap of 15-20; §6 hallucination defense necessity; K2.6's silent-improvement-mid-write tendency; K2.6's OR-clause exploitation pattern; cost-driven configuration values |
| **harness-specific** | Tied to pi version or workflow tooling; revisited when pi changes | Model swaps but NOT harness swaps | pi 0.70.2 GLM `enable_thinking` bug; `/reload` prompt-cache friction; gawk `\b` interpretation; prompt template format conventions |

When `/calibrate` produces edit recommendations, it groups them by bucket so each can be reviewed against the right re-validation cadence:

- **Portable edits** carry the most risk (affect every future feature). Confidence bar should be highest before applying.
- **Model-specific edits** are expected to churn more frequently. Lower confidence bar acceptable; revalidate when model changes.
- **Harness-specific edits** are localized to known external bugs/quirks. Cheapest to apply, safest to retract when the underlying issue is fixed upstream.

When a doctrine edit is added, tag it with its bucket inline (or in a per-section comment). When in doubt, default to portable and let `/calibrate` surface it for re-classification on a future run.

What is NOT a defensive-code constraint (and so does not require mechanical check): architectural decisions ("must be usable from any component without prop drilling"), UX requirements ("UI must expose three states"), naming conventions, design rationale, performance budgets without a numeric threshold. These remain model-judged.

### Read discipline (synthesis phases)

For synthesis phases (`/plan`, `/critique`, `/revise`, `/distill`, `/calibrate`, `/promote`), the readable input scope materially affects output quality and architectural choices. Models anchor to corpus context when allowed; restricting the readable scope changes downstream choices.

**Validated on feature 4 (posts-tags-filter)**: Both K2.6 and DSV4 changed junction-structure choice (between `tagIds: string[]` on Post entity vs separate junction slice) when read discipline blocked access to prior feature artifacts. Step counts also moved (K2.6 7→5, DSV4 9→6 under read discipline). Architectural drift correlates with corpus visibility, not with model capability alone.

**Default readable scope per phase** (the prompt template re-states this; the harness should enforce):

| Phase | Readable | Forbidden |
|---|---|---|
| `/plan` | doctrine, codebase under `src/`, `package.json` + tsconfig + vitest config, `docs/adr/*` | `docs/roadmap/*`, `docs/calibration/*`, sibling artifacts of the slug being planned (`.dsv4`, `.k2p6`, `.claude-cold` suffixed) |
| `/critique` | the plan being critiqued, doctrine, codebase, ADRs | other slugs' critique files, prior feature roadmaps, calibration corpus |
| `/decompose` | the plan, doctrine, codebase, ADRs | prior feature step docs (don't anchor granularity) |
| `/distill` | the roadmap + step docs for the slug, doctrine, prior ADRs | other features' calibration files |
| `/calibrate` | the trace + findings-pending + roadmap + critiques + step docs + ADR for the slug; doctrine; optionally prior calibrations for cross-feature reference | sibling calibration outputs of the same feature being calibrated (those are A/B benchmark inputs, not corpus) |

**Granularity discipline (subset for `/plan`)**: decompose at whatever step count fits the feature's actual complexity natively. Do NOT anchor step count, granularity, or shape to any precedent established by prior features (whether visible to the agent or not). The §3 chunk cap is a per-chunk guard, not a per-feature target.

When a synthesis phase is invoked, the prompt template MUST list its readable scope explicitly so the model has no ambiguity. Filesystem exploration that surfaces forbidden files MUST treat them as absent.

Bucket: portable. Confidence: high — mechanically forced (restricting filesystem reads is deterministic) plus n=2 cross-model evidence (both K2.6 and DSV4 flipped architectural choices under read discipline on feature 4).

### Testing discipline

For features that ship runtime code, every acceptance criterion produced by `/decompose` must map to at least one named test. The mapping is enforced at decompose time (§2 rule 8) and at `/verify` time (gate 5).

**Test type assignment**:
- **Unit tests** — pure logic, selectors, reducers, utility functions. Co-located with source (e.g., `src/store/postsSlice.test.ts`). No DOM, no providers.
- **Integration tests** — component rendering with full provider tree. Live in `src/test/`. Test user-observable behavior, not internal state.
- **E2E tests** — deferred until manual smoke test surface across features exceeds ~30 minutes per ship. Trigger condition: revisit at n=5 features shipped under current discipline.

**Mock state construction**: use entity-adapter API only (`postsAdapter.setAll(postsAdapter.getInitialState(), data)`). Hand-rolling `{ ids, entities }` is forbidden — adapter shape changes break hand-rolled mocks silently. Established in ADR 0003; promoted here as portable doctrine.

**Selector preference (Testing Library)**: prefer `findByRole({ name: ... })` over `findByText(...)`. Substring-match semantics of `findByText` produced a sketch defect on rtk-normalized-posts Step 6 (`findByText(/First Post/i)` matched both heading AND body text). Role-based queries are unique by element type.

**Brittleness avoidance**:
- No snapshot tests. Low signal, high churn.
- No timing-dependent assertions without justification (`waitFor` with arbitrary timeouts, real `setTimeout`).
- No tests that assert internal Redux state shape directly — assert user-observable outcome (rendered DOM, dispatched action return value, selector return value against fixture state).

**Negative-permutation discipline** (extends Mechanical verification rule 5 to test specs): for each test that asserts code behavior, verify the test would FAIL against a deliberately-broken permutation of the implementation (e.g., flip AND to OR in filter — the AND-semantics test must fail under the permutation). Tests that pass against both correct AND broken implementations are rubber-stamps; rebuild them.

Bucket: portable. Confidence: medium — n=1 within feature 4 PM critique R-PM-10 (full user-journey integration test missing) plus n=1 within feature 3 sketch defect (`findByText(/First Post/i)` substring match). Promote to high after n=2 cross-feature confirmation and no brittleness regressions across next 5 features.

---

## 1. Planning phases (1-3 roadmap mds)

- **Pass 1**: generate plan (K2.6 thinking mode)
- **Pass 2**: critique from explicit perspective. Default: K2.6 with role swap. When cross-model is reachable: GLM 5.1 (or MiniMax M2.7 fallback). Pick one perspective per pass:
  - "senior engineer reviewing for risk and edge cases"
  - "PM checking requirements coverage and acceptance criteria"
  - "security/perf reviewer"
- **Pass 3** (required when any critique surfaced incorporable findings): revise against accumulated critiques. Output must include a per-point response table at the bottom of the revised plan with one row per critique point, marked **incorporated** (with cited section), **rejected** (with one-line reason), or **partial** (kept X, dropped Y because Z). No silent drops. Run via `/revise <slug>` — auto-discovers all `docs/roadmap/<slug>.critique-*.md` files.

**Total iteration cap: 5.** Critique-pass sweet spot: **2; 3 if the plan covers cross-cutting infra changes or unfamiliar domains**. Beyond that, the critic fabricates problems to satisfy the prompt; noise floor rises faster than signal.

**Switching perspective every pass is the load-bearing rule.** Same lens repeated produces variations of the same critique; rotating senior-eng → PM → security/perf produces near-zero overlap between passes (validated on dark-mode-toggle), even on the same model. This is the single highest-leverage piece of `/critique`.

**Cross-model critique is high-value when available** — different training data and different failure modes catch a different slice of blind spots than perspective rotation alone. Validated on feature 4 (posts-tags-filter, n=1): DSV4 Pro Think High as cross-model critic produced 3 unique catches not surfaced by K2.6 perspective rotation alone (junction-key delimiter ambiguity, OR-clause in focus-management constraint, RouteErrorBoundary technical inaccuracy). Doctrine's promote rule (≥1 unique catch per pass) met. **Treat cross-model as recommended (not optional polish)**, pending n=2-3 cross-feature confirmation. Primary: DSV4 Pro Think High. Fallback: MiniMax M2.7 (when DSV4 unreachable). GLM 5.1 demoted due to pi 0.70.2 `enable_thinking` bug AND now superseded by DSV4 on quality.

### Discipline-axis check (required on every critique pass)

Six-field scoring catches gaps in stated decisions. It systematically misses gaps in *how decisions were made*. Every critique pass must explicitly scan for these four discipline-axis failures regardless of perspective applied. Each surfaced finding is a critique gap.

1. **Deferred decisions** — any "either X or Y works" / "X is fine; Y is also fine" / "could be A or B" construction in the artifact, anywhere. **Pattern recognition is not pattern elimination**: the model resolves named instances and reproduces unnamed ones one level down. Scan the entire artifact, not just sections previously flagged. (Validated on dark-mode-toggle: Step 5 killed the Decision-4 StrictMode hand-wave but introduced a new "either nesting works" construction in failure modes.)
2. **Unjustified specifics** — concrete choices stated without defense (a particular storage key namespace, an arbitrary timeout, a specific library version pinned without reason). The choice may be correct; absent justification, future readers can't tell.
3. **Circular rejection reasoning** — alternatives rejected because of a constraint the planner set themselves (e.g., "rejected because feature X is out of scope" when the planner set scope). The rejection survives only if the constraint is independent of the rejection.
4. **Missing decisions on cross-cutting concerns** — the plan covers feature decisions but elides decisions about: testing strategy, accessibility, error handling, **instrumentation / observability (logging, metrics, tracing),** deployment, security, **formal performance budgets (numeric thresholds, not hand-waved "negligible")**. A missing decision is not a non-decision; it's an implicit decision someone else will make later.

If any axis surfaces a hit, name it explicitly with a quoted line number and flag for the revision pass to resolve. Per Cross-cutting principles → Evidence over claims, citations are required.

### Reasoning template (apply to every planning prompt)

The template is the **harness contract**, not a model crutch. K2.6 thinking mode and GLM 5.1 both handle most of these moves implicitly — but enforcing the structure as required output gives you (a) determinism for cross-model regression testing, (b) protection against the residual gap on adversarial edge cases, (c) a portable harness that works the same when you swap models or migrate to self-host. Prepend this to all planning and decomposition prompts (regardless of which model produces or critiques):

> For each major decision in the plan, output:
>
> 1. **Decision**: what is being decided
> 2. **Constraints**: what must hold true; cite specific files/code where possible
> 3. **Second-order effects**: what else changes downstream; which other steps does this affect; what new constraints does it create
> 4. **Failure modes**: what could go wrong. **Name the deceptively obvious wrong answer and why it's wrong** — pause before the obvious choice
> 5. **Alternatives considered**: at least 2; explicit reason for rejecting each
> 6. **Self-consistency check**: does this contradict any earlier decision in this plan? Quote the prior decision before answering. If contradictions exist, resolve them here — don't defer. If all decisions return "no contradictions," require a one-sentence explicit statement that the decisions are genuinely orthogonal (e.g., "CSS strategy, storage strategy, and API shape are independent dimensions"). Do not force a synthetic "resolved tension" when none exists, but do not accept a bare "no contradictions" without the orthogonality sentence.

Apply at the **phase level** during high-level planning AND at the **step level** during decomposition. The structure makes implicit reasoning explicit — readable by you, diffable across models, regression-testable when you swap weights.

Critique passes use the same six fields as a review checklist — each item the planner produced gets scored against each axis. If any field is empty or shallow, that's the gap.

---

## 2. Step decomposition (md → steps)

Each step doc must be **self-contained**: someone reading only that doc (no conversation context) can implement the step. Required per step:

- Acceptance criteria
- Files touched
- Concrete API/function signatures (verify they exist in the codebase, don't assume)
- Verification commands (tests, typecheck, lint)

K2.6 in instruct mode fills implicit intent with confident guessing; thinking mode mostly self-corrects, but the doc-level discipline costs nothing and guards against mode regressions. Make intent explicit at the doc level so the model has nothing to guess regardless of mode.

### Cross-step consistency contract

After producing all step docs, run a consistency sweep before claiming decomposition is done. The model writes self-contained step docs reliably; the failure mode is not checking the corpus against itself or against the roadmap.

1. **Roadmap-to-step traceability**: every requirement, decision, and constraint in the roadmap must appear in at least one step's acceptance criteria, implementation, or invariants. Unmapped roadmap items are silent drops — the highest-priority failure mode this catches.
2. **Internal step consistency, with evidence**: each step doc must not contradict itself between Context, Decision, Constraints, and Implementation sections. Per Cross-cutting principles → Evidence over claims, the sweep must quote one Context/Decision/Implementation triplet per step as proof the check happened, even when no contradiction is found. "No contradictions found" with no quoted text is rubber-stamping; reject the sweep and re-run.
3. **Generalized deferred-decision scan**: scan the entire decomposition (every step doc, every section) for ANY "either X or Y works" / "X is fine; Y is also fine" / "could be A or B" construction. Each must be resolved with a one-sentence reason or marked as a genuine equivalence (rare). Pattern recognition is not pattern elimination: resolving the previously-flagged instance does not prevent a new one from appearing one level down. Same axis as §1 Discipline-axis check.
4. **Step-list-shape match**: the produced step list must match the roadmap's step-level outline, OR the consistency sweep must explicitly call out and justify each merge/split. Silent restructuring (e.g., merging a Verification step into the prior step) hides the change from anyone reading the roadmap as the source of truth. **Compare roadmap step count (N) against produced step-doc count (M). If M ≠ N, each merge or split must be explicitly named and justified with a one-sentence reason (e.g., "Step 7 verification merged into Step 6 because both are tested by the same UI component"). "No contradictions found" is insufficient.**
5. **Scope changes require roadmap edits**: if decomposition reveals a roadmap requirement is too large or impractical, edit the roadmap to mark it deferred or out-of-scope, then proceed. Do not write a step doc that silently scopes a requirement out.
6. **Mechanical checks for defensive-code constraints**: per Cross-cutting principles → Mechanical verification, every step doc that lists a defensive-code constraint (try/catch presence, null checks, type guards, specific import patterns, listener cleanup, etc.) must include a `## Mechanical constraint checks` section with bash one-liners that verify the constraint mechanically. **When a step doc lists zero mechanical checks AND modifies executable code, runtime APIs, or error paths, require an explicit one-line justification in the section header (e.g., "No defensive-code constraints — pure config edit with no executable code, no runtime APIs, no error paths"). Steps that modify only static configuration (CSS directives, HTML markup without script logic, JSON/YAML values) may omit both mechanical checks and their justification — the absence is implicitly trivial.** The implementation sketch in the step doc must itself pass every mechanical check — validated as part of the consistency sweep, not deferred to `/verify`. **Additionally, for any step whose implementation sketch contains executable TypeScript/JavaScript code, the consistency sweep must extract the sketch to a temporary file and run the project's typecheck (`tsc --noEmit`) and lint (`eslint`) against it. Lint/typecheck failures in the sketch are consistency-sweep failures, not `/implement` surprises.** For each mechanical check, the consistency sweep must also run it against a deliberately-broken permutation of the sketch (e.g., remove the constraint the check is verifying, or alter a grep pattern to a known false-positive variant) and confirm it exits non-zero. If the check passes against both correct and broken permutations, it is defective and must be rewritten before the step doc ships. Decompose-time validation of sketch-vs-constraint alignment is the load-bearing piece: `/critique` already missed defensive-code OR clauses on dark-mode-toggle Step 2; the consistency sweep is the last opportunity before `/implement` ships unfaithful code.
7. **Findings capture hierarchy**: `/assess` per-step trace (`docs/calibration/trace-<slug>.md`) is the canonical findings source. `findings-pending-<slug>.md` remains for discretionary ad-hoc captures during phases that don't run `/assess`. `/calibrate` deduplicates across both at synthesis time, preferring trace when conflicts arise.

8. **Test mapping** (per Cross-cutting principles → Testing discipline): every step doc that ships runtime code must include a `## Test mapping` section listing each acceptance criterion paired with the named test that verifies it. AC without a test is a consistency-sweep failure. The mapping is the binding answer when `/verify` gate 5 (acceptance criteria) runs — each AC's "evidence" cites the test name. Steps that modify only configuration, documentation, or static assets may omit the section with an explicit one-line justification (e.g., "No runtime code — test mapping N/A"). Format:
   ```
   ## Test mapping
   | AC | Test name | Test type | File |
   |---|---|---|---|
   | <AC text> | `<describe block > it block>` | unit \| integration \| e2e | `src/.../X.test.ts` |
   ```

This contract was added after the dark-mode-toggle run produced: a Step-5 internal contradiction (Context said "outside StrictMode", Decision said "inside"), a silent drop of arrow-key navigation from Step 6 (roadmap UI Spec required it), a propagated "either works" hand-wave on StrictMode placement, a silent merge of Step 7 (Verification) into Step 6, rubber-stamped "no contradictions found" claims with no quoted evidence, and a Step 2 sketch that didn't satisfy its own try/catch constraint (rule 6 added in response). All six failures share the root cause of skipping the cross-step consistency pass with rigor — and the sixth specifically required mechanical, not model-based, enforcement.

---

## 3. Implementation chunks

**Starting target: 15-20 tool calls per chunk** (down from ~30-50 on Opus). This was calibrated against assumed instruct-mode behavior and may be conservative for K2.6 thinking mode — Moonshot claims sustained performance over thousands of steps. **Empirical dark-mode-toggle data (n=1) shows K2.6 thinking mode no drift through ~40 tool calls (Step 6: 39 across preflight+implement+verify; Step 4: 50+ across two implement runs). Retain 15-20 as the default; raise to 30-40 only after n≥2 features confirm the pattern. Document observed caps per feature in Calibration notes rather than changing the default on n=1.**

**What the cap is guarding against** (if any of these show up, the cap is right; if not, raise it):

- Repeating earlier work
- "Helpfully" refactoring code it forgot it wrote earlier in the session
- Inventing imports/APIs that don't exist

**Empirical calibration**: run a real refactor end-to-end without arbitrary chunk-cap interventions and note the round count where one of the above first appears — that's your actual cap. Until then, 15-20 is a safe default; 40-60 is plausibly fine in thinking mode.

If a chunk hits the wall, decompose further. Smaller chunks > heroic chunks.

---

## 4. Cold starts

Every chunk starts in a **fresh context**. Don't carry conversation state. The file system is the memory.

Prompt template:

```
Read $STEP_MD.
Read $RELEVANT_FILES.
Diff state so far: $DIFF (or `git diff main` output).
Implement substep $N of $STEP_MD.
After each file change: run $VERIFY_CMD, report results verbatim.
Do not proceed if $VERIFY_CMD fails — stop and report.
```

---

## 5. Verification gates

Verification runs in this order (per Cross-cutting principles → Evidence over claims and → Mechanical verification):

1. **Mechanical constraint checks** — bash one-liners declared in the step doc's `## Mechanical constraint checks` section. Run each, capture exit code, fail-closed on any non-zero exit. These are the binding answer for defensive-code constraints; model gates do not override.
2. **Constraint gate** — model-based assessment of every constraint from the step doc's "Constraints" section. **Even when mechanical checks cover a constraint, the constraint gate table must still be produced, with the "Evidence" column citing the mechanical check number (e.g., "Satisfied by mechanical check #3 — see above"). Abbreviating the gate to a single sentence ("All constraints covered by mechanical checks") deviates from the required format.** Each "Yes" requires file:line evidence quoted from the produced code or a mechanical check reference.
3. **Invariant gate** — model-based check of "Invariants to preserve" with explicit "how you verified" per invariant.
4. **Verification commands** — the step doc's bash commands (`npm run build`, `npm run lint`, etc.). Report results verbatim including failures.
5. **Acceptance criteria** — each criterion checked against the produced code with evidence.
6. **Verify output persistence** — before claiming PASS, `/verify` must write its full output (mechanical check results, constraint gate table, invariant gate table, verification command output, acceptance criteria checklist) to `docs/calibration/verify-<slug>-step-<N>.md` **when `/assess` will be run on the same step**. If `/assess` is skipped, verify output may remain in terminal scrollback only. This enables `/assess` and `/calibrate` to post-hoc verify that evidence was actually produced, not just claimed.
7. **`/assess` gate** — after `/verify` PASS, run `/assess <slug> <step-N>` before advancing `Current step` in `.pi/active.md`. `/assess` is the primary capture channel for calibration findings and the highest-density observation surface; skipping it loses per-step anti-flattery review that surfaces process friction (sketch drift, harness defects, silent improvements) that `/verify` gates do not catch. If `/assess` is consistently skipped, the calibration corpus becomes thin and `/calibrate` degrades to post-hoc guesswork.

Bake into every prompt:

> "Do not claim success until $CHECK passes. Report results verbatim, including failures."

K2.6 in instruct mode will report "done" when it isn't. Thinking mode reduces this materially but doesn't eliminate it. Without explicit gates, you'll find yourself debugging silent failures it confidently committed.

**Invariant check before claiming done** (cheap belt-and-suspenders regardless of mode):

> "Before claiming success, list the invariants this change must preserve — function signatures, public APIs, existing test names, side-effect contracts, performance characteristics. For each, state explicitly how you verified it's still preserved. Do not proceed without naming each."

Forces the model to reason about what *shouldn't* change, not just what should. Cheap, catches the regressions Opus would flag unprompted.

---

## 6. Hallucination defense (pre-flight)

Before any non-trivial implementation, force a pre-flight pass:

- **List files you'll touch.**
- **List functions/APIs you'll call** — grep each one to confirm it exists at the path you think.
- **List imports you'll add** — confirm the package is in `package.json` / `requirements.txt` / equivalent.
- **List project-level tsconfig/eslint constraints that could break the sketch** (e.g., `noUnusedParameters`, `verbatimModuleSyntax`, `react-refresh/only-export-components`, `allowImportingTsExtensions`). For sketches containing executable code, run `tsc --noEmit` and `eslint` on a temp copy before claiming PASS.

Catches confident-invention failures at near-zero cost. **For integration steps that touch only one existing file, use only previously-verified APIs, and create no new files, `/preflight` may be skipped if the implementer explicitly reads the step doc and the target file before editing. `/preflight` remains mandatory for steps creating new files, using new APIs, or touching >1 file.** Less critical in thinking mode (the model does some of this implicitly) but still worth it — empirically validate by inspecting whether thinking-mode K2.6 tries to invent APIs without the pre-flight; if it never does, the pre-flight becomes optional polish. Add as a required first step in implementation prompts until verified.

---

## 7. Distillation (post-ship)

After a roadmap phase ships, distill it into an **ADR** at `docs/adr/NNNN-<adr-slug>.md`. Run via `/distill <roadmap-slug>`. Note that the ADR slug differs from the roadmap slug — ADR slugs are past-tense decision form (e.g. `use-tursodb-for-edge-cache`), decided during distillation.

**Why this exists.** Roadmaps are working artifacts — heavy with workflow scaffolding (step lists, chunk caps, in-progress notes) that has zero value once the code is on the branch. Without distillation, the durable "why" behind each shipped phase lives only in commit messages and human memory. Pi has no subagents and no plan mode — institutional memory must live in files the agent reads on cold start, and `docs/adr/` is the small, decision-dense corpus that does that job.

**Naming conventions.**

- ADRs: `NNNN-<kebab-slug>.md`, sequential, never reused, four-digit zero-padded (`0001-`, `0042-`). Slug is past-tense decision form (`use-tursodb-for-edge-cache`, not `tursodb-decision`).
- Roadmaps: `<slug>.md` and `<slug>/step-N-<subslug>.md`. **No numeric prefix.** Roadmaps may produce 0 ADRs (abandoned), 1 ADR (typical), or N ADRs (multi-decision phases) — coupling roadmap numbers to ADR numbers creates either gaps or forced renames.
- Cross-link via frontmatter: ADR has `distilled-from: docs/roadmap/<slug>.md`; roadmap gains `produced-adrs: [NNNN, NNNN]` after distillation.

**Distillation contract.** Keep: context/problem, decision drivers, options considered with rejection reasons, chosen option, consequences (+/−/to-watch), validation signals. Drop: step lists, chunk plans, agent-harness mechanics, in-progress markers, anything true during the build but not true about the shipped system. Rewrite the 6-field reasoning template entries into prose — the table format is planning scaffolding, not record format.

**Supersession.** ADRs are append-only. When a later decision invalidates an earlier one, mark the old ADR `superseded-by: NNNN` and the new ADR `supersedes: [NNNN]`. The current state of the system is the set of un-superseded ADRs — this is what makes drift visible.

**When to skip distillation.** Trivial bugfixes, dependency bumps, doc edits, refactors with no public-API change. Distillation is for decisions worth re-litigating later, not for every shipped diff. If you can't write a decision driver paragraph that a future reader would care about, the change doesn't need an ADR.

---

## 8. Post-ship git operations

The workflow uses git worktrees and feature branches to enable parallel feature work. `/plan` creates them; `/finalize` merges them back to main. This is the parallelism-enablement layer — designed for 5+ in-flight features simultaneously without git collision.

### Branch + worktree at /plan time

When `/plan` is invoked inside a git repo:

- Branch: `feature/<slug>`
- Worktree path: `<repo-parent>/<repo-name>-<slug>` (sibling of the original repo)
- Roadmap and `.pi/active.md` are written INSIDE the worktree, NOT the original repo

The original repo's working tree on main is not modified. The roadmap exists only in the feature branch's worktree until `/finalize` merges back.

Pre-flight collision check is atomic: `/plan` refuses if `feature/<slug>` branch exists OR if the worktree path exists. No partial state — either both are created or neither.

Pi sessions in different worktrees do not see each other's state. Each has its own `.pi/active.md` (per-worktree state per Cross-cutting principles → State management).

**Backward compat — non-git-repo case**: `/plan` detects via `git rev-parse --is-inside-work-tree`. If not a git repo, falls back to in-place workflow: active.md in current dir, `Branch: <none>`, no worktree. The feature is already in the only working dir; `/finalize` is N/A.

### /finalize merge sequence

`/finalize <slug>` (or bare `/finalize` when state file resolves slug) ships a feature branch back to main:

1. **Pre-flight gates** — any failure stops, reports, exits:
   - Worktree clean (`git status --porcelain` empty)
   - All step status checkboxes in active.md are `[x]`
   - ADR exists at the path resolved from roadmap's `produced-adrs:` frontmatter
   - Current branch in worktree matches `feature/<slug>`
2. **Rebase**: `git fetch && git rebase origin/main` in the worktree. On conflict: stop, report conflicted files, exit. User resolves manually and re-runs.
3. **Merge**: `git -C <orig-repo> checkout main && git -C <orig-repo> merge --ff-only feature/<slug>`
4. **Cleanup**:
   - `git worktree remove <worktree-path>`
   - Confirm before `git branch -d feature/<slug>` (user prompt; not auto-deleted)
5. **Roadmap footer**: append `Shipped: <date> | Branch: feature/<slug> | ADR: NNNN` to the roadmap (now in main)

**Step-doc awareness**: When the workflow uses per-feature worktrees (Branch + worktree at /plan time, above), the final commit/push step doc must explicitly include the `git merge --ff-only feature/<slug>` operation in the main worktree, not assume the commit is already on `main`. The commit is created in the feature worktree and must be fast-forward-merged to `main` in the sibling worktree before push.

### Hard rules

- **No auto-push.** `/finalize` stops after the local merge. User pushes manually per the doctrine principle "user confirms before pushing." Push affects shared state and warrants explicit confirmation per Executing actions with care.
- **`--ff-only` enforces clean linear history.** Validated workflow on user's prior 19-package monorepo. If `--ff-only` fails, the rebase didn't catch all upstream changes — re-run `/finalize` after re-rebasing in the worktree.
- **Conflict handling is manual.** `/finalize` never resolves conflicts automatically. It stops, reports, exits. User resolves and re-runs.
- **No CI hooks integrated by default.** If a project requires pre-merge CI, that's a per-project addition (e.g., a wrapper around `/finalize` or a pre-merge check). Doctrine doesn't mandate CI integration.

### Bucket: portable

Git worktree + rebase + ff-only merge is git semantics, not pi-specific. Pattern survives model swaps and pi version changes. Validated empirically on user's prior Claude Code workflow with Opus on 19-package monorepo.

---

## Calibration notes

- **Validate K2.6 thinking-mode chunk cap against your real workload.** The 15-20 cap was calibrated against assumed instruct-mode behavior. Moonshot claims K2.6 "scales horizontally to 300 sub-agents executing 4,000 coordinated steps" — discount that for marketing, but plausible cap is 40-60 in thinking mode. **Observed on dark-mode-toggle (n=1)**: K2.6 thinking mode sustained 39 tool calls (Step 6 across preflight+implement+verify) and 50+ across two implement runs (Step 4) without repeating work, inventing APIs, or refactoring earlier code. Default stays at 15-20 until n≥2 features confirm; document each feature's observed cap here as data accumulates.
- **Cost-driven configuration**: K2.6 input is $0.95/M uncached, $0.16/M cached. Keep the prompt prefix stable across chunks (same system prompt, same file reads in same order) to bank the 83% cache discount. Verify with `prompt_tokens_details.cached_tokens` in Fireworks usage telemetry — if cached fraction is below ~30% on chunked sessions, prefix order is breaking caching and there's a fix.
- **Cross-model regression check** (when both models work in the harness): same plan → run K2.6 and GLM 5.1 → diff outputs against the reasoning template fields. Divergent outputs at the same field = the step's constraints are too loose. The harness contract is doing its job when this catches things.
- **Perspective rotation produces near-zero overlap between critique passes on the same model.** Validated on dark-mode-toggle: senior-eng pass caught Vite CSS-injection ordering, `setItem` error handling, `matchMedia` cleanup, step-ordering inversion; PM pass caught UI specification, accessibility, cross-tab sync scope, objective no-flash criterion, browser/OS matrix. Zero overlap. Same pattern observed on prior Opus-era runs. Likely the actual load-bearing piece of `/critique`.
- **K2.6 silent-improvement-mid-write tendency** — validated on dark-mode-toggle Step 6: model silently removed an unused `index` parameter from `options.map((option, index) => {` during verbatim transcription, avoiding a `noUnusedParameters` lint failure. The fix was correct but constitutes drift from the step doc. Mitigation: stricter "transcribe, do not improve" prompting in cold-start procedure, plus mechanical checks that detect parameter count mismatches. Accept silent improvements only when they fix known project constraints (lint, typecheck) without changing logic, and only if the delta is ≤1 parameter removal or ≤1 import reorder.
- **Known harness friction (pi 0.70.2)**:
  - Fireworks's GLM 5.1 endpoint rejects the `enable_thinking` field that pi sends regardless of the `compat: { thinkingFormat: "zai" }` declaration in `~/.pi/agent/models.json`. Workarounds in priority order: (a) **MiniMax M2.7** (same provider, no thinkingFormat compat hack needed; $0.30/$1.20 input/output, cheaper than GLM); (b) Anthropic/OpenAI/Google subscription via `/login`; (c) fall back to K2.6 same-model perspective rotation. File pi issue at github.com/badlogic/pi-mono/issues. Until fixed, GLM 5.1 cannot be relied on as a documented critic.
  - `/reload` does not reliably refresh the prompt template registry. After editing `~/.pi/agent/prompts/*.md`, restart pi (Ctrl+C twice, relaunch) to force a rebuild. AGENTS.md content does reload via `/reload`; prompt cache may persist. Symptom: the next slash-command invocation produces output identical to the previous run despite the template having changed.
  - `/preflight` output file naming (`step-N.preflight.md`) does not match `/implement`'s discovery glob (`step-N-<slug>.preflight.md`), causing the cold-start procedure to report "NO_PREFLIGHT" even when a focused preflight exists. Fix: align `/preflight` output naming with `/implement` discovery glob, or make `/implement` glob more permissive (`step-N*.preflight.md`).
  - Verified changes remain uncommitted by default. `/assess`'s `git diff HEAD~1` strategy fails for the most recent step because the commit does not yet exist. Fallback to `git diff` (working tree vs HEAD) or `git status` is required. Either (a) update `/assess` to use `git diff` as primary diff source when on the feature branch (applied in `~/.pi/agent/prompts/assess.md`), or (b) commit on `/verify` PASS with a templated message.
- **Future self-host** (Qwen3-Coder, DeepSeek V4 distills): re-test chunk cap and critique-pass effectiveness from scratch.
- **Planning quality dropping? Expand the reasoning template before swapping models.** Add required output fields (e.g., "list invariants this change preserves across the codebase," "name what breaks if we do the opposite," "rank the alternatives by reversibility"). The model is rarely the bottleneck — prompt scaffolding usually is. Each new required field is a piece of implicit reasoning made explicit.
- **The workflow is the harness contract.** When a model fails this workflow, investigate the harness before blaming the model. When a model passes this workflow on a task that previously failed, the harness was the bottleneck, not the model.

---

## When to revisit this doc

- New model added to providers (K2.7, GLM 6, self-host) — re-evaluate primary/critic assignment against current K2.6 + GLM 5.1
- Chunk-size empirics drift (you measured the actual cap and want to update the default)
- Cross-model critique consistently finds non-overlapping gaps perspective-rotation misses → re-promote to non-optional. Two consecutive runs where it adds nothing new → fully optional polish.
- pi/GLM `enable_thinking` bug fixed (or pi adds a `--no-thinking-flag` escape hatch) → cross-model becomes operationally cheap again, retest the premium claim.
- LoRA fine-tune of K2.6 lands → reasoning-template scaffolding may bake into the model, simplifying prompts
