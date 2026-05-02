# helm

How to sail. A rigorous methodology for coding agents.

---

## The problem

Coding agents are fast. They're also confidently wrong — regularly. Standard review catches some of their errors but the model is better at arguing *past* a reviewer than any junior engineer you've ever managed. The best engineers cope by stopping the agent every few tool calls to re-read its output — which defeats the speed gain.

The core risk isn't that agents write bad code. It's that they produce **plausible-looking code that passes the obvious checks** and then silently fails on edge cases, state management, or cross-cutting concerns that no single reviewer is looking for.

---

## What helm is

A governance methodology that wraps the coding agent. The agent produces code; helm critiques, decomposes, verifies, and distills. It's the layer between model output and production merge — harness-agnostic, evidence-gated, and self-improving.

```
/plan → /critique (PM) → /critique (senior eng) → /revise
→ /decompose → /preflight → /implement → /verify → /assess
→ /distill → /calibrate → /finalize
```

A reference implementation exists: `pi-orchestrate` — a single bash command that drives all 11 phases with correct model assignment, thinking-level settings per phase, optional human-in-the-loop checkpoints, and structured per-phase logging. The methodology is a spec; the script is one harness that implements it.

Four things make it different:

### 1. Perspective-rotation critique — same model, different role

Every plan gets critiqued from two rotating perspectives (PM requirements coverage, senior-engineer risk/edge cases). Same model each time, different role. Empirically, this produces **near-zero overlap** between passes — the model's attention shifts to different failure surfaces with each role prompt, surfacing gaps a single-pass review would miss.

### 2. Cross-model critique — independent critic, always available

A second model with different training data reviews the same plan. Different model, different blind spots, different failure modes. Two models see different slices of the same problem space — what one misses, the other catches.

### 3. Mechanical gates — the model can't override

Verification runs as a 7-gate ordered pipeline: mechanical checks (bash one-liners with exit codes) → constraint gate → invariant gate → build/test/lint → acceptance criteria → output persistence → assess gate. Fail-closed — a single non-zero exit blocks the entire verification, regardless of what the model claims.

The mechanical checks are the binding answer. No model-based judgment can override a grep that returns non-zero. This matters because models reliably exploit OR clauses in constraint language: a constraint that says "wrap in try/catch *or* guard in callers" will pass model review with neither branch actually satisfied — the model finds the loophole, reasons compliance from it, and moves on. Mechanical checks don't have loopholes.

### 4. Institutional memory that compounds

Every shipped feature produces:
- An **ADR** (architecture decision record) with decision drivers, alternatives rejected with reasons, and measurable validation signals
- A **calibration trace** classifying findings into three buckets: portable (applies regardless of model/harness), model-specific (tied to current model behavior), harness-specific (tied to current tooling version)

When you swap models or harnesses, the portable findings survive. The model-specific ones are flagged for re-validation. This prevents overfitting to any one model version and makes the methodology measurably better with each feature that ships.

---

## Evidence

These claims are backed by observed feature runs. Not speculation.

