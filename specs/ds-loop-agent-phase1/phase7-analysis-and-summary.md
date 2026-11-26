# Phase 7: Post-Modelling Analysis and Model Summary (Stages 4–5)

**Depends on:** `phase6-training-tuning-and-gate.md`
**Status:** Not started

## Design components touched

Post-modelling analysis in the check runner (§5.5; D46), red-flag checks and reopening (D30, D46), model summary (§5.6; PRD §6.2), datasets and answer keys (§11).

## Objective

Stage 4 explains the approved, frozen model and finds where it fails:
- **The platform computes every number** in the check runner: per-segment metrics, SHAP importance and dependence, calibration or residuals by segment, and importance stability across folds.
- **The agent** chooses extra cuts and interprets the results in `analysis.md`.
- **Red-flag checks** catch leaks and segment failures, and the user can reopen stage 2 or 3 with the findings.

Stage 5 writes the model card, with every number traced to the logs and the analysis. A subtle planted leak, designed to get past stage 2, is caught here.

## Independently testable when

- On Ames, with the subtle leak present, stage 4 raises a red flag ("an ID/forbidden-like column is the top feature" or "one dominant feature"). "Reopen stage 2 with these findings" marks stages 3–5 stale.
- The held-out counter is still 1 after stage 4.
- Stage 5's model card passes the "every number matches" check, and fails it when a number is edited.

## Tasks

### 7.1 Design the subtle planted leak (HITL)

- **Type:** HITL
- **Blocked by:** 3.1
- **PRD reference:** §11.2 (subtle planted leak, caught in stage 4)
- **System design reference:** §11 (deferred: exact design per dataset)

**Scope:** Design, for Ames now and Telco in 9.2, a leak that:
1. passes stage 2's checks (target-association threshold, forbidden columns, time leakage);
2. clearly dominates SHAP importance in stage 4.

Add it to each answer key with its expected catching rule. Not included: implementing the analysis (7.2).

**Acceptance criteria:**
- [ ] With the leak present, stage 2's checks pass (verified by running them).
- [ ] The answer key documents the leak and the expected red flag.
- [ ] Verifiable in isolation.

**If HITL:** **Decision needed:** which proxy column to plant and how strong to make it, so that it's subtle enough to get past stage 2 yet obvious in SHAP and honest for an audience. **Who decides:** the project owner, as demo presenter. Neither the PRD nor the system design settles this; both mark it deferred.

### 7.2 Analysis computed by the platform; `analysis.md`

- **Type:** AFK
- **Blocked by:** 6.5
- **PRD reference:** §6.2 (stage 4)
- **System design reference:** §5.5; D46

**Scope:**
- **Agent and skill:** a `post-modelling-analysis-agent` in chat mode, with the `ds-analysis` skill (which analyses to request per task type, how to read SHAP and segments, red-flag patterns).
- **Platform computation, in the check runner:**
  - per-segment metrics for the spec's segments, with held-out results computed **only from the saved gate predictions**;
  - SHAP global importance and dependence on the training data's CV folds (add SHAP to the check-runner image);
  - calibration (classification) or residuals (regression) by segment;
  - importance stability across folds.
- **Extra cuts:** the agent requests them through its output file; the platform computes them.
- **Write-up:** the agent writes `analysis.md` interpreting the results. Tables and plots appear in the file browser.

**Acceptance criteria:**
- [ ] No held-out rows are read: the analysis runs don't mount the held-out set, and the counter stays at 1.
- [ ] The agent sees only aggregates (test: no row-level held-out data in its workspace).
- [ ] Demoable in isolation.

### 7.3 Red flags, the "numbers match" check, and reopening with findings

- **Type:** AFK
- **Blocked by:** 7.1, 7.2, 4.7
- **PRD reference:** §6.2 (post-modelling analysis rules)
- **System design reference:** §5.5; D30, D46

**Scope:**
- **Checks:** every spec segment is covered; every number in `analysis.md` matches a computed result.
- **Red flags:** an ID or forbidden column among the top features; one dominant feature; a segment beyond tolerance; importances unstable across folds.
- **Actions:** "Reopen stage 2 / stage 3 with these findings" triggers the stale cascade and carries the findings into the reopened stage's brief.

**Acceptance criteria:**
- [ ] The subtle Ames leak from 7.1 raises a red flag.
- [ ] Editing a number in `analysis.md` fails the "numbers match" check.
- [ ] Demoable in isolation.

### 7.4 Model summary (stage 5)

- **Type:** AFK
- **Blocked by:** 7.3
- **PRD reference:** §6.2 (stage 5), §14
- **System design reference:** §5.6

**Scope:** The `model-summary-agent` with the `ds-model-summary` skill (model-card template; every number from the log or the analysis) writes `model_summary.md`. Checks: every number matches the logged metrics and analysis results; the limitations and the stage 4 red flags are present.

**Acceptance criteria:**
- [ ] A fabricated or altered number fails the check.
- [ ] Omitting a stage 4 red flag fails the check.
- [ ] Demoable in isolation.
