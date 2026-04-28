# CHECK_REPO_report_review — review of audit recommendations

> Date: 2026-04-12
> Source report: `docs/ai-first-approaches/check_repo_methodics/CHECK_REPO_report.md`
> Goal: decide which recommendations are worth implementing now as repo improvements, which should be deferred, and where the original estimates should be adjusted.

---

## Short answer

- The report is directionally useful, but several recommendations are too optimistic in timing and mix three different things: repo hygiene, architecture enforcement, and product/ops/business logic.
- The highest-ROI repo improvements right now are: `E1`, `E2`, `C3`, `C6`, `G1`, and a **minimal** version of `G3`.
- The biggest correction is around CI: **fast CI now, full CI later**. The current implementation plan already treats full CI as task `5.1` and marks it `blocked`, but its own checklist explicitly allows a stage-1 fast CI first.
- `A4` and `D1` are overstated in the report. The repo already has a usable decision-log-lite in `docs/ARCHITECTURE/ARCHITECTURE_V6.1_index.md`; mandatory `docs/adr/`, runbooks, post-mortems, and `CHANGELOG` are not justified yet.
- Items marked by the заказчик as business logic / process logic generally should indeed stay out of the current repo-improvement wave. The only exception is where they affect core repo safety or retrieval quality.

---

## Main decisions

### Do now

| Item | Decision | Adjusted usefulness | Adjusted complexity | Adjusted KPI | Notes |
|------|----------|---------------------|---------------------|--------------|-------|
| `E1` | Do now | 8 | 1 | +7 | Add one entrypoint such as `make verify` or equivalent |
| `E2` | Do now, together with `E1` | 8 | 1 | +7 | Fast local loop should be part of the same task, not a separate epic |
| `C3` | Do now | 8 | 1 | +7 | Root `README.md` is stale and points to V5/V2 |
| `C6` | Do now | 8 | 1 | +7 | SSOT drift is real and broader than the report shows |
| `G1` | Do now, minimally | 8 | 2 | +6 | Short secrets policy + scanner with baseline/allowlist |
| `G3` (fast CI only) | Do now | 9 | 3 | +6 | Add only stage-1 CI: lint + typecheck + non-integration tests |

### Do later

| Item | Decision | Adjusted usefulness | Adjusted complexity | Adjusted KPI | Notes |
|------|----------|---------------------|---------------------|--------------|-------|
| `G4` | Do later, or minimally now | 7 | 4 | +3 | Pre-commit is useful, but should start small |
| `G5 / A5 / E5` | Do later | 9 | 6 | +3 | Important, but should follow boundary cleanup tasks `R.1-R.3` |
| `B9` | Do later, non-blocking first | 5 | 4 | +1 | Start with reporting/metrics, not a blocking gate |
| `A4` | Do later if decision volume grows | 4 | 4 | 0 | Current architecture index already works as a decision log-lite |
| `C8` | Do later, but change the recommendation | 4 | 2 | +2 | Problem is drift control, not necessarily duplicate file removal |
| `C7` | Do later | 4 | 4 | 0 | Link checker/stale-doc checks help later, but not before basic SSOT cleanup |
| `E3` | Do later | 3 | 3 | 0 | Useful devex, but low ROI compared with verify/CI/SSOT |
| `G2` | Do later, selectively | 3 | 2 | +1 | Avoid comment noise; use only for truly dangerous paths |
| `D2` | Later, user-owned | 3 | 2 | +1 | Cleanup value is real, but severity in the report is overstated |

### Do not treat as current repo-improvement work

| Item | Decision | Adjusted usefulness | Adjusted complexity | Adjusted KPI | Notes |
|------|----------|---------------------|---------------------|--------------|-------|
| `D1` | Not as a bundled task now | 3 | 5 | -2 | Split into architectural memory vs operational memory |
| `E4 / F5` | Not now | 2 | 5 | -3 | This is mainly ops/business/process logic at the current stage |
| `F6` | Not now | 2 | 3 | -1 | Formal post-mortem loop makes sense after real incidents/team scaling |
| `E7` | Not now | 2 | 6 | -4 | This is product/eval logic tied to task `2.3.1`, not repo hygiene |

---

## CI-specific answer

### What is worth implementing now

The report is right that CI is a major gap, but it proposes the wrong timing for the full version.

Current repo facts:

- There is no root `.github/workflows/`.
- `pyproject.toml` already has `ruff`, strict `mypy`, and `pytest` config.
- `docs/IMPLEMENTATION/05_release_and_product.md` marks `5.1 - CI pipeline` as `blocked`, but its own checklist explicitly starts with `stage 1 fast CI: ruff + mypy + pytest -m "not integration"`.
- `docs/TESTING_STRATEGY.md` already defines a 3-stage target pipeline: fast tests, integration, then golden dataset / eval.

