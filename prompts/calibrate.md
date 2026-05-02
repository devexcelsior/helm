---
description: Audit the doctrine against a shipped roadmap+ADR pair; produce concrete edit recommendations
argument-hint: "<slug> <adr-NNNN>"
---
You are running the **Calibration** phase — auditing the operating doctrine in `~/.pi/agent/AGENTS.md` against what actually happened on a shipped feature. Output is concrete edits to the doctrine, not vibes. This is the feedback loop that keeps the harness honest.

**Bump thinking level to high** per Cross-cutting principles → Thinking levels per phase. Synthesis across full corpus: roadmap, critiques, step docs, ADR, run telemetry.

## Step 0 — Argument resolution (smart-parse)

`/calibrate` derives slug from the ADR's frontmatter (`distilled-from:` field), so the slug is never required as an argument. Three invocation forms:

- **Bare** (most ergonomic): `/calibrate`. Resolves to the most recent ADR via `bash ls -t docs/adr/*.md | head -1`. Risk: calibrates against wrong feature if a different feature was distilled more recently — print the resolved ADR number + title before proceeding so user can abort if mismatched.
- **ADR-only**: `/calibrate <ADR-NNNN>`. Glob `bash ls docs/adr/$1-*.md` (expect one match). Read `distilled-from:` from frontmatter to get roadmap path → slug.
- **Legacy explicit**: `/calibrate <slug> <ADR-NNNN>`. Both args explicit. Still works for backward compatibility.

Apply this resolution:

1. **If both `$1` and `$2` empty**: bare form. Find most recent ADR. Derive slug from its `distilled-from:`. Announce: *"Calibrating against ADR <NNNN> — <title> (slug: <slug>). Aborting?"*
2. **Else if `$1` matches 4-digit numeric AND `$2` empty**: `$1` is ADR number. Glob `docs/adr/$1-*.md`. Derive slug from its `distilled-from:`.
3. **Else**: `$1` is slug, `$2` is ADR number (legacy form).
4. **In all cases**: confirm both slug and ADR number resolved before proceeding. Stop with error if either can't be determined.

The slug is needed for finding the roadmap, critiques, and step docs (`docs/roadmap/<slug>/<slug>.md`, `docs/roadmap/<slug>/<slug>.critique-*.md`, `docs/roadmap/<slug>/step-*.md`). Always sourced from the ADR's frontmatter, never from `.pi/active.md` — that file is deleted by `/distill` before `/calibrate` runs.

Slug: <resolved>
ADR number: <resolved> (4-digit zero-padded, e.g. `0001`)

Roadmap: `docs/roadmap/$1/$1.md`
Critiques: discover via `bash ls docs/roadmap/$1/$1.critique-*.md`
Step docs: `bash ls docs/roadmap/$1/step-*.md`
ADR: discover via `bash ls docs/adr/$2-*.md` — should match exactly one file.

## Read everything

1. `~/.pi/agent/coding-agent-workflow.md` — full doctrine.
2. `docs/roadmap/$1/$1.md` — the roadmap (final revised version).
3. All `docs/roadmap/$1/$1.critique-*.md` files.
4. All step docs under `docs/roadmap/$1/`.
5. The ADR matched in pre-flight.
6. `bash git log --oneline --since="$(stat -c %y docs/roadmap/$1/$1.md | cut -d' ' -f1)"` — commits delivered against this roadmap.
7. **`docs/calibration/trace-$1.md`** if it exists — accumulated `/assess` per-step observations with anti-flattery scans and classified findings. This is the highest-density observation channel; weight accordingly.
8. `docs/calibration/findings-pending-$1.md` if it exists — manual mid-run observations (legacy / discretionary capture).
9. **`docs/workflow/workflow-concerns.md`** if it exists — analyzed concerns with proposed fixes, scoped between raw observations (sources 7+8) and prior calibration/promote deferred items. Each open concern needs a verdict: **Accept** (apply as edit), **Defer** (hold for future trigger), **Reject** (archive with reason), or **Clarify** (leave open, prompt user). Treat this as the highest-leverage input alongside the trace file — concerns here have already been scoped with tradeoffs.

