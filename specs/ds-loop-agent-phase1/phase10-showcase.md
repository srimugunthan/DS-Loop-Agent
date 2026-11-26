# Phase 10: Phase 1 Showcase

**Depends on:** `phase9-second-problem-churn.md`
**Status:** Not started

## Design components touched

Observability outputs: the experiment record, MLflow, the budget meter, stop banners, the held-out counter, the gateway's cost records (§9). No new components.

## Objective

Record the about-9-minute showcase (PRD §14) on both problems, as golden runs. Produce a run report with the PRD's success metrics, and write the README and demo script so the project can be presented and reproduced.

## Independently testable when

- The golden-run recordings cover every beat in PRD §14:
  - master agent intake;
  - stage 1 chat with a check contradicting the agent;
  - the stage 2 loop discarding the leak and stopping itself;
  - stage 3 explore → refine and the one-time gate;
  - stage 4 flagging the subtle leak, then reopening;
  - the stage 5 model card;
  - the churn project with the same tool.
- The run report gives every PRD §15 metric, with numbers traced to the logs.
- The Phase 1 exit criteria (PRD §13) are checked off.

## Tasks

### 10.1 Golden runs and the run report

- **Type:** AFK
- **Blocked by:** 9.3
- **PRD reference:** §14, §15, §13 (exit criteria)
- **System design reference:** §9 observability

**Scope:** Record golden runs on Ames and churn. Produce a run report covering:
- end-to-end completion;
- generality;
- loop discipline: every loop stopped by its own rule, and no discarded change reached the final artefacts;
- planted defects caught against each answer key;
- agent claims contradicted by checks (counted);
- held-out discipline (counter = 1 per approved model);
- chat vs loop effort (turns and minutes);
- cost per run, from the gateway.

**Acceptance criteria:**
- [ ] Every report number links to its source in `experiments.jsonl`, MLflow or the logs.
- [ ] All four Phase 1 exit criteria are met and evidenced.
- [ ] Demoable in isolation.

### 10.2 README and demo script

- **Type:** AFK
- **Blocked by:** 10.1
- **PRD reference:** §13 (P1-M4)
- **System design reference:** §9 deployment

**Scope:**
- **README:** what DS-Loop-Agent is (*an agentic, loop-engineered data science workflow*); setup for Docker Compose and `start.sh`; the demo-only API key with a spend limit; how to run each dataset; and where the experiment record, MLflow and the exports live.
- **Demo script:** the PRD §14 beats, with talking points and the fallback (replay from the golden run).

**Acceptance criteria:**
- [ ] A fresh clone, following the README only, reaches a running app and a completed Ames project.
- [ ] The demo script matches the recorded golden run.
- [ ] Demoable in isolation.