### Recommended split

#### Phase 1 — now

- Add a single local entrypoint: `make verify` or equivalent.
- Add fast CI only:
  - `ruff check .`
  - `mypy .`
  - `pytest tests/ -m "not integration"`
- If architecture docs are changed, also run `python3 scripts/architecture_docs.py validate`.

This is aligned with the implementation plan and gives immediate governance value without waiting for full product maturity.

#### Phase 2 — later, when full CI becomes real task `5.1`

- Integration tests with Docker/services.
- Contract suites that depend on delivery/runtime stability.
- Required branch protection tied to stable required checks.
- Broader architecture enforcement (`import-linter`, stricter contract gates).

#### Phase 3 — release/manual gate

- Golden dataset regression.
- `scripts/eval_brain.py` as a manual or release gate.

### Why not do full CI immediately

Because that would couple current repo hygiene work to still-blocked product/runtime tasks. This is exactly what `docs/IMPLEMENTATION/05_release_and_product.md` already warns about.

### Score correction for CI items

- `G3` should be split conceptually into:
  - `G3a fast CI now`: usefulness `9`, complexity `3`, KPI `+6`
  - `G3b full release CI later`: usefulness `8`, complexity `8`, KPI `0`
- `G4` should be reduced from `+5` to roughly `+3` unless kept minimal.
- `G5` remains important, but it is not a quick win; current complexity is understated.

---

## Answers for A4 and D1

### A4 — ADR / decision logs

The question in the report comment is correct: **`docs/adr/` is not automatically better than the current setup.**

What the repo already has:

- `docs/ARCHITECTURE/ARCHITECTURE_V6.1_index.md` contains a structured `Review Response` block with:
  - implemented decisions,
  - known MVP limitations,
  - explicit skipped items with reasoning,
  - deferred roadmap.

This is already a usable decision log-lite.

### When a separate ADR structure becomes better

Separate ADR files become worthwhile when all three are true:

1. there are many independent decisions;
2. decisions need `superseded by` chains and stable IDs;
3. PRs/tests/docs need to reference decisions individually.

That threshold does not seem to be reached yet.

### Better recommendation than the original report

- Do **not** force-create `docs/adr/` now.
- If decision history starts growing, prefer one of these:
  - `docs/ARCHITECTURE/decision_log.md`
  - `docs/ARCHITECTURE/decisions/`

That fits the current SSOT much better than a separate top-level `docs/adr/`.

### D1 — sustainable knowledge

The report bundles together too many different artifact types.

Current reality:

- architecture knowledge is already strong;
- implementation status and execution memory are already strong;
- what is missing is mostly **formalization format**, not knowledge itself.

So `D1` should be split conceptually:

- `D1a architectural decision memory` — partially present already;
- `D1b operational memory` (`runbooks`, `post-mortem`, `CHANGELOG`) — mostly not needed yet as repo-improvement work.

### Is `CHANGELOG` needed if `docs/IMPLEMENTATION/00_status.md` exists?

Right now: **no, not really**.

Why:

- `docs/IMPLEMENTATION/00_status.md` is not a release changelog, but together with `docs/IMPLEMENTATION/*` and git history it already covers internal execution tracking well enough.
- A real `CHANGELOG` becomes useful when there are stable releases, external consumers, or deployment cadence that requires answering “what changed between version N and N+1”.

So the better recommendation is:

- no dedicated `CHANGELOG` yet;
- revisit after `5.1-5.3` when release flow actually exists.

---

## Review of items with explicit customer comments

### `D2` — scratchpad / old docs / TODO files

The report overstates this one.

Important nuance:

- `docs/TODO_now.md` and `docs/TODO_FIT_REVIEW_2_1.md` are already explicitly frozen and redirected to `docs/IMPLEMENTATION/`.
- `docs/old/` is at least partially intentional archive/pointer-mode history according to `docs/IMPLEMENTATION/00_governance_and_drift_cleanup.md`.
- `.planning/codebase/*` is not just scratchpad; it is currently listed in `AGENTS.md` / `CLAUDE.md` under `Read first`, so deleting it blindly could hurt agent onboarding.

What is still important:

- `docs/oldold/` looks much noisier than `docs/old/` and is a genuine retrieval-noise candidate.
- `.planning/codebase/*` is useful only if refreshed; otherwise it becomes stale context, not helpful context.

Conclusion:

- Do not auto-delete now.
- Keep this as a user-owned cleanup task.
- If anything gets promoted in priority here, it is **quarantining or clearly freezing `docs/oldold/`**, not deleting `docs/old/` wholesale.

