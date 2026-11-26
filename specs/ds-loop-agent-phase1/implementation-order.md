# Implementation Order

Phases must be implemented in this order. The order comes from the system design's technical dependency graph (shared infrastructure before the features that consume it; the sandbox before anything that runs agent code; checks before loops; the stage 3 gate before anything that uses its outputs), **not** from PRD feature priority. Each phase is independently testable before moving to the next.

| Order | Phase file | Depends on | Design components | Independently testable when |
|-------|-----------|------------|--------------------|------------------------------|
| 1 | `phase1-foundation.md` | None | Backend (FastAPI), project store, REST/SSE, browser UI, file browser + Monaco (§2, §4, §6; D18, D20, D33, D34) | A fake agent writes files that appear live in the tree and open in viewer tabs |
| 2 | `phase2-agent-sandbox-and-gateway.md` | Phase 1 | Agent container, LLM gateway, agent interface + adapter (§2, §4; ADR-001, ADR-002, ADR-007) | Real agent chat works; the container has no internet and no real key; cost is recorded per turn |
| 3 | `phase3-intake-and-problem-spec.md` | Phase 2 | Import/profiler/splitter, master agent, spec form (§5.1; D42–D44; ADR-006) | Raw Ames CSV → confirmed `problem.yaml`, held-out set in a platform-only folder |
| 4 | `phase4-checks-and-chat-stages-1-2.md` | Phase 3 | Check-runner container, stage structure, script generator, approvals, handoff, stale cascade (§5.2, §5.6; ADR-003, ADR-005) | Stages 1–2 in chat on Ames: planted defect flagged, approve, handoff, stale cascade |
| 5 | `phase5-loop-engine.md` | Phase 4 | Loop runner + job manager, `proposal.json`, budgets, gateway cap (§5.3; D6, D7, D22, D31, D35, D39, D40) | Features loop discards the planted leak and stops itself; resume after a backend kill |
| 6 | `phase6-training-tuning-and-gate.md` | Phase 5 | Stage 3 config, Optuna in the check runner, Pipeline files, held-out gate (§5.4; D8, D37, D47) | Approved champion with held-out counter = 1; a forced gate failure offers reopen |
| 7 | `phase7-analysis-and-summary.md` | Phase 6 | Analysis in the check runner, red flags, reopen, summary checks (§5.5; D46) | Subtle leak flagged as the top feature; reopen works; summary numbers trace to the logs |
| 8 | `phase8-mlflow-and-export.md` | Phase 6 | MLflow mirror, exporter (§4; ADR-004; D24, D38) | MLflow shows loop/experiment runs and the registered model; exported `pipeline.py` runs on a raw CSV |
| 9 | `phase9-second-problem-churn.md` | Phases 7 and 8 | Classification check variants; datasets (D11–D13) | Churn end to end in both modes with no changes to prompts, skills, loop runner or UI |
| 10 | `phase10-showcase.md` | Phase 9 | Observability outputs (§9) | The ~9-minute showcase recorded on both problems |

## Dependency notes

- **Phase 8 can run in parallel with Phase 7.** The MLflow mirror (8.1) needs only the experiment record from the loop runner (5.3). Model logging (8.2) needs Pipeline files and the gate (6.4, 6.5). Export (8.3) needs Pipeline files (6.4). None of it needs post-modelling analysis.
- **Dataset preparation (3.1) has no code dependencies.** It can be done at any time before Phase 3 and in parallel with Phases 1–2.
- **The subtle-leak design (7.1, HITL) depends only on the Ames dataset (3.1).** The decision can be made early, during Phase 4 or 5, so Phase 7 isn't blocked waiting for it.
- **Classification is deliberately late (Phase 9).** Phases 1–8 run on the regression problem. Phase 9 then tests whether anything was hardcoded, which is the PRD's generality criterion (P1-M3). Shared check code should still read everything from `problem.yaml` from the start.
- **The PRD milestones are coarser than these phases:** P1-M1 = Phases 1–5, P1-M2 = Phases 6–8, P1-M3 = Phase 9, P1-M4 = Phase 10.
- Apart from the overlaps above, phases are strictly sequential.

## Do not reorder

Do not start a phase before its listed dependencies are complete and verified. If a later phase shows that an earlier phase's design assumption was wrong, stop and flag it. Don't silently patch it forward: a design change goes back through `decision-log.md`, and an ADR if it clears the bar.
