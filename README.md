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

## How it differs from Superpowers

| | Superpowers | helm |
|---|---|---|
| **Critique** | Single-pass review | Perspective rotation — senior eng → PM → security/perf. Near-zero overlap between passes. |
| **Cross-model** | None | Second model as independent critic. Different training data, different failure modes. |
| **Verification** | "Check if it's fixed" | 7 ordered gates including mechanical bash checks that can't be talked around. |
| **Constraints** | Model self-polices | Mechanical grep/awk one-liners with exit codes. Model can't override. |
| **Memory** | Nothing durable | ADR distillation with supersession tracking. Institutional memory in files. |
| **Calibration** | None | Per-feature calibration loop. Findings classified into portable/model-specific/harness-specific buckets. |
| **Foundation** | Trust the platform | MPL-2.0 harness — can't be removed. Build whatever you want on top. |

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
