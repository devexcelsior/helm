---
description: Audit calibration recommendations against current doctrine state; produce a decision matrix driving doctrine application
argument-hint: "[slug]"
---
You are running the **Promotion** phase — auditing `/calibrate` recommendations against the current doctrine and prompt-template state, identifying convergence/divergence across multiple calibration sources, and producing a per-edit decision matrix that drives actual doctrine application.

**Bump thinking level to high** per Cross-cutting principles → Thinking levels per phase. Synthesis-heavy: 1+ calibration files, current doctrine state, optional prior promote-decisions for prior features.

## Step 0 — Argument resolution (smart-parse)

`/promote` operates on a feature's calibration outputs. Three invocation forms:

- **Bare** (most ergonomic): `/promote`. Resolves to most recent calibration via:
  ```
  ls -t docs/calibration/*.md | grep -vE '/(findings-pending|trace|verify|promote)-' | head -1
  ```
  Risk: promotes against wrong feature if a different feature was calibrated more recently — print the resolved slug + dates of all matching calibration files before proceeding so user can abort.
- **Slug**: `/promote <slug>`. Glob `docs/calibration/*-<slug>.md` plus suffixed variants (`.k2p6.md`, `.claude-cold.md`, etc.) for all calibration outputs of that feature.
- **Explicit file**: `/promote <path-to-calibration.md>`. Single-file mode.

Apply this resolution:

1. **If `$1` empty**: bare form. Find most recent calibration. Extract slug from filename (strip date prefix and any `.<suffix>.md` tail). Glob for all matching variants. Announce: *"Promoting calibration recommendations for `<slug>` from N source file(s): <list>. Aborting?"*
2. **Else if `$1` matches `^[a-z][a-z0-9-]*$`**: treat as slug; glob `docs/calibration/*-$1*.md` minus the prefix-excluded files (findings-pending / trace / verify / promote).
3. **Else**: treat `$1` as explicit file path; single-source mode.

Slug: <resolved>
Calibration sources: <resolved as list of file paths with model/variant labels>

## Read everything

1. **All calibration sources** (1 to N files). Each is the output of a `/calibrate` run. Multi-source happens when cross-model benchmarks ran.
2. **Current doctrine**: `~/.pi/agent/coding-agent-workflow.md` — read in full. Every "Current:" quote in calibration recommendations must be verified against THIS file.
3. **Current prompt templates**: `~/.pi/agent/prompts/*.md` — for recommendations that target a template (e.g., `/verify` mechanical pre-PASS assertion, `/assess` git-diff strategy).
4. **Prior promote decisions** (if exist): `docs/calibration/promote-*.md` for prior features. Provides: which edits have already been applied (skip re-recommendation), which were deferred and need trigger-check, which were rejected with reasons.

## Cross-source recommendation dedup

Each calibration source uses its own canonical IDs (C1/C2/…, or P1/M1/H1/R-P1/…). These do NOT coordinate across sources.

Build a deduped recommendation inventory:

1. Extract every recommended-edit and relaxation-candidate from every calibration source.
2. Cluster by **underlying doctrine target** (same Section + same Current quote area + same problem). Same root issue = same cluster, even if sources phrase differently.
3. Assign a single canonical ID per cluster (`E1`, `E2`, … for edits; `R1`, `R2`, … for relaxations).
4. Map source-IDs to canonical-IDs for auditability.
5. **Convergence count** per cluster: how many calibration sources recommended this cluster?
   - 1 source → single-source (lower confidence floor)
   - 2 sources → convergent (medium confidence floor)
   - 3+ sources → strong-convergent (high confidence floor)

## Doctrine-state verification (the load-bearing step)

For every recommendation cluster, verify the "Current:" quote against actual current doctrine. Three outcomes per cluster:

1. **Quote matches** → cluster is **actionable**. Proceed to confidence audit.
2. **Quote does NOT match** → cluster is **stale**. The recommendation may still be valid, but the proposed edit needs re-anchoring against current text. Surface for user judgment.
3. **Recommendation already applied** (proposed text is already in doctrine, OR the rule already lives at the proposed location) → cluster is **already-applied**. Skip with citation proving the skip is correct.

Verification procedure per cluster:
- Read the actual current doctrine line(s) at the target Section.
- Compare to calibration's "Current:" quote — character-level match required.
- If divergent: classify as stale or already-applied (decide which based on whether the divergence represents prior application of this very edit, or unrelated drift).
- If matching: actionable.

