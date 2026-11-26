# DS-Loop-Agent Phase 1 — Overview

## Source documents

- **PRD:** [`../../PRD-phase1-skills-agent.md`](../../PRD-phase1-skills-agent.md) (Draft v3)
- **System design:** [`../../system-design.md`](../../system-design.md), with [`../../decision-log.md`](../../decision-log.md) and [`../../docs/adr/`](../../docs/adr/)

## Goal

DS-Loop-Agent is *an agentic, loop-engineered data science workflow*. A user chats with a **master agent**, which takes the problem and the dataset, drafts the problem spec and guides them through five stages: data-prep, features-gen, model-training-eval, post-modelling-analysis and model-summary.

A skills-driven agent built on the Claude Agent SDK does the work in each stage, in **chat mode** (all stages) or **loop mode** (stages 1–3, one change per iteration). Platform-owned checks judge every output, nothing advances without human approval, and the held-out set is reachable only by the one-time stage 3 gate.

Phase 1 proves three things:
- the workflow and both modes work with a skills-driven agent;
- loop engineering is visible and works;
- the tool is problem-agnostic, shown on house prices (regression) and customer churn (classification).

It also leaves a clean agent interface for Phase 2's AI coworkers.

## Scope

- Two-pane local web app (FastAPI + React): stepper with the Project step and five stages, per-agent chat, read-only file browser and Monaco code/diff viewer.
- Master agent: intake, problem spec drafting, verdict explanations, handoff notes. Guide only.
- Five stage agents; chat mode for all, loop mode for stages 1–3.
- Agent container per project, LLM gateway (key holder, cost metering, $ cap), check-runner container (checks, Optuna, SHAP, held-out gate).
- Shared itemised stage structure, generated scripts, export with `pipeline.py`.
- Experiment record (folders + JSONL, nothing deleted) mirrored to MLflow.
- Stage 3 explore → refine tuning, one-SE rule, held-out gate with counter.
- Post-modelling analysis with platform-computed segments and SHAP, red flags and reopening.
- Two problems: Ames Housing and IBM Telco Churn, with planted defects and answer keys.
- Docker Compose mode, plus a `start.sh` dev mode without Docker.

## Non-goals

- AI coworkers, the agent selector and agent comparisons (Phase 2).
- Deployment, canary rollout, monitoring and retraining.
- Fully autonomous runs across stages; the master agent never approves or advances.
- Loop mode for stages 4 and 5.
- Multi-user collaboration, login, or hosting beyond localhost.
- Production or sensitive data (public datasets only).
- Problem types beyond tabular supervised learning.

## Phases

| Phase | File | Summary | Depends on |
|-------|------|---------|------------|
| 1 | `phase1-foundation.md` | Backend, project store, REST + SSE, UI shell, file browser and viewer, with a fake agent; `start.sh` | None |
| 2 | `phase2-agent-sandbox-and-gateway.md` | Agent container + LLM gateway; real agent chat through the agent interface; cost metering; dev mode | Phase 1 |
| 3 | `phase3-intake-and-problem-spec.md` | Ames dataset + answer key; import, profile; master agent drafts the spec; spec form; held-out split | Phase 2 |
| 4 | `phase4-checks-and-chat-stages-1-2.md` | Check runner; stages 1–2 in chat with checks, approval, handoff, stale cascade | Phase 3 |
| 5 | `phase5-loop-engine.md` | Loop program, `proposal.json`, loop runner, budgets and $ cap, timeline/chart, pause/resume; loops for stages 1–2 | Phase 4 |
| 6 | `phase6-training-tuning-and-gate.md` | Stage 3: model config, Optuna search plans, explore → refine, Pipeline files, held-out gate | Phase 5 |
| 7 | `phase7-analysis-and-summary.md` | Subtle leak design (HITL); post-modelling analysis with red flags and reopening; model summary | Phase 6 |
| 8 | `phase8-mlflow-and-export.md` | MLflow mirror and model registry; export bundle with `pipeline.py` | Phase 6 (parallel with Phase 7) |
| 9 | `phase9-second-problem-churn.md` | Classification checks; Telco dataset; generality run with no code changes | Phases 7 and 8 |
| 10 | `phase10-showcase.md` | Golden runs, run report, README and demo script | Phase 9 |

See `implementation-order.md` for the full dependency detail and `README.md` for how to work through this folder.
