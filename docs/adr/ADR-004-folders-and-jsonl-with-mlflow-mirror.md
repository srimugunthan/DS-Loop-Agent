# ADR-004: Plain folders and JSONL are the source of truth; MLflow is a mirror

**Status:** Accepted
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

Every message, file version, loop iteration, check result and approval must be logged (decision 14). The user wants every experiment kept, including failures, with the exact code that ran, plus a log of all experiments (grilled Q8). The user also wants MLflow's tracking UI and model registry. Agents work through files (ADR-001), and the right pane is a file browser over the project folder (decision 34). The app is local and single-user (decision 15).

---

## Decision

We will store each project as plain folders with numbered version directories plus JSONL logs. Every experiment and chat revision gets a permanent folder:
- `proposal.json`;
- the full code as it ran;
- `diff.patch`;
- `verdict.json`;
- `run.log`;
- stage 3 also gets `search_plan.json`, `trials.csv` and per-fold CV results.

Each experiment also gets one line in `experiments.jsonl` with status `kept`, `discarded`, `error`, `timeout` or `invalid`. Nothing is deleted. Model files are kept only for kept experiments and the champion (decision 25).

The platform mirrors this record to MLflow:
- project + stage → MLflow experiment;
- loop → parent run;
- experiment → child run;
- the approved model is registered.

Agents never write to MLflow.

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Folders + JSONL, mirrored to MLflow (chosen)** | The file browser shows the store directly; agents and checks work with plain files; easy to inspect, copy and export; MLflow gives a familiar UI without being a dependency for correctness | Two representations to keep consistent; no query engine (lists come from scanning JSONL); concurrent writers need care |
| Git repo, one commit per approved stage or kept iteration | Built-in diffs and history | Discarded experiments don't fit commit history well; awkward for large data files; adds git semantics to "undo" |
| SQLite as the primary store | Queries and transactions | Code and artefacts still live in files, so the store is split anyway; the file browser would need a translation layer |
| MLflow as the primary store | One system | The platform's own rules (statuses, versions, approvals, held-out counter) don't map cleanly onto it; a dependency for every write |

---

## Consequences

**Positive:**
- Every experiment, including failures, can be re-run from its folder: the data fingerprint, seed and folds are recorded.
- Resuming an interrupted loop (decision 31) reads state straight from the folders.
- If MLflow is down or rebuilt, nothing is lost. The mirror can be replayed.

**Negative / Accepted Tradeoffs:**
- Mirror lag or failure means MLflow can briefly disagree with the folders. The folders win.
- The dataset size envelope (up to about 1M rows, in memory, decision 21) and single-user use keep the file approach workable. Multi-user would need revisiting.

---

## Follow-up Actions

- [ ] Make the MLflow mirror replayable from `experiments.jsonl`, for rebuilds.