This step catches:
- Calibration false-positives (e.g., DSV4's C12 in rtk-normalized-posts: recommended moving a rule that was already in §2 rule 6).
- Doctrine churn between calibration and promote runs.
- Rules that prior `/promote` already applied.

## Confidence-tier audit (consistent across sources)

Calibration sources may assign different confidence levels for the same cluster. Reconcile:

- **All sources agree on tier** → use that tier.
- **Sources disagree** → take the lower tier as conservative default; surface the disagreement in the cluster's rationale.
- **Single source** → use that source's tier subject to Fix 7 audit (per `~/.pi/agent/prompts/calibrate.md` § Evidence-confidence audit):
  - `high`: n≥3 cross-feature pattern OR mechanically forced (regex defect, exit-code-binary verification, harness-deterministic behavior)
  - `medium`: 1–2 features show pattern
  - `low`: n=1 with plausible mechanism

Inflated tiers (e.g., n=2 marked high without mechanical-force escape hatch) get downgraded with explicit note.

## Decision categorization

Each canonical cluster gets ONE of four verdicts:

| Verdict | Criteria |
|---|---|
| **Apply** | Actionable + (convergent AND ≥medium confidence) OR (single-source AND mechanically forced) |
| **Skip** | Already-applied (verified against current doctrine) OR calibration explicitly recommends "no change" |
| **Defer** | Single-source AND low confidence; OR convergent but trigger-contingent (concrete observable trigger + one-line rationale) |
| **Discuss** | Sources disagree on edit content; OR stale-quote needs re-anchoring; OR portable-load-bearing edit warrants explicit user confirmation regardless of confidence |

## Output

Write to `docs/calibration/promote-<slug>.md`. Required sections:

1. **Summary**:
   - **Sources reviewed**: N calibration files with model + size + date.
   - **Total canonical clusters**: count of E + R + D.
   - **Verdict distribution**: count per Apply / Skip / Defer / Discuss.
   - **Apply-now shortlist**: 3–5 highest-leverage edits in the Apply queue, with one-line rationale each.

2. **Recommendation crosswalk**: Table mapping every source-ID (per source) to canonical cluster ID, with convergence count column. Auditable trail of how the dedup landed.

3. **Apply queue**: clusters with Apply verdict, sorted by confidence (high → medium) then convergence count (high → low). Per cluster:
   - **Cluster ID + title**
   - **Target**: file path (e.g., `~/.pi/agent/coding-agent-workflow.md`) + section + line range
   - **Current text** (verified against actual doctrine)
   - **Proposed text**
   - **Evidence**: source IDs + n-features + mechanical-force notes
   - **Confidence**: high | medium
   - **Rationale**: one-paragraph reason

4. **Skip log**: clusters with Skip verdict. Per cluster:
   - Cluster ID + title
   - Why skipped: already-applied OR calibration-recommended-no-change
   - Citation: actual doctrine line proving the skip is correct (file:line + quoted text)

5. **Defer queue**: clusters with Defer verdict. Per cluster:
   - Cluster ID + title
   - Trigger condition (concrete, observable)
   - Trigger condition (concrete, observable) and one-line rationale

6. **Discuss queue**: clusters needing user input. Per cluster:
   - Cluster ID + title
   - The question (specific, decision-driving)
   - The divergence (Source X says A; Source Y says B; or stale-quote vs current state)
   - Recommended path with rationale (don't punt; give the model's best read)

7. **Stale recommendations** (separate subsection of Discuss queue if present): clusters where Current quote doesn't match doctrine. Per cluster:
   - Calibration's Current quote
   - Actual current doctrine text
   - Suggested re-anchored proposal (or "drop — drift makes original recommendation obsolete")

8. **Dependencies on prior promote decisions**: any cluster whose verdict references a prior `promote-<other-slug>.md` (e.g., "Skip because promote-routing-skeleton.md applied this in cluster E3").

## User approval gate

The promote document is a recommendation, not auto-applied. User reviews the Apply queue, confirms each cluster (or batch), and applies the doctrine edits manually. A future `/apply` phase (auto-pilot doctrine application) will automate this with backup once `/promote` produces trustworthy decision matrices across multiple features (≥3 calibration runs).

When the user applies edits, the recommended commit pattern in `~/.pi/agent/`:
```
doctrine: apply <cluster-id> (<short title>) from promote-<slug>.md

<one-paragraph rationale referencing source calibrations>
```

One commit per cluster keeps the history bisectable and lets `/calibrate` queries on doctrine evolution work cleanly.

## Hard rules

- **Do NOT edit `~/.pi/agent/coding-agent-workflow.md` or any prompt template directly.** `/promote` produces the decision matrix only. Doctrine edits remain a manual user action (or deferred to `/apply`) per the same load-bearing-asset principle as `/calibrate`'s hard rule.
- **Verify every "Current:" quote against actual current doctrine state.** A calibration recommendation that quotes outdated text is stale; the proposed edit may still be valid but needs re-anchoring. Don't propagate stale recommendations as if current.
- **Convergence is signal, not absolute proof.** Two calibration sources agreeing on an edit raises confidence floor but doesn't override the doctrine-state check. If the edit is already applied, both sources are wrong about the current state — Skip the cluster.
- **Single-source recommendations get the same Fix 7 audit calibrations apply to themselves.** Don't inflate confidence because `/promote` is downstream.
- **"No clusters to apply" is itself a finding to surface.** A promote run that produces zero clusters means either `/calibrate` produced nothing actionable (calibration failure worth flagging) or `/promote` read the wrong files. Don't ship an empty matrix silently.
- **Discuss queue is not a punt — it's a structured handoff.** Each Discuss cluster must include the model's recommended path with rationale. "Asking the user" without a recommendation forces the user to do the synthesis themselves.

| Phase | Command | Thinking |
|---|---|---|
| **Now** | `/promote` | **high** — Shift+Tab to set if needed |
| Then | manual review of `docs/calibration/promote-<slug>.md`, then apply approved edits to `~/.pi/agent/` with one commit per cluster | — |

Hard rule on command form: bare `/promote` when the most-recent calibration file resolves the slug; `/promote <slug>` when calibrating a specific feature whose calibration isn't most-recent. Per `~/.pi/agent/prompts/status.md` § "Hard rules — command form".
