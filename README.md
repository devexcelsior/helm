# helm

How to sail. A rigorous methodology for coding agents.

## What

helm is a multi-pass coding methodology that guides AI coding agents through a disciplined workflow. It's harness-agnostic — works with any coding agent that reads files and writes code.

## The workflow

```
/plan → /critique (PM) → /critique (senior eng) → /revise
→ /decompose → /preflight → /implement → /verify → /assess
→ /distill → /calibrate → /finalize
```

## Features

- **Perspective-rotation critique** — senior engineer, product manager, and security/perf reviewers rotate on every plan. Same model, different role. Near-zero overlap between passes.
- **Cross-model critique** — a second model (different training data, different failure modes) reviews independently when available.
- **Mechanical verification** — bash one-liners with exit codes. grep, awk, ripgrep. The model can't talk around a failing check.
- **7-gate verification** — ordered: mechanical checks → constraint gate → invariant gate → build/test/lint → acceptance criteria. Fail-closed.
- **ADR distillation** — every shipped feature leaves a decision record with supersession tracking. Institutional memory lives in files, not model weights.
- **Calibration buckets** — per-feature findings classified as portable, model-specific, or harness-specific. Improves the methodology itself, not just the code.

## Contents

| File | Purpose |
|---|---|
| `AGENTS.md` | Doctrine and workflow specification. The constitution. |
| `prompts/plan.md` | Planning phase — 6-field reasoning template, branch + worktree setup |
| `prompts/critique.md` | Critique phase — perspective rotation, discipline-axis check |
| `prompts/revise.md` | Revision phase — response table, no silent drops |
| `prompts/decompose.md` | Step decomposition — self-contained step docs, 7-rule consistency contract |
| `prompts/preflight.md` | Pre-flight — file listing, API verification, import validation |
| `prompts/implement.md` | Implementation — cold-start procedure, constraint contract, invariants check |
| `prompts/verify.md` | Verification — 7 ordered gates, mechanical checks first |
| `prompts/assess.md` | Assessment — per-step anti-flattery review |
| `prompts/distill.md` | Distillation — roadmap → ADR, supersession tracking |
| `prompts/calibrate.md` | Calibration — cross-corpus synthesis, bucket classification |
| `prompts/finalize.md` | Finalize — rebase, ff-only merge, worktree cleanup |
| `prompts/status.md` | Status — active feature state, step tracking |

## Usage

This methodology is harness-agnostic. Use it with:

- [keel](https://github.com/devexcelsior/keel) + [hull](https://github.com/devexcelsior/hull) — the open pi stack
- Claude Code, Codex, Cursor, Copilot — any agent that reads files
- Your own custom harness — the prompts are markdown, the checks are bash

## License

MIT. See [LICENSE](LICENSE).