### `C8` — `AGENTS.md` / `CLAUDE.md` duplication

The report finds a real issue, but recommends the wrong fix.

What is true:

- `AGENTS.md` and `CLAUDE.md` are near-identical.
- The repo also has `.planning/codebase/*`, creating a third context layer.

What should change in the recommendation:

- If two files are intentionally needed for different agents, keep them.
- Do not switch to a symlink unless agent-loader behavior is known to be safe.
- The real fix is to reduce drift risk:
  - either add a sync check,
  - or generate one file from the other,
  - or make one clearly declared mirror of the other.

Also note that the more urgent drift problem is not this duplication alone, but stale references inside `.planning/codebase/*`, `README.md`, `docs/AI_PRACTICES.md`, and `components/brain/README.md`.

### `E3` — setup script

This is not business logic, but it is also not an urgent repo-improvement task.

Best positioning:

- low-priority devex/onboarding task;
- revisit after verify and fast CI are in place.

### `E4 / F5` — runbooks / incident playbooks

The customer comment is basically correct for the current phase.

At this stage, most of these recommendations are about operating model and future incident handling, not about making the repository immediately more AI-friendly for code changes.

Possible narrow exception later:

- one minimal troubleshooting note for local Stoneforge/runtime startup could be justified if onboarding friction becomes recurrent.

But the full recommendation in the report is too early.

### `E7` — regression gate on golden data

The customer comment is also correct here.

This is not general repo hygiene. It belongs to Brain quality/product verification and is already represented by task `2.3.1` plus the future stage-3 CI/release flow.

---

## Corrections to the original report itself

### 1. `A6` evidence contradicts `G4`

The report says:

- `A6`: zero cross-component imports, grep-verified
- `G4`: there is a real cross-component import violation in `components/backend/backend/rest/admin.py`

`G4` is correct here. The `A6` evidence text should be softened or corrected.

Recommended correction:

- keep `A6` as strong-but-not-perfect, or lower it to `1/2` if interpreted strictly;
- at minimum remove the claim of “zero cross-component imports”.

### 2. `D2` score is probably too low

Because frozen legacy docs with pointer mode are already in place, `D2` looks closer to `1/2` than `0/2`.

### 3. `C6` problem is broader than the report says

The report mentions `README.md` and `docs/AI_PRACTICES.md`, but the actual drift also affects at least:

- `components/brain/README.md`
- `.planning/codebase/CONCERNS.md`
- `.planning/codebase/STACK.md`

This makes `C6` even more worth doing now.

### 4. `E5 / A5` should acknowledge existing partial architecture checks

The repo already has partial architecture/contract checks, for example:

- `tests/test_architecture_docs.py`
- `tests/components/test_materializer_syncer_contract.py`

So the gap is real, but the wording should be “partial and not yet boundary-enforcing”, not “almost absent”.

### 5. `G3` should not assert branch protection absence too strongly from local evidence alone

What is definitely visible from the repo is:

- there is no root CI workflow;
- therefore there are no repo-defined required checks yet.

That is enough to keep `G3` at `0/2`, but the wording should be closer to “branch protection is not evidenced here” unless checked directly in GitHub settings.

---

## What I would actually put into the implementation plan

### Wave 1 — immediate repo improvements

1. `E1 + E2`: one local verify entrypoint.
2. `C3 + C6`: fix stale SSOT links and stale top-level guidance.
3. `G1`: minimal secrets policy + scanner setup.
4. `G3a`: fast CI only.

### Wave 2 — after that

5. `G4`: small pre-commit profile (`ruff`, maybe formatting, `detect-secrets`).
6. Refresh or demote `.planning/codebase/*` from `Read first` if it stays stale.
7. Optional: drift-control mechanism for `AGENTS.md` / `CLAUDE.md`.

### Wave 3 — after boundary cleanup / later product maturity

8. `G5 / A5 / E5`: import-linter or equivalent boundary enforcement, after tasks `R.1-R.3`.
9. `B9`: complexity thresholds as reporting first, gating later.
10. `A4`: only if decision volume outgrows the current architecture index.
11. `E7`: only when `2.3.1` and release-grade eval workflow are ready.

---

## Final position

- The report correctly identifies the weakest areas: governance, verify wiring, and some memory/doc hygiene.
- The main thing to change is **timing and scope**:
  - do the lightweight repo improvements now;
  - do not bundle them with full CI, runbooks, post-mortems, or product eval gates yet.
- The most valuable immediate changes are not `docs/adr/` or runbooks, but **one verify entrypoint, fixed SSOT references, minimal secrets policy, and fast CI**.
