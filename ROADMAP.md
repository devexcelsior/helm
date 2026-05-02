# Roadmap

What helm ships next, and why.

---

## Now — seed the foundation

**Goal:** The core methodology is complete, reference-able, and backed by evidence from real features.

| What | Status | Evidence |
|---|---|---|
| `AGENTS.md` — doctrine and workflow specification | ✅ Done | |
| `prompts/` — 13 phase prompt templates | ✅ Done | |
| `LICENSE` — MIT | ✅ Done | |
| `README.md` — methodology overview for platform leadership | ✅ Done | `b432975` |
| `.gitignore` — local artifact hygiene | ✅ Done | `0ff325c` |
| `pi-orchestrate` — reference harness implementation | ✅ Done | `bin/pi-orchestrate` |
| Evidence corpus — 3+ features with full trace artifacts | ✅ Done | `dark-mode-toggle`, `rtk-normalized-posts`, `posts-tags-filter` |
| `ROADMAP.md` — this file | ✅ Done | |

---

## Next — hardening

**Goal:** A team of 5+ engineers can use helm with their existing toolchain and adapt it to their org's standards.

### Observability

| What | Why |
|---|---|
| Per-phase pass/fail rates across features | Throughput data to justify continued investment. Without it, it's faith-based. |
| Per-feature token spend (input + output, cached vs. uncached) | Cost forecasting. Can't budget without per-feature spend data. |
| Structured telemetry export (JSON log, Prometheus metrics, or similar) | Pipe agent performance into existing monitoring stacks. |
| Median defect escape count per feature (from calibration traces) | Quality trend. Shows whether the methodology is improving or plateauing. |

### Team model

| What | Why |
|---|---|
| Per-team AGENTS.md overrides | Different teams have different constraints. The methodology can't be one-size-fits-all at scale. |
| Per-team calibration buckets | Calibration findings are team-contextual. A finding from team A's React codebase shouldn't silently become doctrine for team B's Python backend. |
| Cross-team promote gate | When a finding is genuinely portable across teams, there must be a gate to promote it to org-level doctrine. |
| Team-scoped ADR namespacing | ADRs need team ownership so an architect can trace which team made which decision. |

### Integration

| What | Why |
|---|---|
| GitHub Issues/Labels status sync | `.pi/active.md` is local state. Project management needs feature status in the tool the PMO uses. Minimum viable: a label-based status mapping (e.g., `helm:in-progress`, `helm:verified`). |
| Jira/Linear webhook targets | Stretch. Many orgs run on Jira. |
| CI/CD pipeline hook | `pi-orchestrate` currently runs interactively. Needs to be runnable in CI with a pass/fail exit code. |
| Code review integration | How does helm verification output feed into a GitHub PR review? Can mechanical gate results surface as PR checks? |

### Customization

| What | Why |
|---|---|
| Org-specific mechanical check injection | A team must be able to add "must use `@adobe/design-system` components, not raw HTML" as a mechanical check in `/verify`. |
| Custom critique perspectives | Teams may need domain-specific perspectives beyond PM/senior-engineer/security-perf (e.g., "accessibility reviewer," "i18n reviewer"). |
| Per-project testing standards overrides | Coverage thresholds, mutation score targets, required test types — these vary across orgs. |

### Model management

| What | Why |
|---|---|
| Model evaluation matrix | Documented performance of each available model on each phase, with evidence from real features. Supports model selection with data, not hunches. |
| Multi-model failover strategy | What happens when the primary model's provider is down? Can the harness hot-swap to a fallback without losing in-flight state? |
| Breaking-change migration strategy | When AGENTS.md updates, in-flight features on the old doctrine need a documented path: rebase to new doctrine, or complete under old and calibrate the delta. |

### Testing & quality

| What | Why |
|---|---|
| Coverage threshold guidance | "No snapshot tests" and "prefer findByRole" are style rules. Measurable quality gates beyond style. |
| Negative-permutation validation tooling | The doctrine requires tests to fail against deliberately-broken permutations. This is currently manual. A script that automates permutation testing would close the gap between doctrine and practice. |
| Property-based / fuzz testing guidance | Stretch. For pure-logic reducers and selectors, property-based tests catch edge cases unit tests miss. |

---

## Later — cross-team

**Goal:** The methodology works across multiple teams with auditability and predictable performance.

### Compliance

| What | Why |
|---|---|
| Auditable ADR chain | Each ADR links to the verification output that proves it was satisfied. External auditors can trace a production defect → ADR → verification gate → mechanical check that passed or failed. |
| Phase sign-off artifacts | Each phase produces a signed artifact (commit hash + timestamp + model identity). In a SOC2 audit, you prove which model reviewed which code and when. |
| Attestation API | Programmatic access to "was feature X verified by helm?" for compliance automation. |

### Predictability

| What | Why |
|---|---|
| Time-to-ship targets (p50, p95, p99 per feature complexity tier) | Predictable delivery windows. Needs data from 20+ features to establish. |
| Verification gate pass-rate targets | How often does `/verify` clear on first run vs. requiring re-implementation? A low first-pass rate signals methodology gaps. |
| Defect escape rate target | What percentage of calibration findings represent defects that would have shipped without the methodology? This is the core effectiveness metric. |

### Incident loop

| What | Why |
|---|---|
| Incident → calibration trace formal path | When a production defect escapes helm, the post-mortem must feed into the calibration corpus and trigger promote review. Currently this is ad-hoc. |
| Regression test injection from incidents | Each incident should produce at minimum one new mechanical check or test that would have caught it. |

### Aggregation

| What | Why |
|---|---|
| Cross-team dashboard | Single view: all features in flight across teams, per-phase status, aggregate pass rates, aggregate token spend. |
| Methodology version tracking | When the org-level AGENTS.md updates (via promote), which teams are on which version? What's the upgrade lag? |

---

## Deferred / out of scope (for now)

| What | Why deferred |
|---|---|
| Multi-language support (Python, Go) | The evidence corpus is TypeScript + React. Not a priority for current scope. |
| E2E test integration in the harness | Per doctrine: deferred until manual smoke test surface exceeds ~30 min per ship. Revisit at 5 features shipped. |
| IDE plugin / editor integration | The methodology is harness-agnostic by design. An IDE plugin ties it to a specific editor. Revisit if it becomes the obvious next step. |
| Fine-tuned model for methodology-specific reasoning | Depends on LoRA fine-tune of K2.6 or equivalent. Track if reasoning-template scaffolding proves insufficient. |
| Real-time collaboration / multi-agent | Helm is single-agent-per-feature by design. Multi-agent coordination is a different problem space. |

---