## Cross-source finding dedup (required before scoring)

Findings can appear in multiple sources with independent F-numbering:
- Trace file uses F1, F2, F3, … per step (assigned by `/assess` at capture time)
- Findings-pending file uses F1, F2, … (assigned manually as discretionary captures)
- Workflow-concerns uses 1, 2, … (numbered list of concerns)

These numbering schemes do NOT coordinate. A single underlying observation may surface as trace-F5 *and* findings-pending-F2 *and* concerns-#3. Before per-phase scoring, build a deduped finding inventory:

1. Extract all findings from sources 7, 8, 9 (trace, findings-pending, concerns).
2. Cluster by underlying observation (semantic match, not literal text match — same root cause = same cluster).
3. Assign a single canonical ID per cluster (`C1`, `C2`, …).
4. Map source-IDs to canonical-IDs in the calibration output's "Finding map" section so the trail is auditable.
5. Use canonical IDs in the recommended-edits sections.

If a cluster has only one source, that's fine — it gets its own canonical ID. The dedup is to prevent counting one issue as three separate findings and recommending three duplicate edits.

6. **Singleton audit**: After the initial cluster pass, list all singletons (clusters with only one source). For each, ask: *"Does this observation share an underlying root cause with another cluster?"* Common merge patterns:
   - Multiple "validation gap" observations (extension-vs-content, lint-vs-sketch, typecheck-vs-sketch) → may merge into one "decompose-time validation surface" cluster
   - Multiple "model behavior" observations (zero drift, follows STOP rule, silent improvement) → may merge into one "K2.6 thinking-mode behavior profile" cluster
   - Multiple "harness friction" observations (preflight naming, /reload cache, GLM bug) → may merge into one "pi 0.70.2 friction patterns" cluster

   If a merge is plausible, do it. Singletons that survive the audit get noted as *"Distinct because <one-line reason>."*

7. **Cluster size sanity check**: A typical 6-step feature should produce roughly 8-15 canonical clusters. If you have 20+ singletons, the clustering is too granular. If you have 3-4 huge clusters, it's too coarse. Aim for the middle.

## Score each phase

For each phase below, output four lines: **Observed** (what happened, with quoted evidence), **Doctrine claim** (what `coding-agent-workflow.md` said would/should happen, quote the section), **Gap** (delta — or "none"), **Recommended edit** (specific section + new wording, or "no change").

Per Cross-cutting principles → Evidence over claims, do not handwave. If the evidence is thin, say "1 data point — low confidence" and propose nothing yet.

### §1 Planning
- Did each major decision have a non-strawman entry in field 4 ("deceptively obvious wrong answer")?
- Were "Alternatives considered" real options with real rejection reasons, or padding?
- Did the self-consistency check ever catch a contradiction, or was it always "no contradictions"?
- Did the cross-cutting concerns section cover all 7 (testing, accessibility, error handling, observability, deployment, security, performance budgets)?

### §1 Discipline-axis check
- For each axis (deferred decisions, unjustified specifics, circular rejections, missing cross-cutting concerns): did `/critique` surface findings? Did `/revise` resolve them or hedge?
- Did any axis-class issue survive into step docs or implementation? Quote the survival.

### §1 Cross-model vs perspective rotation
- If cross-model was used, quote one thing the cross-model critic raised that K2.6 self-critique would have missed. **If none, this is the load-bearing data point** — flag it explicitly.
- If perspective rotation only, did the two passes find non-overlapping issues? Quote one finding from each.

