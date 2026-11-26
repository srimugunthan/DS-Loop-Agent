# PRD Phase 1: DS-Loop-Agent with a Skills-Driven Agent

| | |
|---|---|
| **Status** | Draft v3: adds the post-modelling-analysis stage (five stages) and explore → refine tuning in stage 3. Draft v2 added the master agent and the file browser. Aligned with `decision-log.md` (decisions 1–47) |
| **Date** | 2026-11-27 (v1: 2026-11-26) |
| **Phase** | 1 of 2. Phase 2 (AI coworkers) is in `PRDs/PRD-phase2-ai-coworkers.md` and starts only when this phase's exit criteria are met |
| **Inputs** | `UI.png`, `AGENT_DS_WORKFLOW_BRAINSTORMING.md` (formerly `AGENT_DS_WORKFLOW_PLAN.md`, revision 2), the two loop-engineering decks, `loop-engineer-skill.skill` |

---

## 1. Summary

Phase 1 builds **DS-Loop-Agent** (*an agentic, loop-engineered data science workflow*) end to end with **one kind of agent: a skills-driven agent**, meaning a general agent harness configured with our own prompts, tools and skills (starting from the `loop-engineer` skill).

A data scientist starts by chatting with a **master agent**, which takes the problem and the dataset location, drafts the problem spec, and then guides them through the workflow. The work itself is done by five stage agents in sequence; the master agent explains each stage's check results and suggests the handoff, but only the user can approve a stage. With the stage agents, the user works in two modes:
- **Chat mode:** instruct the agent turn by turn (the mockup in `UI.png`).
- **Loop mode:** give a goal; the agent iterates on its own under objective checks until a stopping rule fires; then examine the results.

Every stage's output is reviewed in the right pane and approved before moving on.

Phase 1 proves three things:
1. The workflow, both modes and the review step work with a skills-driven agent.
2. Loop engineering (the Karpathy loop from your decks) works and is visible in the UI.
3. The tool is problem-agnostic: it runs unchanged on house prices (regression) and customer churn (classification).

It also leaves a clean **agent interface** so Phase 2 can plug in AI coworkers without changing the UI.

> *Claim for Phase 1:* an agent's output in a DS workflow is only trustworthy when each stage has a narrow scope, a human review step, and objective checks the agent cannot overrule — whether the agent is chatting or looping.

---

## 2. Problem

- Data scientists already use agents to write cleaning, feature and training code, usually in one long chat. Stages blur together, outputs are hard to review, and nothing stops a plausible-looking mistake from flowing into the model.
- **The mockup's own sample output shows the risk.** `data_cleaning.py` fits median and mode imputers on the *whole* dataset before any train/test split, which is a mild leak. It reads as correct, and the agent's message calls it done.
- **Iterative work is where agents should save the most time, and where they are least trustworthy.** Trying 15 features or 12 model configurations is tedious for a human. An agent that judges its own attempts drifts towards whatever *looks* best, including leaky features and models that won by luck. Loop mode lets the agent do the iterating while the platform does the judging.

---

## 3. Users

| User | Goal |
|---|---|
| **Data scientist (primary)** | Reach a reviewed model faster; delegate tedious iteration (loop mode) while keeping control (chat mode, approval) |
| **Demo presenter (you)** | Show loop engineering live: staged agents, goal-driven loops, objective checks |

---

## 4. Goals and non-goals

### Goals
1. A working two-pane app matching section 5.
2. A **master agent** for project intake and guidance between stages (section 6.1), and five stage agents, run in order, each with its own history and output.
3. **Chat mode** for all five stages.
4. **Loop mode** for stages 1 to 3.
5. A **skills-driven agent** behind all stages, in both modes, working end to end.
6. A **review step per stage**, the same in both modes: output plus objective checks, then approve or request changes.
7. **Problem-agnostic for tabular supervised learning**, shown on two problems.
8. **Phase 2 readiness:** the agent sits behind a documented interface (section 10.3).

