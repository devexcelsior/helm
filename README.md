# helm

How to sail. A rigorous methodology for coding agents.

---

## Architecture

```
helm (MIT)         ← this repo — methodology, prompts, orchestration
hull (MIT)         ← TUI, web UI, plugins, CLI
keel (MPL-2.0)     ← agent engine + LLM API
```

helm is the methodology layer. It specifies how coding agents plan, critique, implement, verify, and learn. Runs on any harness that reads files and writes code. The reference implementation is [keel](https://github.com/devexcelsior/keel) + [hull](https://github.com/devexcelsior/hull).

---

## The problem

Coding agents produce plausible-looking code that passes obvious checks and fails on edge cases, state management, or cross-cutting concerns no single reviewer catches. The model argues past reviewers better than any junior engineer. Stopping the agent to re-read output defeats the speed gain.

---

## Four things that make it different

### Perspective-rotation critique

Every plan gets two independent reviews from different perspectives (PM requirements, senior-engineer risk). Same model, different role. Empirically: near-zero overlap between passes.

### Cross-model critique

A second model with different training data reviews the same plan. Different model, different blind spots. What one misses, the other catches.

### Mechanical gates

Verification runs as a 7-gate pipeline: mechanical checks (bash one-liners with exit codes) → constraint gate → invariant gate → build/test/lint → acceptance criteria → output persistence → assess gate. Fail-closed. A non-zero exit blocks everything.

Mechanical checks are the binding answer. Models exploit OR clauses in constraints — a rule saying "wrap in try/catch *or* guard in callers" passes model review with neither satisfied. grep doesn't have loopholes.

### Institutional memory

Every feature produces an ADR with decision drivers and validation signals, plus a calibration trace classifying findings as portable, model-specific, or harness-specific. When you swap models, portable findings survive. Model-specific ones flag for re-validation. The methodology gets better with each feature.

---

## Evidence

| Claim | Source |
|---|---|
| Perspective rotation: zero overlap | `dark-mode-toggle`: 10 findings across two passes, zero overlap |
| Cross-model: unique catches | `posts-tags-filter`: 3 unique catches K2.6 perspective rotation missed |
| OR-clause exploitation caught | `dark-mode-toggle` Step 2: 5 review layers rubber-stamped a constraint neither branch satisfied. Mechanical check caught it. |
| Sustained agent performance | `dark-mode-toggle`: 39 tool calls (Step 6), 50+ (Step 4) — no drift, no invented APIs |

---

## Reference implementation

[`pi-orchestrate`](bin/pi-orchestrate) drives the full pipeline — `--model` for primary, `--critique-model` for second, `--hitl` for human-in-the-loop, structured per-phase logging. A single bash command.

---

## What it doesn't solve

| Risk | Mitigation | What escapes |
|---|---|---|
| Novel failure modes | Cross-model critique (different training data) | Truly new error classes pass until detected |
| Compounding phase errors | Cross-step consistency contract | Shared assumptions propagate if unchallenged |
| Model-swap disruption | Calibration buckets survive portable findings | New models need new calibration data |
| Token cost | Prompt caching for input discount | Higher total cost than single-pass usage |
| Setup friction | One-time seed per project | No one-click setup yet |

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