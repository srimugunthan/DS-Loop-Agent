# PRD: DS-Loop-Agent — An agentic, loop-engineered data science workflow

The PRD is split into two phases:

| Phase | Document | Agent | Summary |
|---|---|---|---|
| **1** | [PRD-phase1-skills-agent.md](../PRD-phase1-skills-agent.md) (Draft v3) | Skills-driven agent only | The full app: two-pane UI with a file browser and code viewer; a **master agent** for project intake and guidance between stages; five stage agents (including post-modelling analysis); chat mode and loop mode; objective checks; review and approval; problem spec; house prices and churn. Ends with a documented agent interface |
| **2** | [PRD-phase2-ai-coworkers.md](PRD-phase2-ai-coworkers.md) (Draft v2) | AI coworkers (OpenHands, Goose, Grok Bot, optionally Hermes) | Adapters on the Phase 1 interface, agent selector for the stage agents, manual mode for coworkers without an API, comparison view, experiment protocol, safety for third-party agents (coworkers reuse the Phase 1 LLM gateway, so they never hold the API key) |

Phase 2 starts only when Phase 1's exit criteria are met (Phase 1 PRD, section 13).

**System design (Phase 1):** decisions are recorded in [decision-log.md](../decision-log.md), with the significant ones as ADRs in [docs/adr/](../docs/adr/). The design itself is [system-design.md](../system-design.md) (draft, pending your confirmation).

**History:**
- 2026-11-26: the single-document draft v2 was split into the two phases. No requirements were dropped. Phase 1 added the skills to write (10.2) and Phase 2 readiness (10.3). Phase 2 added the experiment protocol (9) and coworker safety (10).
- 2026-11-27: Phase 1 Draft v3 adds the post-modelling-analysis stage (five stages) and explore → refine tuning in stage 3.
- 2026-11-27: Phase 1 Draft v2 adds the master agent and the file browser, and aligns with the system design decisions. Phase 2 Draft v2 follows from those changes.
