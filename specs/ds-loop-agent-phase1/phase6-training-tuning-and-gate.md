# Phase 6: Stage 3 — Training, Tuning and the Held-Out Gate

**Depends on:** `phase5-loop-engine.md`
**Status:** Not started

## Design components touched

Stage 3 structure and generated `train.py` (ADR-005; D29, D38), Optuna in the check runner (§5.4; D22, D47), simplicity rule and fixed folds (D8), Pipeline files (D25, D37), held-out gate, counter and gate predictions (§5.4; ADR-003, D5, D8, D46).

## Objective

Stage 3 finds the right model:
- In chat mode, the agent edits a model config and trains in the check runner.
- In loop mode, each iteration proposes one **search plan** that Optuna runs on fixed CV folds:
  - **explore** compares model families with small budgets, and the one-SE rule picks a champion;
  - **refine** spends a bigger budget on the champion only.
- Kept experiments and the champion are saved as one scikit-learn `Pipeline`.
- When the loop stops, the **held-out gate** runs exactly once. It scores raw held-out rows with the saved Pipeline, increments the counter, and saves gate predictions for stage 4.

## Independently testable when

- On Ames, a stage 3 loop runs explore then refine; the leaderboard, chart and timeline show the switch; the one-SE rule keeps a simpler champion when gains are within the fold spread.
- The gate runs once (counter = 1), the stage is approved, and the gate predictions file exists in the platform folder.
- A forced gate failure (e.g. a stricter threshold) offers "Reopen features loop" or "Continue training loop" with the reasons.

## Tasks

### 6.1 Stage 3 in chat: model config, `train.py`, leaderboard

- **Type:** AFK
- **Blocked by:** 4.7
- **PRD reference:** §6.2 (stage 3)
- **System design reference:** ADR-005; D29, D38

**Scope:**
- **Config and skill:** the stage 3 model config, plus the `ds-train-eval` skill (model families by task type, search-plan format, bias/variance and segment diagnosis).
- **Training:** in chat, the agent edits the config, the platform generates `train.py`, and training runs in the check runner on fixed CV folds from the spec.
- **Leaderboard:** CV score, fold spread, train/CV gap and status per candidate.

**Acceptance criteria:**
- [ ] The same folds are used for every candidate (test via the recorded fold ids).
- [ ] Stage 3 checks run: fold stability, train/CV gap, beats the naive baseline, no segment disproportionately bad (blocking).
- [ ] Demoable in isolation.

### 6.2 Search-plan proposals run by Optuna

- **Type:** AFK
- **Blocked by:** 5.3, 6.1
- **PRD reference:** §8.6
- **System design reference:** §5.4, §4 `proposal.json`; D22

**Scope:**
- **Proposal:** the stage 3 `proposal.json` schema is one search plan (family, ranges, trials) plus a rationale.
- **Search:** Optuna runs the plan in the check runner, within the 30-minute stage 3 per-experiment limit. Outputs are `search_plan.json`, `trials.csv` and per-fold CV results.
- **Keep rule:** "CV improves by at least the minimum, fold spread within tolerance, gap acceptable, no CV segment much worse".

**Acceptance criteria:**
- [ ] Every trial is recorded; the seed is fixed and recorded.
- [ ] Data and features are frozen in stage 3 (a test fails if a proposal tries to change them).
- [ ] Demoable in isolation.

### 6.3 Explore → refine phases and the one-SE rule

- **Type:** AFK
- **Blocked by:** 6.2
- **PRD reference:** §8.6
- **System design reference:** §5.4; D47

**Scope:**
- **Phases in the program:** explore is one family per iteration with a small budget (e.g. 20 trials). Refine is the champion family only, with narrower ranges, a bigger budget (e.g. 100 trials), and "keep only if the gain is at least the fold spread". The switch happens when explore's stop rule fires.
- **One-SE rule:** a more complex model that wins by less than the fold spread doesn't replace a simpler champion.
- **UI:** the chart and timeline mark the phase switch.

**Acceptance criteria:**
- [ ] The phase is recorded on every experiment.
- [ ] One-SE rule test: a near-tie keeps the simpler model.
- [ ] Demoable in isolation.

### 6.4 Pipeline files for kept experiments and the champion

- **Type:** AFK
- **Blocked by:** 6.2
- **PRD reference:** §8.8 (model files)
- **System design reference:** §3 ModelArtifact; D25, D37

**Scope:** Save the whole pipeline (stage 1 rules + stage 2 features + model) as one scikit-learn `Pipeline` with joblib, for kept experiments and the champion only. Discarded experiments keep code, metrics and logs, and can be re-run to rebuild the model. Files are loaded only inside sandbox containers.

**Acceptance criteria:**
- [ ] The Pipeline scores **raw** rows identically to running stages 1–3 separately (test).
- [ ] No model file is written for discarded experiments; re-running one rebuilds it.
- [ ] Verifiable in isolation.

### 6.5 Held-out gate, counter and gate predictions

- **Type:** AFK
- **Blocked by:** 6.4, 3.5
- **PRD reference:** §8.6, §9
- **System design reference:** §5.4, §8; ADR-003, D5, D8, D46

**Scope:**
- **Gate run:** when the loop stops, or the user requests evaluation in chat mode, a check-runner run with the held-out set mounted scores raw held-out rows with the champion Pipeline and applies the gate checks.
- **Counter:** incremented and shown in the UI ("held-out set used: N times"), including uses from chat mode.
- **Gate predictions:** predictions, targets and the spec's segment columns are saved to the platform folder.
- **Classification:** the threshold must come from out-of-fold predictions, never held-out data. This is implemented in 9.1; here the contract is defined.

**Acceptance criteria:**
- [ ] The held-out set is mounted only for this run (test the mounts of every other run type).
- [ ] The counter reads 1 after one gate; the gate predictions file exists and is not visible in the file browser.
- [ ] Demoable in isolation.

### 6.6 Gate-fail routing

- **Type:** AFK
- **Blocked by:** 6.5
- **PRD reference:** §8.6 (nested routing)
- **System design reference:** §5.4, §10

**Scope:** On gate failure, offer "Reopen features loop with these failures" or "Continue training loop", carrying the reasons into the reopened stage's brief or the loop state. Reopening stage 2 triggers the stale cascade (4.7) for stage 3.

**Acceptance criteria:**
- [ ] Both routes carry the failure reasons forward (visible in the brief or the loop state).
- [ ] The counter increments again only if the gate runs again.
- [ ] Demoable in isolation.
