# Phase 5: Loop Engine (Stages 1–2)

**Depends on:** `phase4-checks-and-chat-stages-1-2.md`
**Status:** Not started

## Design components touched

Loop runner + in-process job manager (§2, §5.3; D6, D7, D32), `proposal.json` contract (§4; ADR-001, D22), experiment record (§3, §6; D24), budgets and limits (D35, D40), gateway cap (ADR-007), failure handling (§10; D31, D39).

## Objective

Loop mode works for stages 1 and 2:
1. The user switches the agent card to Loop and types a goal.
2. The agent drafts a loop program, which the user edits and starts.
3. Each iteration, the agent writes exactly one change to `proposal.json`. The platform validates and applies it, the check runner judges it, and the loop runner keeps or discards it and logs the experiment.
4. Budgets, per-experiment limits and the dollar cap stop the loop with a clear reason.

The user can watch the timeline and chart, pause, stop, add hints, and resume after a crash or API error.

## Independently testable when

- On Ames stage 2, the features loop discards the planted leak (despite its huge apparent lift), keeps real features, and stops by its own rule with a stop banner explaining why.
- Killing the backend mid-loop leaves the loop *Interrupted*; Resume continues with the remaining budget.
- Every experiment, including failures, has a folder and an `experiments.jsonl` line with the right status.

## Tasks

### 5.1 Goal → loop program form

- **Type:** AFK
- **Blocked by:** 4.6
- **PRD reference:** §8.2
- **System design reference:** §3 LoopRun

**Scope:**
- **Mode switch:** Chat/Loop, hidden for the Project step and stages 4–5.
- **Goal box:** the agent, using the generalised `loop-engineer` skill whose metric and thresholds come from the spec, drafts `program.md`.
- **Program tab:** shows it as an editable form (goal, metric and direction, scope, constraints, keep rule, budgets including the $ cap and per-experiment limits, stop rule, escalation). Agent-filled defaults are highlighted.
- **Start Loop:** disabled until a stop rule exists.

Defaults use the PRD's example budgets (open PRD question §17.2: editable here).

**Acceptance criteria:**
- [ ] A loop cannot start without a stop rule.
- [ ] The program as run is saved read-only with the LoopRun.
- [ ] Demoable in isolation.

### 5.2 `proposal.json` schemas for stages 1–2

- **Type:** AFK
- **Blocked by:** 5.1
- **PRD reference:** §8.1
- **System design reference:** §4 `proposal.json`; ADR-001, D22

**Scope:** Versioned JSON schemas:
- stage 1: one cleaning-rule function;
- stage 2: add a feature (name + function) or remove one by name.

Each includes a one-line rationale. A validator turns malformed or multi-change proposals into `invalid` experiments.

**Acceptance criteria:**
- [ ] Schema tests: valid, malformed and multi-change cases.
- [ ] An invalid proposal is logged with its reason and doesn't stop the loop.
- [ ] Verifiable in isolation.

### 5.3 Loop runner core (stage 2)

- **Type:** AFK
- **Blocked by:** 5.2
- **PRD reference:** §8.3, §8.8
- **System design reference:** §5.3; D6, D7, D24, D32

**Scope:** An in-process background job, one loop per project. Each iteration:
1. call `propose(loop_state)`, with past iterations, reasons, the current best and any hint;
2. validate the proposal;
3. apply the one change to a candidate version;
4. run the check runner;
5. keep (promote) or discard (don't promote);
6. write the experiment folder (`proposal.json`, code as run, `diff.patch`, `verdict.json`, `run.log`, fingerprint, seed, folds, cost) and append to `experiments.jsonl`;
7. push a timeline card over SSE.

**Acceptance criteria:**
- [ ] On Ames, the planted leak proposal is discarded with "leakage" as the reason.
- [ ] Discarded changes never appear in the stage's current version (test).
- [ ] Demoable in isolation.

### 5.4 Budgets, limits, the dollar cap and the stop banner

- **Type:** AFK
- **Blocked by:** 5.3
- **PRD reference:** §8.3, §8.4
- **System design reference:** §5.3; D35, D40

**Scope:** Enforce:
- iteration and wall-clock budgets;
- per-experiment limits of 10 minutes and 4 GB on the check runner, giving `timeout` when exceeded;
- consecutive no-keep stop rules;
- the $ cap: the gateway refuses the request, and the loop runner stops the loop.

A budget meter shows live progress, and a stop banner states the reason and whether the goal was met.

**Acceptance criteria:**
- [ ] Each stop reason can be triggered in a test and is reported correctly.
- [ ] A `timeout` experiment is logged and the loop continues.
- [ ] Demoable in isolation.

### 5.5 Chart and Experiments tab

- **Type:** AFK
- **Blocked by:** 5.3
- **PRD reference:** §8.4, §8.8
- **System design reference:** §3 Experiment; D24

**Scope:**
- **Chart:** the metric over iterations, with kept points filled, discarded points hollow, the champion line and the goal as a dashed line.
- **Experiments tab:** lists all experiments across loops and chat revisions, filterable by status. Each item opens its code, diff (in the Monaco diff view), verdict and log.

**Acceptance criteria:**
- [ ] The chart matches `experiments.jsonl`.
- [ ] Every status filter works, including `error`, `timeout` and `invalid`.
- [ ] Demoable in isolation.

### 5.6 Pause, stop, hints, API-error pause and resume

- **Type:** AFK
- **Blocked by:** 5.3
- **PRD reference:** §8.3
- **System design reference:** §10; D31, D39

**Scope:**
- **Controls:** Pause and Stop. A hint box: the agent reads the hint before its next proposal, and a hint never overrides a check.
- **API errors:** after the SDK's retries fail, the experiment is logged `error` and the loop pauses ("Paused — API error"). A failed iteration doesn't use up the iteration budget, but its cost counts.
- **Crash recovery:** loop state is saved after every experiment. After a backend restart, the loop shows *Interrupted* with Resume, and the in-progress experiment is logged `error`.

**Acceptance criteria:**
- [ ] Kill the backend mid-experiment, then restart and Resume: it continues with the correct remaining budget.
- [ ] A simulated gateway error pauses the loop, and Resume continues it.
- [ ] Demoable in isolation.

### 5.7 Stage 1 loop

- **Type:** AFK
- **Blocked by:** 5.3
- **PRD reference:** §8.5
- **System design reference:** §5.3

**Scope:** Loop mode for data-prep: one cleaning rule per iteration. The metric is a data-quality score; the keep rule is "quality improves AND no unlogged row drops AND target untouched"; the natural stop is all checks passing.

**Acceptance criteria:**
- [ ] A rule that drops rows to "pass" the null check is discarded (the showcase moment).
- [ ] The loop stops when all stage-1 checks pass.
- [ ] Demoable in isolation.

### 5.8 End-of-loop actions

- **Type:** AFK
- **Blocked by:** 5.4
- **PRD reference:** §8.4, §8.7
- **System design reference:** §5.6; D3

**Scope:** After a stop, the user can:
- **Approve stage**, which uses the normal approval flow;
- **Continue the loop**, with a bigger budget or an edited program;
- **Switch to chat**, with the loop's result as the current version;
- **Discard**, returning to the previous version.

**Acceptance criteria:**
- [ ] Each action leaves the stage in the right state with the right current version.
- [ ] Demoable in isolation.