### §2 Decomposition
- Were step docs self-contained, or did `/implement` need clarification mid-chunk?
- Did "files touched" or "API signatures" turn out wrong (i.e., the planner guessed)?
- Were invariants concrete (testable) or generic ("preserve API")?
- Did the consistency sweep produce quoted evidence per §3, or rubber-stamp?
- Did the step-list-shape check catch any silent merges/splits?

### §3 Implementation chunks
- Empirical chunk cap: how many tool calls per chunk before *any* of repeating earlier work / inventing APIs / refactoring code written earlier in session? Report the round count.
- Compare to the doctrine's 15-20 default. If you sailed past 30 without drift, recommend raising — quote the exact line to edit.
- Did `/implement` ever "improve" code mid-write (deviating from step doc snippets)? Quote any deviation and whether it was step-doc bug or model bug.

### §4 Cold starts
- Did `/new` between steps matter? If you ran two steps in the same session by accident, did anything go wrong?
- If nothing went wrong, the cold-start discipline may be over-strict for this stack.

### §5 Verification gates / §6 Pre-flight
- Did `/verify` catch any invariant violations? Or rubber-stamp every step?
- Did `/preflight` catch any invented APIs? Or was it always confirming things the model already knew?
- Did either phase produce evidence (file/line/command output) per the Cross-cutting principles requirement, or claim "verified" without showing work?

