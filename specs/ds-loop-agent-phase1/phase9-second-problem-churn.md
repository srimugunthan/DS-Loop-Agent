# Phase 9: Second Problem — Customer Churn

**Depends on:** `phase7-analysis-and-summary.md`, `phase8-mlflow-and-export.md`
**Status:** Not started

## Design components touched

Problem spec as the only source of dataset details (§3 ProblemSpec; D11), problem scope (D12), demo datasets (D13), the classification variants of every check (PRD §6 check table).

## Objective

Prove the tool is problem-agnostic:
- Add the classification variants of every check.
- Prepare the IBM Telco Churn dataset with natural and planted defects and an answer key.
- Run the full workflow on churn, in both modes, with **no changes** to agent prompts, skills, the loop runner or the UI.

Any change needed exposes something hardcoded. It is fixed in the shared code and recorded.

## Independently testable when

- Churn runs end to end, from the Project step through stage 5, using chat and loop modes; every planted defect and leak in the churn answer key is caught.
- A diff of prompts, skills, loop runner and UI between the Ames and churn runs shows no problem-specific changes.

## Tasks

### 9.1 Classification variants of every check

- **Type:** AFK
- **Blocked by:** 7.4
- **PRD reference:** §6 (check table: classification column)
- **System design reference:** D11, D12

**Scope:** All variants are selected purely from `problem.yaml`:
- primary metric ROC-AUC, or PR-AUC when the positive class is rare;
- a majority-class / base-rate baseline;
- a single-feature AUC leak test;
- stratified folds, or time-ordered if a time column is declared;
- class balance and calibration;
- the decision threshold chosen on out-of-fold predictions, never held-out data;
- per-segment recall;
- calibration by segment in stage 4.

**Acceptance criteria:**
- [ ] Unit tests for each variant on a synthetic classification dataset.
- [ ] No check branches on a dataset name or column name (grep test).
- [ ] Verifiable in isolation.

### 9.2 Telco churn dataset and answer key

- **Type:** AFK
- **Blocked by:** 7.1
- **PRD reference:** §11.2
- **System design reference:** D13; §11

**Scope:**
- **Natural defect:** verify that `TotalCharges` is text with blanks for new customers.
- **Planted defects:** duplicate customers and inconsistent labels.
- **Planted leaks:** a post-churn column (e.g. a cancellation reason) for stage 2, and the subtle stage-4 leak designed in 7.1.
- **Answer key** and a reproducible build script.

**Acceptance criteria:**
- [ ] The answer key covers every defect and leak with its expected catching check.
- [ ] The dataset is rebuilt from the public source by a script.
- [ ] Verifiable in isolation.

### 9.3 Generality run on churn

- **Type:** AFK
- **Blocked by:** 9.1, 9.2, 8.3
- **PRD reference:** §13 (P1-M3), §15 (generality)
- **System design reference:** D11

**Scope:**
1. Create a churn project; the master agent drafts a stratified spec.
2. Run stages 1–5, using loop mode for at least stages 2 and 3.
3. Approve, then export.
4. Record any change that was needed outside `problem.yaml` and the dataset, fix it in the shared, problem-agnostic code, and re-run Ames to confirm nothing regressed.

**Acceptance criteria:**
- [ ] All five stages approved on churn using both modes where available; every answer-key item is caught.
- [ ] Ames still passes end to end after any generalisation fix.
- [ ] Demoable in isolation.
