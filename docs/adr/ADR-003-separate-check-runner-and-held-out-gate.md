# ADR-003: Run checks in a separate check-runner container; only the stage 3 gate sees held-out data

**Status:** Accepted
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

The product's central claim is that agent output is trusted only when it passes objective checks that the agent cannot edit or overrule (PRD Phase 1 §1 and §9, decision 4). Checks must *run* the agent's code (e.g. compute a proposed feature, train a candidate model), so they can't run inside the backend. The held-out set must stay hidden from every agent and be used only by the one-time stage 3 gate (decisions 5, 8). If the held-out set leaked into anything the agent could later read, it would turn into a tuning set.

---

## Decision

We will run every check in a separate check-runner container, one per check run:
- the workspace is mounted read-only, and there is no network;
- results are written only to `verdict.json` in a platform-owned folder;
- the held-out set is mounted only for the stage 3 gate run;
- agent code receives data as function arguments, never as file paths.

The platform imports the dataset and splits off the held-out set, after the user confirms `problem.yaml`, into a folder that no agent container mounts (decision 44). The pipeline is loaded from its joblib file only inside this sandbox (decision 37). In dev mode the check runner is a separate local subprocess.

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Separate check-runner container per run (chosen)** | The agent never sees check code, thresholds or held-out data; agent code can't write held-out rows anywhere the agent can read later; untrusted code never runs in the backend | A container starts for each check run; a code path is needed to pass data in as arguments |
| Checks inside the agent container | Fast, no second container | The agent can read and change check code and results files; held-out data would have to be in its filesystem |
| Checks inside the backend process | Simplest | Runs untrusted agent code next to secrets and the held-out set |

---

## Consequences

**Positive:**
- "The agent cannot overrule the verdict" is enforced by the infrastructure, not just by the prompt.
- The held-out counter means something, because the gate is the only place that can read held-out data.
- The same guarantee holds for Phase 2 coworkers with no extra work.

**Negative / Accepted Tradeoffs:**
- Starting a container for each experiment adds seconds per loop iteration.
- Per-experiment limits (10 minutes and 4 GB; 30 minutes for stage 3) must be enforced on this container (decision 40).
- The Optuna tuner for stage 3 runs here, so long searches live in the check runner, not the agent.
- The gate run also saves held-out predictions, targets and the spec's segment columns to a platform-owned folder. The post-modelling-analysis stage (decision 46) computes held-out segment results from that saved file, so it never re-reads the held-out set and the counter stays at 1. Its SHAP analysis uses the training data's CV folds.
