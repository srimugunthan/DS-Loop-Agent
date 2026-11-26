# DS-Loop-Agent — Phase 1 Specs

Spec-driven implementation plan for **DS-Loop-Agent Phase 1** (the skills-driven agent), generated from the PRD and `system-design.md`. This folder is the source of truth for implementation. Each phase file is self-contained and independently testable.

## Contents

- **`overview.md`**: goal, scope, non-goals, and a one-line summary of each phase.
- **`implementation-order.md`**: the phase dependency order and why it's ordered that way. Read this before starting implementation.
- **`phase1-foundation.md` … `phase10-showcase.md`**: one file per phase, in implementation order. Each contains the phase's objective, what "independently testable" means for it, and its vertical-slice tasks with acceptance criteria.

| # | File |
|---|---|
| 1 | `phase1-foundation.md` |
| 2 | `phase2-agent-sandbox-and-gateway.md` |
| 3 | `phase3-intake-and-problem-spec.md` |
| 4 | `phase4-checks-and-chat-stages-1-2.md` |
| 5 | `phase5-loop-engine.md` |
| 6 | `phase6-training-tuning-and-gate.md` |
| 7 | `phase7-analysis-and-summary.md` |
| 8 | `phase8-mlflow-and-export.md` |
| 9 | `phase9-second-problem-churn.md` |
| 10 | `phase10-showcase.md` |

## How to use this folder

1. Read `overview.md` for context, then `implementation-order.md` for the sequence.
2. Open `phase1-foundation.md` and implement its tasks in the order listed.
3. Verify the phase against its "Independently testable when" section before moving on.
4. Repeat for each later phase file, in order. Do not skip ahead: later phases assume earlier phases are done and verified. (The one allowed overlap, phase 8 alongside phase 7, is explained in `implementation-order.md`.)

**References in task headers:**
- **D#** is a row in `decision-log.md`.
- **ADR-00N** is a file in `docs/adr/`.
- **§** under "System design reference" is a section of `system-design.md`.
- **§** under "PRD reference" is a section of `PRD-phase1-skills-agent.md`.

## Provenance

- PRD: [`../../PRD-phase1-skills-agent.md`](../../PRD-phase1-skills-agent.md) (Draft v3)
- System design: [`../../system-design.md`](../../system-design.md), with [`../../decision-log.md`](../../decision-log.md) (D1–D47) and [`../../docs/adr/`](../../docs/adr/) (ADR-001 to ADR-007)
- Generated: 2026-11-27