| Claim | Evidence |
|---|---|
| Perspective rotation produces zero-overlap findings | `dark-mode-toggle`: senior-engineer pass caught Vite CSS injection ordering, `setItem` error handling, `matchMedia` cleanup, step-ordering inversion. PM pass caught UI specification gaps, accessibility requirements, cross-tab sync scope, browser/OS matrix. **10 findings, zero overlap across two passes.** |
| Cross-model catches unique gaps | `posts-tags-filter`: DeepSeek V4 Pro Think High as cross-model critic produced **3 unique catches** that K2.6 perspective rotation alone did not surface (junction-key delimiter ambiguity, OR-clause in constraint language, RouteErrorBoundary technical inaccuracy). |
| Model-based verification rubber-stamps OR clauses | `dark-mode-toggle` Step 2: a try/catch constraint with an "or in callers" escape hatch was rubber-stamped by 5 independent review layers (`/critique`, `/decompose` consistency sweep, `/implement` constraint contract, `/verify` constraint gate, and the engineer's own read-through). All five found the loophole and reasoned compliance — none caught that neither branch was actually satisfied. Mechanical checks surfaced it immediately. |
| K2.6 thinking mode sustains non-trivial chunks | `dark-mode-toggle`: sustained 39 tool calls (Step 6 across preflight+implement+verify) and 50+ tool calls (Step 4) without repeating work, inventing APIs, or refactoring earlier code. Drift threshold is empirically higher than what chunk-size caps assume. |

---

## Where it fits

helm is not a tool. It's a **process specification** — a set of phase contracts, prompt templates, and verification rules that any coding-agent harness can implement. It sits between the model and the repo:

```
Model output → [helm verification layer] → production merge
                   ↑
              Evidence artifacts
           (ADRs, calibration traces)
```

It does not replace CI/CD, code review, or existing QA gates. It adds a **pre-merge governance check** specifically designed for the failure modes of LLM-generated code: confident hallucination, constraint-exploitation, silent drift, and compounding errors across tool calls.

---

## What it doesn't solve

A methodology is honest about its limits.

| Risk | How helm handles it | What escapes |
|---|---|---|
| Novel failure modes the model hasn't seen before | Cannot catch categorically. Mitigated by cross-model critique (different training data, lower chance of shared blind spot). | A genuinely new class of LLM error will pass until the methodology is updated to detect it. |
| Compounding errors across phases | Per-phase evidence requirements + cross-step consistency contract catch contradictions between artifacts. | Sequential phases that all share the same underlying assumption (and no phase challenges it) will propagate the error. |
| Model-swap disruption | Calibration buckets classify findings by portability. Portable findings survive swaps; model-specific ones are flagged for re-validation. | A model with fundamentally different failure modes (e.g., self-hosted fine-tune vs. API) requires new calibration data. No prior artifact can predict this. |
| Token cost bloat | Multiple critique passes + cross-model add cost. Mitigation: prompt caching achieves ~83% input discount on stable prefix. | Total cost per feature is higher than single-pass agent usage. The trade is defect prevention vs. compute spend. |
| Setup friction | Requires seeding `AGENTS.md` + prompt templates + initial calibration. One-time cost per project. | New project ramp is manual. No one-click setup yet. |

---

## Why not just use `.cursorrules`?

`.cursorrules`, Copilot instructions, and Claude Code custom instructions are **prompt engineering**. They tell the model what to care about. They don't verify that the model actually did it.

helm adds verification — mechanical checks the model can't argue past, a consistency contract across step docs, and institutional memory that persists after the conversation window closes. The prompt rules are the *intent*. The methodology is the *enforcement*.

---

## Usage

Harness-agnostic. Works with any coding agent that reads files and writes code.

**Reference implementation:** [`pi-orchestrate`](bin/pi-orchestrate) drives the full pipeline — `--model` for primary, `--critique-model` for second model, `--hitl` for human-in-the-loop gates, structured per-phase logging.

**Other harnesses:**

- [keel](https://github.com/devexcelsior/keel) + [hull](https://github.com/devexcelsior/hull) — the open pi stack
- Claude Code, Codex, Cursor, Copilot
- Your own — the prompts are markdown, the checks are bash

---

## Contents

| File | Purpose |
|---|---|
| `AGENTS.md` | Doctrine and workflow specification |
| `prompts/plan.md` | Planning — 6-field reasoning template, branch + worktree setup |
| `prompts/critique.md` | Critique — perspective rotation, discipline-axis check |
| `prompts/revise.md` | Revision — per-point response table, no silent drops |
| `prompts/decompose.md` | Decomposition — self-contained step docs, 7-rule consistency contract |
| `prompts/preflight.md` | Pre-flight — file listing, API verification, import validation |
| `prompts/implement.md` | Implementation — cold-start procedure, constraint contract |
| `prompts/verify.md` | Verification — 7 ordered gates, mechanical-first |
| `prompts/assess.md` | Assessment — per-step anti-flattery review |
| `prompts/distill.md` | Distillation — roadmap → ADR, supersession tracking |
| `prompts/calibrate.md` | Calibration — cross-corpus synthesis, bucket classification |
| `prompts/promote.md` | Promote — doctrine update gate |
| `prompts/finalize.md` | Finalize — rebase, ff-only merge, worktree cleanup |
| `prompts/status.md` | Status — active feature state, step tracking |

---

## License

MIT. See [LICENSE](LICENSE).