### §7 Distillation
- Re-read the ADR cold (pretend you're 6 months from now, no other context). Does it tell you why the code looks the way it does?
- Reads like a cleaned-up roadmap → didn't strip enough scaffolding. Reads like a commit message → stripped too much. Quote the offending paragraph if either.

### `/assess` phase (post-`/verify` per-step capture)

- For each step that ran `/assess`, did the anti-flattery questions surface findings beyond what `/verify` had already produced? Quote one finding from each `/assess` entry that wasn't trivially derivable from the verify output.
- If `/assess` produced findings that `/verify` already implicitly captured, the redundancy isn't necessarily wasted (different framing surfaces different things) but should be noted.
- Did `/assess` ever produce a "looks good across the board, no findings" entry? That's the failure mode the anti-flattery prompting was designed to prevent. If it happened, `/assess` is broken; recommend tightening prompts.
- If `/assess` consistently produced 3+ substantive findings per step, it's earning its keep — recommend promoting from optional to required-phase status.
- If `/assess` is consistently producing low-value findings (e.g., implementation drift "none" without quoted evidence), tighten prompts with more pointed questions.

**Doctrine claim**: `/assess` exists because *"the same model that produced the work is now reviewing it. Without explicit prompting it'll mostly say 'looks good.'"* Anti-flattery questions are the forcing function.

### Cross-cutting principles
- Was "Evidence over claims" honored across phases, or did some phases produce unsupported claims?
- Did the per-phase thinking-level guidance match the actual difficulty? Note any phase where the default felt wrong.
- Was the pre-flight smoke test (smoke `/critique` on a throwaway doc) run? If skipped, did harness friction cost time?

## Findings classification (3-bucket — required)

For every finding produced from per-phase scoring AND from the trace file's classified findings, assign one of three buckets. This is non-negotiable per Cross-cutting principles → Calibration buckets (in the doctrine).

| Bucket | Definition | Examples of doctrine claims that go here |
|---|---|---|
| **portable** | Applies regardless of model/harness; survives K2.6 → K2.7 and pi 0.70 → pi 0.71 | Evidence-over-claims, perspective rotation, response-table revision, mechanical verification with self-validation, generalized deferred-decision scan |
| **model-specific** | Tied to current model behavior | Chunk cap of 15-20, thinking-level guidance, K2.6's silent-improvement tendency, K2.6's OR-clause exploitation pattern |
| **harness-specific** | Tied to pi version or workflow tooling | `enable_thinking` bug on Fireworks GLM, `/reload` cache friction, gawk `\b` interpretation, prompt template format |

Trace file findings tagged `[?]` must be resolved into one of the three buckets before the calibration doc ships. If a finding genuinely fits multiple buckets, pick the dominant one and note the other in parentheses.

## Output

Write to `docs/calibration/<YYYY-MM-DD>-$1.md`. Sections:

1. **Summary** (three required sub-points):
   - **Earn-its-weight verdict**: did the workflow earn its weight on this feature? Where did it pull, where did it drag?
   - **Comparison to baseline (when stated)**: if the user has stated empirical experience with the workflow shape on prior work (different harness, different scale, different model), explicitly compare this run's velocity, drift rate, and friction patterns against that baseline. Comparable axes: time-per-step, step-doc correction cycles, mechanical-check defect rate, cross-model critique value, distillation quality, calibration finding density. If no baseline stated, write *"No baseline available — first run on this stack/harness. Calibration is establishing the reference, not measuring against one."* This is **not** a substitute for per-phase scoring — it's the single highest-leverage question for the user, especially when calibrating a new harness against a known-good workflow shape.
   - **Auto-pilot trigger status**: state which feature this is in the calibration sequence (e.g., *"calibration run #N of dark-mode-toggle, feature 1 of 2-3 needed"*) and current bucket-classification confidence (subjective high/medium/low based on how confident the classifications felt). Auto-pilot doctrine application (not yet implemented) is triggered after `/calibrate` produces trustworthy bucket classifications across 2-3 features. State estimated runs remaining before auto-pilot trigger. This explicit gate-tracking prevents auto-pilot from being implemented prematurely (e.g., on n=1 evidence) and gives the user concrete visibility into the maturation curve.
2. **Finding map**: deduplication audit trail. Two-column table mapping every source-ID (trace-F<N>, pending-F<N>, concerns-#<N>) to its canonical cluster ID (C<N>). Each canonical ID appears once with the underlying observation summarized in one line. Per-phase scoring and recommended edits below use canonical IDs.
3. **Per-phase scoring**: the four-line block for each phase above. Reference findings by canonical ID.
4. **Concern verdicts** (only if `workflow-concerns.md` was read): for each open concern, produce one of:
   - **Accept**: applied as a doctrine/template edit in section 5/6/7 below.
   - **Defer**: note the trigger condition (concrete, observable) and a one-line rationale. Hold for future calibration review.
   - **Reject**: one-line reason. No further action — concern is dropped.
   - **Clarify**: leave open; prompt user with the specific question that needs answering before proceeding.
5. **Recommended doctrine edits — Portable**: edits to portable doctrine sections (Cross-cutting principles, §1 Discipline-axis check, §2 cross-step consistency contract, §5 verification gates, §7 Distillation core). Format:
   ```
   Section: §N <title>
   Current: "<exact line from coding-agent-workflow.md>"
   Proposed: "<replacement>"
   Reason: <evidence from this run; reference canonical finding IDs>
   Evidence strength: <n=1 single-feature | n=2-3 cross-feature | n≥3 cross-feature | mechanically forced>
   Confidence: high | medium | low
      - high: n≥3 cross-feature pattern OR mechanically forced (regex defect, harness incompatibility, exit-code-binary verification that consistently fails on incorrect code)
      - medium: 1–2 features show pattern; recommended but watch for inversion on next features
      - low: n=1 with plausible mechanism; default to "Findings to defer" unless trigger justifies acting now (state the trigger in Reason)
   ```

   **Evidence-confidence audit** (applies to sections 5, 6, 7): any edit with `n=1` evidence (single-feature observation) defaults to `medium` confidence unless the mechanism is mechanically forced. To upgrade to `high`, cite ≥2 features OR an exit-code-binary verification that consistently fails on incorrect code (e.g., a regex defect that produces false PASSes on broken implementations). To downgrade to `low`, the edit moves to section 11 "Findings to defer" with a trigger condition. Marking `high` confidence on `n=1` evidence is the failure mode this rule defends against — it propagates extrapolation into doctrine before the data warrants it.
6. **Recommended doctrine edits — Model-specific**: edits scoped to model-tied doctrine sections (Cross-cutting principles → Thinking levels per phase, §3 Implementation chunks chunk cap, Calibration notes about K2.6 behavior). Same format including `Evidence strength:` and tiered `Confidence:`. These edits get re-validated when the model changes.
7. **Recommended doctrine edits — Harness-specific**: edits scoped to harness-tied doctrine sections (Calibration notes → Known harness friction). Same format including `Evidence strength:` and tiered `Confidence:`. Re-validated on pi version change.
8. **Recommended doctrine relaxations (anti-accumulation)**: for each rule in the doctrine that **did not fire** during this run (or fired but added no observable value), evaluate whether the rule is genuinely earning its keep or is ceremony. Produce relaxation candidates per bucket (portable / model-specific / harness-specific). **Each bucket must produce at least one candidate OR an explicit "no relaxation candidates this run because <reason>" justification.** "Couldn't think of anything" is not a justification — that's the same accumulation bias this section exists to fight. Same format as add-this edits (Section / Current / Proposed / Reason / Evidence strength / tiered Confidence), where Proposed is the relaxed version. The Evidence-confidence audit applies — relaxation candidates with n=1 evidence default to `medium` confidence.

   Relaxation patterns to watch for:
   - **Mechanical check coverage on trivial edits**: pure config edits with no executable code may earn an exemption from mechanical-check ceremony if the absence is defensible.
   - **Self-consistency check on orthogonal decisions**: if N/N decisions return "no contradictions," the field may be over-engineered for plans where decisions are genuinely orthogonal.
   - **Cross-model critique under-performance**: if cross-model produces no findings unique to perspective rotation across consecutive runs, retire per existing doctrine rule.

   **Hard rule**: if every bucket returns "no candidates," re-read the doctrine with the question *"which rule did not fire on this run?"* and produce at least one candidate. Empty relaxation lists across all buckets is itself a calibration failure mode worth noting.
9. **Validated doctrine claims (keep-this)**: for each doctrine claim that **fired during this run AND produced observable value**, record it as validated. Positive-signal counterpart to relaxation candidates in section 8. Same bucket structure (portable / model-specific / harness-specific). Format:
   ```
   | Claim | Evidence from this run | Bucket |
   |---|---|---|
   | <doctrine claim text> | <file:line, command output, or quoted observation> | portable / model-specific / harness-specific (pick one) |
   ```
   Future calibrations should reference this list when considering relaxation candidates: a claim with multiple "validated" entries across features should not be relaxed even if it doesn't fire on a particular run. Antidote to two failure modes: (a) future calibration proposing to retire a load-bearing rule because the data wasn't aggregated; (b) doctrine accumulating rules without anyone tracking which actually earn their keep.
10. **Findings to defer (low-confidence)**: low-confidence observations worth tracking across calibration runs but not editing on yet. Still classified into the three buckets. Include trigger condition for each.

Auto-pilot doctrine application is **not yet implemented**. For now, every recommendation requires explicit user approval before any edit lands. The classification structure is the prerequisite for future auto-pilot; the human-approval gate is the safety layer until classification proves trustworthy across multiple features (≥3 calibration runs with ≥95% bucket-classification accuracy).

## Hard rules

- **Do NOT edit `~/.pi/agent/coding-agent-workflow.md` directly.** Surface diffs and ask the user to confirm each high-confidence edit before applying. The doctrine is load-bearing; one bad calibration run could degrade every future feature.
- **"No changes needed" is a red flag, not a pass.** Either the calibration questions are too soft (update this template) or you didn't look hard enough. One signal worth recording per phase, minimum.
- **Two failure modes to avoid**: (a) recommending edits that just match the model's recent vibes — calibration is empirical, not aspirational; (b) recommending nothing because the run "felt fine" — feel is not signal.