### Non-goals (Phase 1)
- **AI coworkers** (OpenHands, Goose, Grok Bot, Hermes, …), the agent selector and agent comparisons. These are Phase 2.
- Deployment, canary rollout and monitoring/retraining (the decks' stages 5 and 6).
- Fully autonomous runs across stages. Loop mode is bounded autonomy *within one stage*; moving to the next stage always needs human approval. The master agent guides but never approves or advances.
- Loop mode for stages 4 and 5 (post-modelling analysis and summary have no single metric to loop on).
- Multi-user collaboration, sharing and permissions.
- Production data. Public datasets only.
- Problem types beyond tabular supervised learning (time series, text, images, unsupervised, recommendation).

---

## 5. The target UI

### 5.0 Project step (master agent)

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ✦ DS-Loop-Agent                                  ● <Project name>   Saved   (avatar)     │
├───────────────────────────────────────────┬──────────────────────────────────────────────┤
│ DATASCIENCE AGENT WORKFLOW     Project    │ PROBLEM SPEC                                 │
│ [project] › [data-prep] › [features-gen] ›│ Dataset: telco_churn.csv · 7,043 rows · ✓ fp │
│   [model-training-eval] › …               │ ┌ problem.yaml (draft — confirm below) ───┐ │
│───────────────────────────────────────────│ │ task_type      binary_classification ✎  │ │
│ (icon) master-agent             ● Active  │ │ target         Churn                 ✎  │ │
│        Sets up the problem, guides you    │ │ positive_class Yes                   ✎  │ │
│        through the stages                 │ │ id_columns     customerID            ✎  │ │
│───────────────────────────────────────────│ │ primary_metric roc_auc               ✎  │ │
│ Master 10:20  What are we predicting,     │ │ split          stratified, 20% held out│ │
│               and where is the data?      │ │ …  (agent-filled values highlighted)   │ │
│ You 10:21  Predict churn. ~/data/telco.csv│ └────────────────────────────────────────┘  │
│ Master 10:21  Imported and profiled. I've │ Profile: 21 columns · TotalCharges: text,    │
│   drafted the spec — check the split.     │   11 blanks                                  │
│───────────────────────────────────────────│ [ Edit ]               [ Confirm spec ✓ ]    │
│ [ Message master agent…             ➤ ]   │                                              │
└───────────────────────────────────────────┴──────────────────────────────────────────────┘
```

Confirming the spec splits off the held-out set and unlocks stage 1. At every later handoff the master agent posts a short summary in the next stage's chat (e.g. "Stage 1 passed 6/6 checks; 3 rules added. Watch the `TotalCharges` blanks.") above the stage agent's greeting.

### 5.1 Chat mode (from `UI.png`)

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│ ✦ DS-Loop-Agent                                  ● <Project name>   Saved   (avatar)     │
├───────────────────────────────────────────┬──────────────────────────────────────────────┤
│ DATASCIENCE AGENT WORKFLOW        1 of 5  │ AGENT OUTPUT                  [Copy] [⤓]     │
│ [data-prep] › [features-gen] ›            │ </> data_cleaning.py                         │
│   [model-training-eval] › …               │ ┌ tabs: data_cleaning.py │ Checks │ … ────┐  │
│───────────────────────────────────────────│ │ 1  import pandas as pd                 │  │
│ (icon) data-prep-agent          ● Active  │ │ 2  from sklearn.impute import …        │  │
│        Cleans, validates, and prepares    │ │ …  (line-numbered, highlighted code)   │  │
│        Mode: [ Chat | Loop ]              │ │                                        │  │
│───────────────────────────────────────────│ │                                        │  │
│                 TODAY                     │ │                                        │  │
│ Data prep 10:24  Hi! I'm your data prep…  │ │                                        │  │
│                You 10:25  Clean the …     │ └────────────────────────────────────────┘  │
│ Data prep 10:25  I've prepared a cleaning │ ● Generated just now               30 lines  │
│───────────────────────────────────────────│ [ Request changes ]     [ Approve stage ✓ ]  │
│ [ Message data prep agent…          ➤ ]   │                                              │
└───────────────────────────────────────────┴──────────────────────────────────────────────┘
```

### 5.2 Loop mode

```
┌───────────────────────────────────────────┬──────────────────────────────────────────────┐
│ DATASCIENCE AGENT WORKFLOW        2 of 5  │ LOOP OUTPUT     ● Running · iter 7 of ≤15     │
│ [data-prep ✓] › [features-gen] › …        │ tabs: Program │ Timeline │ Chart │ Files │ Checks │
│───────────────────────────────────────────│ ┌ Chart ─────────────────────────────────┐  │
│ (icon) features-gen-agent    ● Looping    │ │ CV RMSE                                │  │
│        Mode: [ Chat | Loop ]              │ │  ●──●        ●──●──●   (kept = filled) │  │
│───────────────────────────────────────────│ │       ○  ○ ●         ○ (discarded = ○) │  │
│ GOAL                                      │ │ 1  2  3  4  5  6  7   iteration        │  │
│ "Find features that cut CV RMSE by ≥5%,   │ └────────────────────────────────────────┘  │
│  no leakage, at most 8 new features"      │ ┌ Timeline ──────────────────────────────┐  │
│───────────────────────────────────────────│ │ #7 house_age_at_sale    KEPT  −2.1%    │  │
│ #5 nbhd_mean_price  ✗ leakage (corr .98)  │ │    "age drives depreciation"           │  │
│ #6 log_lot_area     ✓ kept  −1.4%         │ │ #6 log_lot_area         KEPT  −1.4%    │  │
│ #7 house_age_at_sale ✓ kept −2.1%         │ │ #5 nbhd_mean_price  DISCARDED leakage  │  │
│───────────────────────────────────────────│ └────────────────────────────────────────┘  │
│ [ Add a hint for the next iteration… ➤ ]  │ [ Pause ] [ Stop ]                           │
│                                           │ Budget: 7/15 iters · 4m12s of 20m            │
└───────────────────────────────────────────┴──────────────────────────────────────────────┘
```

### 5.3 UI requirements

From the mockup (all are requirements):

| Area | Element | Behaviour |
|---|---|---|
| Header | App name, project name with status dot, "Saved", user avatar | Project state persists automatically |
| Left | "DATASCIENCE AGENT WORKFLOW" stepper with the Project step, 5 stage agents and "n of 5" | Order, active stage, progress |
| Left | Agent card: icon, name, description, status badge | Who the user is working with |
| Left | Chat thread grouped by day, sender and timestamp per message | Per-agent history |
| Left | Message box, Enter to send, Shift+Enter for a new line | Plain-language instructions |
| Right | Output title, current file name, file tabs | One tab per artefact |
| Right | Line-numbered, syntax-highlighted code view | Readable review of generated code |
| Right | Copy and Download | Take the artefact out of the app |
| Right | Footer: freshness and size | Tells the reviewer what they are looking at |

Added in this PRD:

| Area | Element | Behaviour |
|---|---|---|
| Stepper | **Project** step before stage 1 | Chat with the master agent; problem spec form (5.0) |
| Agent card | **Mode switch: Chat / Loop** | Section 8. Hidden for the Project step and stages 4–5 |
| Stepper | Status badges | *not started, active, looping, awaiting review, approved, stale* |
| Right | **File browser** (default view) | Tree of the project folder; version folders labelled *approved* / *latest*; refreshes live as the agent writes files. Folder links in agent messages (e.g. "Output is in `stages/data-prep/v003/`") open and expand that folder. Read-only: changes go through the agent. The held-out set and check code never appear |
| Right | **File tabs** | Double-clicking a file opens it in a tab in a read-only code viewer, with a side-by-side diff view for experiments. Previews by type: CSV = first 100 rows as a table, images shown, Markdown rendered, large or binary files show a size notice |
| Right | **Checks** tab | Pass/fail per check with reasons, separate from the agent's claims |
| Right | **Approve stage** / **Request changes** | The review step (section 9) |
| Right, loop mode | **Program**, **Timeline**, **Chart**, **Leaderboard** (stage 3) tabs; Pause, Stop; budget meter; stop banner | Section 8.4 |
| Left, loop mode | Goal box; compact iteration feed; **hint box** | The user's hint is read by the agent before its next proposal |

**Defect in the mockup to avoid:** the code view shows highlighter markup leaking into the code (`class="code-string">`). The real view must render the source exactly as written; Copy and Download give the raw file.

---

## 6. The workflow: the master agent and the five stage agents

The user moves through the stepper left to right: the Project step, then five stages. Each stage receives the **approved** output of the previous stage and nothing else.

### 6.1 The master agent

| | |
|---|---|
| **Job** | Intake: asks what is being predicted and where the data is, then drafts `problem.yaml`. Between stages: explains the check results in plain language, writes a short handoff note, and suggests when the stage looks ready |
| **Dataset intake** | The user gives a file path or uploads a CSV. **The platform**, not the agent, copies it into the project, records a fingerprint and profiles it. The master agent sees only the profile and a small row sample. After the user confirms the spec, the platform splits off the held-out set into a folder no agent can reach |
| **Authority: guide only** | The master agent **cannot** approve a stage, advance the workflow, skip or re-run a check, or change a verdict. Approvals and stage transitions belong to the user, through the platform |
| **Handoffs** | The platform builds each stage's brief from the approved artefacts, the problem spec and the last verdict. The master agent's handoff note is attached as a labelled note, so the essential inputs never depend on what the agent remembers to include |

### 6.2 The five stage agents

| # | Agent | Job | Typical artefacts | Objective checks (section 9) | Loop mode |
|---|---|---|---|---|---|
| 1 | **data-prep-agent** — "Cleans, validates, and prepares datasets" | Profile; clean, impute, fix types, units, duplicates | `data_cleaning.py`, cleaned-data preview, data quality report | Schema, null rates, ranges/outliers, duplicates, **row count (no silent drops)**, imputers fitted on train only, target untouched | Yes: one cleaning rule per iteration |
| 2 | **features-gen-agent** | Propose, build and prune features | `features.py`, feature list with rationale | Leakage (association with target, forbidden columns, time leakage), redundancy (VIF), CV lift on the spec's metric | Yes: one feature per iteration |
| 3 | **model-training-eval-agent** | Choose, train and tune models (hyperparameter tuning lives here, as the explore → refine phases in 8.6); evaluate the champion | `train.py`, leaderboard, CV results, held-out metrics, segment errors, plots | CV fold stability; train/CV gap; held-out metrics **computed by the platform**; beats the naive baseline; no segment disproportionately bad (blocking) | Yes: one search plan per iteration, then the held-out gate |
| 4 | **post-modelling-analysis-agent** | Explain the approved model and find where it fails: segment analysis, explainability, calibration or residuals by segment | `analysis.md` (findings and red flags), segment tables, SHAP plots | All numbers **computed by the platform** (per-segment metrics, SHAP importance and dependence, calibration/residuals by segment, importance stability across folds); every spec segment covered; every number in the write-up matches a computed result; **red flags**: an ID or forbidden column among the top features, one dominant feature, a segment beyond tolerance, importances unstable across folds | No (chat only) |
| 5 | **model-summary-agent** | Write up what was done, metrics, findings, limits, recommendation | `model_summary.md` (a model card) | Every number matches the logged metrics and the analysis results; limitations and red flags from stage 4 present | No (chat only) |

**Post-modelling analysis rules (stage 4).** The model is frozen: this stage explains, it doesn't change anything. The agent chooses extra cuts to examine (e.g. "churn recall for month-to-month customers in their first year") and interprets the results, but never produces a number itself. Held-out results come from the predictions saved at the stage 3 gate, so the **held-out counter stays at 1**; the agent sees only aggregates, never held-out rows. If a red flag points at a real problem, the user can **"Reopen stage 2 / stage 3 with these findings"**, which marks later stages stale and carries the findings into that stage's brief.

Checks are written once and **set up from the problem spec**:

| Check | Regression (house prices) | Classification (churn) |
|---|---|---|
| Primary metric | RMSE / MAE | ROC-AUC or PR-AUC (PR-AUC when the positive class is rare) |
| Naive baseline to beat | Predict the training mean/median | Predict the majority class / base rate |
| Lift per feature (stage 2) | Drop in CV RMSE | Gain in CV AUC |
| Target-association leak test | Correlation with the target | AUC of the single feature alone (near 1.0 is suspicious) |
| Split and CV folds | Time-ordered if a time column is declared | Stratified by the target (time-ordered if declared) |
| Extra stage 3 checks | Residuals by segment | Class balance, calibration, threshold chosen on out-of-fold predictions, per-segment recall |
| Segments | e.g. price band, neighbourhood | e.g. contract type, tenure band |

Stage rules:
- **Narrow scope.** Each agent may only change files in its own stage folder, and may read the previous stage's approved outputs.
- **Approve to advance.** Approval freezes the stage's artefacts as a version and unlocks the next stage.
- **Going back.** Reopening and re-approving an earlier stage marks all later stages **stale**.
- **Evaluation is a gate, not part of any loop.** The agent never sees the held-out set. Held-out metrics come from a platform check, and every use is counted and shown.

---

## 7. User flow — chat mode

1. **Create a project** and land on the **Project** step. The master agent asks what is being predicted and where the data is.
2. **Give the problem and the dataset** (a path or an upload). The platform imports and profiles it; the master agent drafts the **problem spec** (section 11.1).
3. **Confirm the spec** in the form. The platform splits off the held-out set and stage 1 unlocks.
4. **Stage 1 opens** in chat mode with the master agent's handoff note. The stage agent greets the user and offers a first profile of the data.
5. **The user instructs** ("Clean the customer churn dataset. Handle missing values and remove duplicates.").
6. **The agent works.** The chat shows progress; files appear in the right pane's file browser as they are written; the agent's code runs in the sandbox.
7. **The agent replies** with a short summary and a link to its output folder.
8. **Review.** The user opens the output folder, double-clicks files to read them, and checks the **Checks** tab. The master agent explains the verdicts on request. The user either requests changes (back to step 6; "Send check results to agent" passes failures back) or approves.
9. **Handoff.** The master agent posts a summary and note in the next stage's chat. Repeat steps 5–8 for stages 2 to 5, in chat or loop mode (loop mode for stages 2–3 only).
10. **Export** all approved artefacts as a bundle: per-stage scripts, an end-to-end `pipeline.py` that runs on a raw CSV, `requirements.txt`, metrics, summary, loop logs and the review log.

---

## 8. Loop mode

### 8.1 What it is

Loop mode applies the pattern from your revised deck and the `loop-engineer` skill to one stage:

| Karpathy-loop element | In loop mode |
|---|---|
| One editable scope | The agent changes **one thing per iteration** (one cleaning rule, one feature, one model change) |
| One objective metric | Taken from the problem spec (e.g. CV RMSE or CV AUC) |
| Verifier separate from the proposer | The platform check decides keep or discard; the agent cannot override it |
| Bounded retries | Hard caps on iterations, wall-clock time and consecutive failures, enforced by the platform |
| Stopping condition set before the loop starts | The loop **cannot start** without a stop rule |
| Logging | Every iteration is logged and shown as a timeline |

### 8.2 Switching to loop mode and giving a goal

1. The user flips the agent card's switch from **Chat** to **Loop**.
2. The user types a **goal** in plain language, for example:
   - stage 1: *"Get every data-quality check passing without dropping more than 1% of rows."*
   - stage 2: *"Find features that cut CV RMSE by at least 5%, no leakage, at most 8 new features."*
   - stage 3: *"Find the best model by CV AUC within 20 minutes; prefer interpretable models when results are close."*
3. The agent drafts a **loop program** (`program.md`, from the skill's template) from the goal plus the problem spec. It is shown as an editable form in the **Program** tab:

| Program field | Example (stage 2, house prices) |
|---|---|
| Goal | Cut CV RMSE by ≥5% versus the current feature set |
| Metric and direction | CV RMSE (from the problem spec), minimise |
| Editable scope | Add or remove one feature per iteration in `features.py` |
| Constraints | No forbidden columns; no target-derived features; no change to the target or the split |
| Keep rule | Lift ≥ 0.5% AND leakage checks pass AND VIF ≤ 8 |
| Budget | Max 15 iterations, 20 min total, 2 min per iteration, max 8 kept features |
| Stop rule | Goal reached, OR 4 iterations in a row with no keep, OR budget spent |
| Escalation | If the same idea fails twice, or the loop stops short of the goal, report to the user instead of continuing |

4. The user adjusts anything and presses **Start loop**. Values the agent filled in on its own are highlighted.

### 8.3 While the loop runs

Each iteration:
1. The agent reads the current state (best result so far, past iterations and their reasons, any user hint).
2. It **proposes one change** with a one-line rationale.
3. The platform **applies and runs it** in the sandbox.
4. The platform **check** decides keep or discard against the keep rule.
5. The iteration is **logged**: proposal, rationale, verdict, metric change, reasons, time taken.
6. Kept changes become the new current state; discarded ones are not promoted (they stay in the experiment record, 8.8).

Limits per experiment: 10 minutes and 4 GB by default (30 minutes in stage 3), plus a **dollar cap per loop** (default $0.25), all editable in the program. If the model API keeps failing after retries, the loop **pauses** with a Resume button; failed iterations don't use up the iteration budget. If the app crashes or restarts mid-loop, the loop shows as **Interrupted** and can be resumed from the last finished experiment.

The user can **watch**, **pause**, **stop**, or **add a hint** ("try interaction terms with neighbourhood"). A hint never overrides a check.

### 8.4 Examining the output

When the stop rule fires, a **stop banner** says why it stopped and whether the goal was met. If it wasn't, it says so plainly, e.g. "Stopped: 4 iterations with no gain. Goal not reached: −3.2% of the −5% target."

| Tab | Contents |
|---|---|
| **Program** | The loop program as run (read-only after start) |
| **Timeline** | One card per iteration: proposal, rationale, KEPT/DISCARDED, metric change, reasons; filter by verdict |
| **Chart** | Metric over iterations: kept filled, discarded hollow, champion line, goal as a dashed line |
| **Leaderboard** (stage 3) | Every candidate model: CV score, fold spread, train/CV gap, status |
| **Files** | The resulting artefacts |
| **Checks** | Full check results for the final state |

Then the user picks: **Approve stage**, **Continue the loop** (bigger budget or edited program), **Switch to chat** (the loop's result becomes the starting point), or **Discard** (back to the previous version).

### 8.5 Loops for stages 1 and 2

| | Stage 1 — data-prep loop | Stage 2 — features loop |
|---|---|---|
| One change per iteration | One cleaning rule (e.g. "impute `LotFrontage` by neighbourhood median") | Add or remove one feature |
| Metric | Data-quality score: passing checks, plus null and outlier rates | CV primary metric from the spec |
| Keep rule | Quality improves AND no unlogged row drops AND target untouched | Lift ≥ minimum AND leakage checks pass AND VIF within limit |
| Natural stop | All checks pass | No gain for N iterations, or feature budget reached |
| Showcase moment | A rule that drops rows to "pass" the null check is discarded | A leaky feature with a huge apparent lift is discarded; the curve flattens and the loop stops itself |

### 8.6 Loop for stage 3 — finding the right model

Training is a loop; **held-out evaluation is a one-time gate after it.** If the loop could see held-out metrics, the held-out set would become a tuning set and the final score would be optimistic.

```
 program.md (set before the loop starts)
   metric: CV <primary_metric from problem spec>
   budget: e.g. max 12 iterations, 30 min total, 30 min per experiment
   keep if: CV improves ≥ 0.5% AND fold spread within tolerance
            AND train/CV gap acceptable AND no CV segment much worse
   phases: explore → refine (switch when explore stops)
   stop if: 4 iterations with no keep, or budget spent
        │
        ▼
 ┌──────────── TRAINING LOOP (training data only, fixed CV folds) ────────────┐
 │  PHASE A — EXPLORE                                                          │
 │   one model family per iteration, small tuner budget (e.g. 20 trials)       │
 │   agent reads leaderboard + diagnostics → proposes ONE search plan          │
 │   platform check → keep (new champion) / discard; simplicity rule applies   │
 │                          │ explore stop rule fires                          │
 │                          ▼                                                  │
 │  PHASE B — REFINE                                                           │
 │   champion family only, narrower ranges, bigger budget (e.g. 100 trials)   │
 │   keep only if the gain ≥ the fold spread                                   │
 └───────────────────────────────┬────────────────────────────────────────────┘
                                 │ stop rule fires
                                 ▼
          HELD-OUT EVALUATION GATE (once, on the champion only;
                                    predictions saved for stage 4)
            pass → approve → post-modelling-analysis-agent
            fail → reopen the training or features loop, with the reasons
```

- **Tuning lives here, not in its own stage.** Choosing a model family and tuning it can't be separated fairly (an untuned family loses to a tuned one), so both happen in this loop: *explore* compares families with small budgets, *refine* spends a bigger budget on the champion only. The Timeline and Chart mark the switch from explore to refine. There is one approval and one held-out gate.

- **What the agent may change, one per iteration:** model family, a hyperparameter search range, regularisation, class weighting, loss or target transform (e.g. log price). Data and features are frozen.
- **Hybrid with a tuner.** Each iteration the agent proposes a **search plan** (e.g. "gradient boosting, depth 3–8, learning rate 0.01–0.1, 20 trials"). A tuner such as Optuna runs it within the per-iteration budget, and the check judges the best result. The agent's value is diagnosis ("train error low, CV error high: overfitting"), choosing the next family and explaining the winner.
- **Fair comparison:** the same CV folds every iteration (time-ordered for house prices, stratified for churn). For classification, the decision threshold is chosen from out-of-fold predictions, never the held-out set.
- **Simplicity rule:** if a more complex model beats the champion by less than the fold spread, the simpler one stays champion (the "one standard error" rule).
- **Held-out counter:** the UI shows "held-out set used: N times". Normally 1 per approved model, and it counts uses from chat mode too.
- **Nested routing:** if the gate fails, the UI offers **"Reopen features loop with these failures"** or **"Continue training loop"**, carrying the reasons forward.
- **Honest limit:** many iterations on the same folds still overfit to them slightly. The minimum gain, the simplicity rule and the one-time gate keep this small but don't remove it.

### 8.7 Mode rules

- Both modes produce the same kind of artefacts and end in the same review and approval step.
- A loop starts from the stage's current version.
- Switching from loop to chat keeps the loop's result as the current version.
- One loop at a time per project.

### 8.8 Experiment record

Every loop experiment, and every chat turn that changes code, is kept permanently, **including failures**:

| Item | Detail |
|---|---|
| One folder per experiment | `proposal.json` (what was proposed and why), the **full code exactly as it ran**, `diff.patch` against the starting version, `verdict.json` (checks, metrics, keep/discard, reasons), `run.log` (output, errors, timing). Stage 3 adds `search_plan.json`, `trials.csv` (every tuner trial) and per-fold CV results |
| One log line per experiment | In `experiments.jsonl`: ids, stage, mode, backend, starting version, proposal summary, status, metric before/after, reasons, duration, tokens/cost, file paths |
| Statuses | `kept`, `discarded` (failed a check), `error` (code crashed), `timeout`, `invalid` (broke the proposal rules) |
| Nothing is deleted | "Undo" means an experiment is not promoted to the stage's current version |
| Re-runnable | Each experiment records the data fingerprint, seed and CV folds |
| Model files | Kept only for kept experiments and the champion; discarded models can be rebuilt from their folder |
| MLflow mirror | The platform (never the agent) mirrors every experiment to MLflow: project + stage → MLflow experiment, loop → parent run, experiment → child run. Params, metrics, status tags and the folder as artifacts; kept and champion models logged; the approved model registered. Folders + JSONL remain the source of truth |
| In the UI | Each Timeline card opens its experiment's code, diff, verdict and log. An **Experiments** tab lists all experiments across loops and chat, filterable by status |

---

## 9. Review and verification

Reused from `AGENT_DS_WORKFLOW_BRAINSTORMING.md` sections 1.1, 2.2 and 7. Applies to **both modes**.

| Requirement | Detail |
|---|---|
| **Human review is mandatory** | No stage advances without explicit approval |
| **Objective checks run outside the agent** | Deterministic Python checks, one set per stage, owned by the platform. The agent cannot edit them. Results are shown separately from the agent's claims |
| **The agent never grades its own work** | In chat mode, a failed check is shown prominently even if the agent says "done". In loop mode, only the check can keep a change |
| **The held-out set is hidden from the agent** | Only the stage 3 gate can read it; every use is counted |
| **Limits are enforced by the platform** | Loop budgets and stop rules are enforced by the loop runner, not just written in the prompt |
| **Chat-mode retry nudge** | After 5 failed attempts in chat, the UI suggests the user step in or switch to loop mode with a clearer goal |
| **Agent code runs in a sandbox** | Agents run in a per-project container that sees only its stage workspace and can reach only the Anthropic API. Checks run in a separate container with no network and read-only access. In `start.sh` dev mode these are local processes and the UI shows an "Unsandboxed — dev mode" banner |
| **API key handling** | An **LLM gateway** holds the only copy of the API key, read from a git-ignored `.env`. Agent containers have no internet and only a placeholder key; all model calls go through the gateway, which records cost per experiment and enforces each loop's dollar cap per request. Also: a demo-only key with a monthly spend limit, and anything that looks like a key is masked in logs |
| **The master agent never approves** | It explains and suggests; approvals and stage transitions are the user's, through the platform |
| **Approving with a failed check** | Allowed, but needs a reason, which is logged |
| **Everything is logged** | Messages, file versions, loop iterations, check results and approvals |

Reuse from the existing skill:
- `program_template.md` becomes the loop program form (8.2).
- `feature_check.py` becomes the stage 2 check after the plan's fixes (remove `eval()`, fix the docstring example, configurable model), and is **generalised beyond regression** (today it only computes RMSE with a linear model).
- `references/*.md` read their metric, baseline and segments from the problem spec instead of assuming house prices.
- The plan's defect-injection approach (seeded defects and planted leaks with an answer key) is reused for both datasets.

---

## 10. The skills-driven agent

### 10.1 What it is

A general agent harness that **we** configure per stage:

| Part | Detail |
|---|---|
| Harness | Claude Agent SDK, used "coworker-shaped": the agent sees only a workspace folder, its brief and the loop program, and returns results as files (e.g. `proposal.json`). No custom tools that call the backend |
| Model | Claude Haiku 5.5 by default for the master agent and all stages, in both modes; configurable per stage and mode |
| Skills | `loop-engineer` (the loop pattern and program template), `ds-master` (intake and handoffs) and one skill per stage (data prep, features, training/evaluation, summary). Skills read everything dataset-specific from the problem spec |
| Stage prompt | Role, scope, the problem spec, the previous stage's approved outputs, the master agent's handoff note |
| Tools | The SDK's built-in tools (read, write, edit, run commands), acting only inside the agent's workspace. No access to checks code, the held-out set or other stages' folders |
| Modes | Chat: normal multi-turn session. Loop: called once per iteration to write one `proposal.json` |

### 10.2 Skills to write

| Skill | Contents |
|---|---|
| `loop-engineer` (existing) | Generalised: metric and thresholds from the problem spec; program template unchanged |
| `ds-master` | Intake questions, how to draft `problem.yaml` from a data profile (split choice, leak-prone columns), how to explain verdicts, handoff note format, the "guide only" rule |
| `ds-data-prep` | Profiling routine, cleaning rule catalogue, "never drop rows silently", fit-on-train-only rule |
| `ds-features` | Feature idea catalogue by data type, leakage pitfalls, one-feature-per-iteration protocol |
| `ds-train-eval` | Model families by task type, search-plan format for the tuner, diagnosis patterns (bias/variance, segments) |
| `ds-analysis` | Which analyses to request per task type, how to read SHAP and segment results, red-flag patterns (leaks showing up as top features, segment failures), when to recommend reopening an earlier stage |
| `ds-model-summary` | Model card template; every number must come from the log or the analysis results |

### 10.3 Agent interface (Phase 2 readiness)

The skills-driven agent is wrapped in an adapter that implements this interface. The UI and the loop runner talk **only** to the interface. In Phase 2 the AI coworkers implement the same interface, so no UI changes are needed. The master agent uses the same interface, with `stage = project`.

| Operation | Purpose |
|---|---|
| `start(stage, project_context)` | Open a session with the stage brief, scope, previous outputs and problem spec |
| `send(message)` | Chat mode: pass a user message or check results |
| `propose(loop_state)` | Loop mode: given the program, current best, past iterations with reasons and any hint, return **one** change plus a rationale |
| `events()` → stream | Chat text, status, tool calls, file written, errors, done |
| `artifacts()` | Files produced in the stage folder, with versions |
| `stop()` | End or interrupt the session |
| `capabilities()` | What the backend supports (Phase 1: everything) |

Also required for Phase 2 readiness:
- Every log entry records the **backend id and version** and, where available, **tokens and cost**.
- The **loop runner owns the loop**: it calls `propose`, applies the change, runs the check, logs, and decides whether to continue.

---

## 11. Problems, datasets and state

### 11.1 The problem spec

A small file (`problem.yaml`) saved with the project. Every agent receives it, and every check and loop program reads its metric, split and segments from it. No prompt, skill or check may hardcode a dataset, column or metric.

| Field | Example: house prices | Example: churn |
|---|---|---|
| `task_type` | `regression` | `binary_classification` |
| `target` | `SalePrice` | `Churn` |
| `positive_class` | — | `Yes` |
| `id_columns` | `PID` | `customerID` |
| `time_column` (optional) | `YrSold`/`MoSold` | none |
| `primary_metric` | `rmse` | `roc_auc` |
| `split` | time-ordered, last period held out | stratified random, 20% held out |
| `cv_folds` | 5, time-ordered | 5, stratified |
| `segments` | price band, neighbourhood | contract type, tenure band |
| `forbidden_columns` | none | any column recorded after the customer left |
| `domain_notes` | "Prices are in USD; area should be sqft" | "Churn is measured over the last month" |

The platform profiles the dataset (types, missing values, cardinality, likely id, time and target columns); the master agent drafts the remaining values; the user confirms everything in a form, with agent-filled values highlighted. The spec is shown on every review screen.

### 11.2 The two problems

**Same agent, skills, checks, loop programs and UI; only the dataset and spec change.**

| | House price prediction | Customer churn |
|---|---|---|
| Task | Regression | Binary classification |
| Why it is here | Matches your decks, the skill and the plan | Matches the UI mockup; exercises the classification checks |
| Suggested dataset | Ames Housing (public, has sale dates) | IBM Telco Customer Churn (public, about 7,000 customers) |
| Defects | Planted: $50 price typo, sqft/sqm mixing, nulls, duplicates | Natural (verify): `TotalCharges` stored as text with blanks for new customers. Planted: duplicate customers, inconsistent labels |
| Planted leak (caught in stage 2) | Neighbourhood mean price over all years | A post-churn column (e.g. a cancellation reason) |
| Subtle planted leak (caught in stage 4) | A weak proxy for the sale price that passes the stage 2 association threshold but dominates SHAP importance (exact design in the answer key) | A weak proxy for churn, such as a support-ticket count recorded partly after cancellation (exact design in the answer key) |
| What it shows | Time-ordered split, residual segments | Class imbalance, AUC versus accuracy, per-segment recall |

The plan's house-price dataset design (`AGENT_DS_WORKFLOW_BRAINSTORMING.md` section 2.4) still applies. Churn needs its own answer key.

### 11.3 State

- **Project** = dataset (with fingerprint) + problem spec + the master agent's chat + the five stage records.
- **Stage record** = backend id, mode history, chat history, loop runs (program, iterations, stop reason), artefact versions, check results, approval status.
- **Storage:** a project folder with numbered version directories plus JSONL logs. Saved automatically. Datasets up to about 1M rows (fits in memory with pandas).
- **Stale stages:** when an earlier stage is re-approved, later stages are automatically re-run and re-checked on the new inputs, then wait for the user to re-approve.

---

## 12. Architecture

```
Browser (React)                Backend (Python, FastAPI, 127.0.0.1)          Containers (Docker Compose)
┌────────────────┐  SSE    ┌────────────────────────────────────┐      ┌──────────────────────────────┐
│ Stepper        │◄────────│ Session manager                    │─────►│ Agent container per project  │
│ Chat pane      │  REST   │ Loop runner (budgets, stop rules,  │      │  master + stage agents       │
│ Spec form      │────────►│   keep/discard, resume)            │      │  (Agent SDK, Haiku 5.5)      │
│ File browser + │         │ Agent interface (10.3)             │      │  workspace only, no internet,│
│  code viewer   │         │   └ skills-driven agent adapter    │      │  placeholder key             │
│ Loop panel     │         │ Dataset import + profiler          │      └──────────────┬───────────────┘
│ Checks panel   │         │ Script generator / export          │                     ▼ /exp/<id>
└────────────────┘         │ Project store (folders + JSONL) ◄──┼─cost─ LLM gateway (holds key, meters $,
                           │                                    │        enforces cap) ──► api.anthropic.com
                           │                                    │      ┌──────────────────────────────┐
                           │ Background job manager             │─────►│ Check-runner container       │
                           └───────────────┬────────────────────┘      │  per run: no network,        │
                                           │ mirror                    │  workspace read-only,        │
                                           ▼                           │  Optuna tuner (stage 3),     │
                                MLflow (tracking UI, model registry)   │  SHAP + segment analysis     │
                                                                       │  (stage 4); held-out set     │
                                                                       │  mounted only for the gate   │
                                                                       └──────────────────────────────┘
```

`start.sh` dev mode runs the same parts without Docker: agents and checks as local processes, with an "Unsandboxed — dev mode" banner. The full design and its reasoning are in `decision-log.md` and, when written, `system-design.md`.

---

## 13. Milestones

| Milestone | Scope | Exit criteria |
|---|---|---|
| **P1-M0: Skeleton** | Two-pane UI, stepper (with the Project step), chat pane, **file browser and code viewer** on a fake agent; mode switch and loop panel on canned data; project storage | Mockups in 5.0–5.2 are clickable; double-clicking a file opens it; code renders correctly; Copy and Download work |
| **P1-M1: Intake and stages 1–2, house prices** | **Master agent intake** (import, profile, spec form, held-out split); skills-driven agent behind the interface; sandbox; **LLM gateway**; checks for stages 1–2; approve/advance with handoff notes; chat mode for stages 1–2; **loop mode for stage 2** | The master agent drafts a correct spec from the raw CSV. In chat, a planted defect is flagged. The features loop discards the planted leak, stops by its own rule, and its result can be approved |
| **P1-M2: All stages, house prices** | Stages 3–5 in chat; **stage 3 training loop** (explore → refine, leaderboard, tuner, simplicity rule, held-out gate with saved predictions); **post-modelling analysis** (platform-computed segments and SHAP, red flags, reopen routing); stage 1 loop; stale handling; export | End-to-end run produces an approved model, analysis and summary; held-out counter reads 1 per approved model, including after stage 4; the subtle planted leak is flagged by stage 4; summary numbers match the logs and the analysis |
| **P1-M3: Second problem** | Classification checks; churn dataset, spec and answer key | End-to-end on churn, in both modes, with **no changes to agent prompts, skills, loop runner or UI**. Any change needed exposes something hardcoded |
| **P1-M4: Phase 1 showcase** | Golden-run recording; README; run report | Showcase flow (section 14) recorded |

**Phase 1 exit criteria** (the gate for starting Phase 2):
1. All five stages approved on both problems, using both modes where available.
2. Every planted defect and leak caught, against both answer keys.
3. The agent interface (10.3) documented, and the skills-driven agent uses nothing outside it.
4. Logs record backend id and cost.

---

## 14. Phase 1 showcase flow (about 9 minutes)

| Min | Beat | Mode | What the audience sees |
|---|---|---|---|
| 0–1 | The problem and the spec | **Master agent** | "Predict sale price; data is in `~/data/ames.csv`." Seconds later, a pre-filled spec with a time-ordered split; the user confirms |
| 1–2 | Stage 1 | **Chat** | The user asks for cleaning; the Checks tab flags something the agent called "done" |
| 2–4 | Stage 2 | **Loop** | Goal typed; program form; the loop runs; the leaky feature's huge "lift" is discarded; the curve flattens; the loop stops itself and says why |
| 4–6 | Stage 3 | **Loop** | Explore: the leaderboard grows; a lucky model is rejected for unstable folds; the simplicity rule keeps a simpler champion. Refine: the champion gets a bigger tuning budget. The held-out gate runs once (counter = 1) |
| 6–7 | Stage 4: post-modelling analysis | **Chat** | Segment table and SHAP importance; a **red flag**: a subtle planted leak that passed the stage 2 checks is the top feature. "Reopen stage 2 with these findings" (shown, or replayed from the golden run). Counter still 1 |
| 7–8 | Stage 5 | **Chat** | The model card; every number traced to the log and the analysis |
| 8–9 | Same tool, different problem | **Master agent** | Churn project: a new CSV, a stratified split drafted automatically; same agents, same loops; the closing claim |

---

## 15. Success metrics

| Metric | Target |
|---|---|
| End-to-end completion | All 5 stages approved on **both** problems, using both modes where available |
| Generality | House prices to churn needs only a new dataset and spec |
| Loop discipline | Every loop stops by its own rule within budget; 0 cases where a discarded change reached the final artefacts |
| Planted defects and leaks caught | All of them, against each answer key |
| Agent claims contradicted by checks | Counted and shown; the evidence for the thesis |
| Held-out discipline | Held-out set used once per approved model |
| Human effort, chat versus loop | Turns and minutes of user time per stage, by mode |

---

## 16. Risks

| Risk | Mitigation |
|---|---|
| A loop overfits to the CV folds | Minimum gain, simplicity rule, one-time held-out gate |
| A loop runs too long or costs too much | Budgets and stop rules enforced by the loop runner; a loop cannot start without them |
| A vague goal produces a bad program | The program is shown and must be confirmed; agent-chosen defaults are highlighted |
| The agent games the keep rule (e.g. dropping rows) | Checks cover known games (row counts, forbidden columns, target untouched); every iteration is reviewable |
| The agent edits checks or reads the held-out set | Checks and held-out data live outside the agent's sandbox and scope |
| Generated code runs unsafely | Sandbox per stage, no network or secrets |
| Reviewers approve without reading | Failed checks shown prominently; approving over a failed check needs a logged reason |
| Nondeterministic agent makes the demo vary | Record a golden run |
| Hidden dataset assumptions | Problem spec is the only source of dataset details; P1-M3's "no changes" rule |
| "Any problem" over-promised | Scoped to tabular supervised learning, said so in the demo |
| The interface turns out wrong for coworkers | Review it against the Phase 2 shortlist (OpenHands, Goose) before P1-M2 ends |
| The master agent is trusted as a judge | It has no approve or advance capability; verdicts come only from platform checks, shown separately from its explanation |
| The master agent drafts a wrong spec (e.g. random split on time data, a leak column not forbidden) | The spec form highlights agent-filled values and must be confirmed; the platform's profile flags likely time columns |
| A cheap default model (Haiku 5.5) gives weak proposals | The model is configurable per stage and mode; move only the weak stage to a larger model |

---

## 17. Open questions

1. **Review controls:** are the additions in 5.3 (approve buttons, Checks tab, status badges, mode switch) consistent with your vision?
2. **Loop defaults:** are the example budgets reasonable (stage 2: 15 iterations / 20 minutes; stage 3: 12 iterations / 30 minutes), or smaller for live demos?
3. **Hints during a loop:** allowed (8.3), or should loops be hands-off once started?

Resolved during system design (see `decision-log.md`): agent execution in a sandbox (decisions 27–28), the harness (Claude Agent SDK, decision 19), and the form (a local app via Docker Compose or `start.sh`, decision 17).
