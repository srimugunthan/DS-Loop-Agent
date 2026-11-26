# System Design: DS-Loop-Agent — Phase 1 (Skills-Driven Agent)

> Derived from PRD: [`PRD-phase1-skills-agent.md`](PRD-phase1-skills-agent.md) (Draft v3).
> Every decision below was either stated by the PRD or settled through a grilling
> session with the project owner (rounds 1–4, Q1–Q31). None were assumed.
> Decision numbers (e.g. **D27**) refer to rows in [`decision-log.md`](decision-log.md).
> Where an ADR exists, it is linked instead of re-arguing the trade-off.
> Anything marked *illustrative* is an example shape, not a settled decision.

---

## 1. Overview

DS-Loop-Agent is a local, single-user web app (D15, D17) that takes a tabular dataset to a reviewed model through a fixed workflow (D1):
- a **Project** step, where a **master agent** handles intake and guidance (D42–D45);
- then five stages: **data-prep → features-gen → model-training-eval → post-modelling-analysis → model-summary**.

Each stage is worked by a skills-driven agent built on the Claude Agent SDK (D19), in one of two modes (D2):
- **chat mode** (all stages): the user instructs the agent turn by turn;
- **loop mode** (stages 1–3): the agent proposes one change per iteration under a loop program, and the platform decides whether to keep it (D6, D7, D22).

Three guarantees run through the design:
- **The agent never judges its own work.** Deterministic, platform-owned checks do (D4).
- **Nothing advances without human approval** (D3).
- **The held-out set is reachable only by the one-time stage 3 gate** (D5, D8).

The design satisfies the PRD's Phase 1 goals: the two-pane UI, both modes, review and approval, problem-agnostic operation on house prices and churn (D11–D13), and a documented agent interface that lets Phase 2 swap in AI coworkers without UI changes (D10, [ADR-001](docs/adr/ADR-001-coworker-shaped-agent-boundary.md)).

---

## 2. Component Boundaries

```
 ┌─────────────────────────── Host (127.0.0.1 only, D41) ─────────────────────────────────────┐
 │                                                                                            │
 │  ┌──────────────────────┐   SSE (events)   ┌──────────────────────────────────────────┐   │
 │  │ Browser UI (React)   │◄─────────────────│ Backend (FastAPI)                        │   │
 │  │  stepper · chat      │   REST (actions) │  Session manager                         │   │
 │  │  spec form           │─────────────────►│  Loop runner + job manager (in-process)  │   │
 │  │  file browser +      │                  │  Agent interface + skills-agent adapter  │   │
 │  │   Monaco viewer      │                  │  Dataset import / profiler / splitter    │   │
 │  │  loop panel · checks │                  │  Check orchestrator                      │   │
 │  └──────────────────────┘                  │  Script generator / exporter             │   │
 │                                            │  Project store (folders + JSONL)         │   │
 │                                            │  MLflow mirror                           │   │
 │                                            └───┬──────────────┬───────────────┬───────┘   │
 │                                 start/stop,    │              │ run check     │ mirror    │
 │                                 brief, files   │              │ (per run)     ▼           │
 │  ┌─────────────────── Docker internal network ─┼──────────────┼────────┐  ┌──────────┐    │
 │  │                                             ▼              ▼        │  │ MLflow   │    │
 │  │  ┌───────────────────────────┐   ┌─────────────────────────────┐   │  │ server   │    │
 │  │  │ Agent container           │   │ Check-runner container      │   │  └──────────┘    │
 │  │  │ (one per project)         │   │ (one per check run)         │   │                  │
 │  │  │  master + stage agents    │   │  checks · Optuna · SHAP     │   │                  │
 │  │  │  Agent SDK, Haiku 5.5     │   │  workspace read-only        │   │                  │
 │  │  │  stage workspace (rw)     │   │  no network                 │   │                  │
 │  │  │  no internet, fake key    │   │  held-out mounted ONLY for  │   │                  │
 │  │  └─────────────┬─────────────┘   │  the stage 3 gate           │   │                  │
 │  │                │ /exp/<id>       └─────────────────────────────┘   │                  │
 │  │                ▼                                                    │                  │
 │  │  ┌───────────────────────────┐                                     │                  │
 │  │  │ LLM gateway               │── HTTPS ──► api.anthropic.com       │                  │
 │  │  │ holds API key · meters $  │                                     │                  │
 │  │  │ enforces loop $ cap       │                                     │                  │
 │  │  └───────────────────────────┘                                     │                  │
 │  └────────────────────────────────────────────────────────────────────┘                  │
 └────────────────────────────────────────────────────────────────────────────────────────────┘
```

| Component | Responsibility | Why the boundary falls here |
|---|---|---|
| **Browser UI** (React, D18) | Stepper with the Project step and five stages, per-agent chat, spec form, read-only file browser and Monaco code/diff viewer (D34), loop panel (program, timeline, chart, leaderboard, budget meter), checks, approve / request changes | Presentation only. All state and decisions live in the backend; the UI can't change files (D34) |
| **Backend** (FastAPI, D18) | The platform. It owns stage state, approvals, transitions, the loop, the checks' verdicts, the project store and the MLflow mirror | Single owner of every trust decision (D3, D4, D6) |
| ↳ Session manager | Starts and stops agent sessions; builds each stage's brief from approved artefacts, `problem.yaml`, the last verdict and the master agent's labelled note (D45) | Essential inputs never depend on what an agent remembers to include |
| ↳ Loop runner + job manager | Runs loops as in-process background tasks, one loop per project (D32). Calls `propose`, validates and applies the one change, requests checks, keeps or discards, logs, enforces budgets and stop rules, saves state after each experiment (D6, D7, D31) | The loop is the platform's, not the agent's ([ADR-001](docs/adr/ADR-001-coworker-shaped-agent-boundary.md)) |
| ↳ Agent interface + adapter | `start`, `send`, `propose`, `events`, `artifacts`, `stop`, `capabilities` (D10). The skills-driven adapter drives the Agent SDK in the agent container | The seam where Phase 2 coworkers plug in |
| ↳ Dataset import / profiler / splitter | Copies the dataset into the project, fingerprints and profiles it, and after spec confirmation splits off the held-out set into a folder no agent mounts (D44) | The agent never opens arbitrary paths or sees future held-out rows |
| ↳ Check orchestrator | Launches a check-runner container per check run, passes inputs, collects `verdict.json` (D28) | Untrusted code never runs in the backend ([ADR-003](docs/adr/ADR-003-separate-check-runner-and-held-out-gate.md)) |
| ↳ Script generator / exporter | Builds `data_cleaning.py`, `features.py`, `train.py` from the itemised stage structure; export adds `pipeline.py` and `requirements.txt` (D38) | Generated views stay consistent with the single stored structure ([ADR-005](docs/adr/ADR-005-shared-stage-structure-for-chat-and-loop.md)) |
| ↳ Project store | Folders with numbered versions and JSONL logs; the source of truth (D20, D24) | [ADR-004](docs/adr/ADR-004-folders-and-jsonl-with-mlflow-mirror.md) |
| ↳ MLflow mirror | Copies experiments, runs, models and the approved-model registration to MLflow (D24) | MLflow is a view, never a dependency for correctness |
| **Agent container** (one per project) | Runs the master agent and stage agents (Agent SDK, Haiku 5.5 by default, D26). Sees only its workspace; no internet; only a placeholder API key (D27, D36) | Everything the agent does, including its own shell commands, is sandboxed ([ADR-002](docs/adr/ADR-002-agent-runs-in-per-project-container.md)) |
| **Check-runner container** (one per check run) | Executes checks, training, Optuna search (stage 3), the held-out gate and the stage 4 analysis (SHAP, segments). Workspace read-only, no network, writes only `verdict.json` and platform-owned outputs (D28, D46, D47) | Verifier is separate from proposer; held-out data is reachable only here, only at the gate ([ADR-003](docs/adr/ADR-003-separate-check-runner-and-held-out-gate.md)) |
| **LLM gateway** | Only holder of the API key. Forwards model requests, streams replies, meters tokens and $ per experiment, refuses requests once the loop cap is reached (D35, D36) | Key never in a sandbox; one enforcement point for cost ([ADR-007](docs/adr/ADR-007-llm-gateway-holds-api-key-and-meters-cost.md)) |
| **MLflow server** | Tracking UI and model registry (D24) | Mirror target only |

**Dev mode (`start.sh`, D17, D23):** the same components without Docker. The agent and the check runner are local subprocesses limited to the project folder; the held-out set sits outside the workspace; the gateway is a local process on `127.0.0.1`. The UI shows an **"Unsandboxed — dev mode"** banner. Demos and comparisons use Docker mode.

---

## 3. Data Model

| Entity | Owner | Key contents | Relationships |
|---|---|---|---|
| **Project** | Backend (project store) | Name, dataset reference, problem spec, master agent chat, five stage records | Has one Dataset, one ProblemSpec, five Stages |
| **Dataset** | Import/profiler | Imported copy, **fingerprint**, profile (types, missing values, cardinality, likely id/time/target columns) (D44) | Split into a **development set** and a **HeldOutSet** after spec confirmation |
| **HeldOutSet** | Platform only | Held-out rows, in a folder no agent container mounts (D5, D44) | Read only by the stage 3 gate run; each use counted |
| **ProblemSpec** (`problem.yaml`) | Backend; drafted by the master agent, confirmed by the user | `task_type`, `target`, `positive_class`, `id_columns`, `time_column`, `primary_metric`, `split`, `cv_folds`, `segments`, `forbidden_columns`, `domain_notes` (D11) | Read by every agent, check and loop program |
| **Stage** | Backend | Status (not started, active, looping, awaiting review, approved, stale), mode history, chat history, versions, approval record | Has Versions, LoopRuns, Experiments |
| **StageStructure** | Backend (written by the agent or the platform) | Stage 1: rule files + ordered list; stage 2: feature files + list; stage 3: model config (D29) | Source for generated scripts (D38) and the saved Pipeline (D37) |
| **Version** | Backend | Numbered, immutable snapshot of a StageStructure plus outputs; labels *approved* / *latest* (D20, D34) | Approval freezes one; re-approval upstream makes downstream **stale** (D3, D30) |
| **LoopRun** | Loop runner | Loop program (goal, metric, keep rule, budgets incl. $ cap and per-experiment limits, stop rule, stage 3 phases explore/refine), state, stop reason (D31, D35, D40, D47) | Parent of Experiments; mirrored as an MLflow parent run |
| **Experiment** (loop) / **Revision** (chat) | Loop runner / session manager | Folder: `proposal.json`, full code as run, `diff.patch`, `verdict.json`, `run.log`; stage 3 adds `search_plan.json`, `trials.csv`, per-fold CV results. Status `kept` / `discarded` / `error` / `timeout` / `invalid`. Data fingerprint, seed, folds, backend id, tokens, cost (D14, D24) | One line each in `experiments.jsonl`; MLflow child run |
| **Verdict** | Check runner (written), backend (read) | Per-check pass/fail with reasons, metrics, keep/discard (D4) | Belongs to an Experiment, Revision or Version |
| **ModelArtifact** | Check runner | Whole pipeline (stage 1 rules + stage 2 features + model) as one scikit-learn `Pipeline` saved with joblib; only for kept experiments and the champion (D25, D37) | Logged to MLflow; the approved one is registered |
| **GatePredictions** | Check runner (stage 3 gate) | Held-out predictions, targets and the spec's segment columns, in a platform-owned folder (D46) | The only held-out-derived input to stage 4 |
| **CostRecord** | LLM gateway | Tokens in/out and $ per request, attributed to an experiment id (D36) | Rolled up into the Experiment and the LoopRun budget meter |
| **HandoffNote** | Master agent (written), session manager (attached) | Short note per stage transition, labelled as the master agent's (D45) | Attached to the next stage's brief |

---

## 4. Interfaces / APIs

| Contract | Between | Protocol | Shape (high level) | Owner |
|---|---|---|---|---|
| **UI actions** | UI → backend | REST (D33) | Create project, upload or point to dataset, send message, confirm spec, start / pause / stop / resume loop, add hint, approve (with reason if a check failed), request changes, reopen a stage, export; read-only list and read of project files (confined to the project folder, never held-out data or check code, D34) | Backend |
| **UI events** | Backend → UI | Server-Sent Events (D33) | Agent text and status, tool activity, file written / changed (file watcher), loop iteration results, budget meter updates, stage status changes, errors | Backend |
| **Agent interface** | Session manager / loop runner → adapter | In-process Python interface (D10) | `start(stage, project_context)`, `send(message)`, `propose(loop_state)`, `events()`, `artifacts()`, `stop()`, `capabilities()` | Backend (adapters implement it) |
| **Agent ↔ platform exchange** | Adapter ↔ agent container | **Files only** (D19, [ADR-001](docs/adr/ADR-001-coworker-shaped-agent-boundary.md)) | In: workspace, brief (incl. handoff note), loop program, loop state, hint. Out: edited stage files (chat), `proposal.json` (loop), `problem.yaml` draft (master agent), handoff note | Backend (schemas) |
| **`proposal.json`** | Agent → loop runner | File, validated by schema (D22) | Exactly one change: stage 1 = one cleaning-rule function; stage 2 = add feature (name + function) or remove by name; stage 3 = one search plan (family, ranges, trials), tagged with the explore or refine phase (D47), plus a one-line rationale. Malformed or multi-change → `invalid` | Backend |
| **Check run** | Check orchestrator → check-runner container | Container run with mounts | In: read-only workspace (code), development data, check suite and thresholds from the spec; held-out set **only** for the gate run. Agent code receives data as **function arguments**, never paths (D28). Out: `verdict.json`; plus model file (kept/champion), `trials.csv`, GatePredictions, analysis tables as applicable | Backend |
| **Model calls** | Agent SDK → gateway → Anthropic | HTTPS, Anthropic Messages API via `ANTHROPIC_BASE_URL=http://gateway:8080/exp/<id>` (D36) | Requests forwarded unchanged with the real key substituted; streamed replies passed through unbuffered; refusal once the loop's $ cap is reached | Gateway |
| **Cost reporting** | Gateway → backend | Gateway records cost per experiment into the experiment record (D36) | Tokens and $ per experiment id; cap-reached signal to the loop runner (D35) | Gateway |
| **MLflow mirror** | Backend → MLflow | MLflow tracking API (D24) | Project+stage → experiment; LoopRun → parent run; Experiment/Revision → child run; params, metrics, status tags, folder as artefacts; kept/champion models (sklearn flavour, D37); approved model registered | Backend |

---

## 5. Data Flow

### 5.1 Intake (Project step)

```
User ──"predict churn; data at ~/data/telco.csv"──► Master agent (agent container)
                                                        │ asks the platform to import (via its output file)
Backend: import → fingerprint → profile ────────────────┘
   │ profile + small row sample (workspace)
   ▼
Master agent drafts problem.yaml ──► Backend shows spec form (agent-filled values highlighted)
   ▼
User confirms ──► Backend splits: development set → workspace │ held-out set → platform-only folder
   ▼
Stage 1 unlocks; session manager builds its brief (+ master agent's handoff note)
```

The master agent never reads the user's path itself. The platform imports, fingerprints and profiles the data, so the agent sees only the profile and a sample (D44). The held-out split happens only after the user confirms the spec, so the split strategy (time-ordered or stratified) comes from a human-confirmed `problem.yaml` (D11, D44).

### 5.2 Chat-mode revision (any stage)

```
User message ─REST─► Session manager ─send()─► Agent edits stage structure files in workspace
                                                     │ files written → file watcher → SSE → file browser
Agent reply ("Output is in stages/<stage>/v003/") ◄──┘
   ▼
Backend: snapshot revision folder (code, diff.patch) → check run → verdict.json
   ▼
UI: Checks tab (separate from the agent's claims) ─► User: Request changes | Approve
```

Chat mode edits the shared stage structure freely (D29). Every code-changing turn becomes a permanent revision with its own verdict (D24). Approving over a failed check needs a reason, which is logged (PRD §9).

### 5.3 Loop iteration (stages 1–3)

```
Loop runner ──propose(loop_state)──► Agent writes proposal.json (one change + rationale)
     │                                        model calls ─► Gateway (meters $, enforces cap)
     ▼
Validate schema ── invalid? ──► log `invalid`, next iteration
     │ valid
     ▼
Apply to a new candidate version of the stage structure (platform)
     ▼
Check-runner container: run code + checks (stage 3: Optuna search plan) under per-experiment limits
     │  timeout → `timeout`   crash → `error`
     ▼
verdict.json ──► keep rule ──► kept: becomes current state │ discarded: not promoted
     ▼
Append experiments.jsonl, write experiment folder, save loop state, mirror to MLflow, SSE update
     ▼
Stop rule / budget / $ cap? ── no ──► next iteration      yes ──► stop banner with the reason
```

One change per iteration (D7, D22). The platform applies and judges; "undo" means not promoting (D22, D24). Limits: iteration and time budgets, per-experiment 10 min / 4 GB (30 min in stage 3) (D40), and the $ cap enforced per request at the gateway (D35, D36). API failures after SDK retries log `error` and **pause** the loop; failed iterations don't consume the iteration budget, but their cost counts (D39). State is saved after every experiment, so a crash leaves the loop **Interrupted** and resumable (D31).

### 5.4 Stage 3: explore → refine → held-out gate

```
EXPLORE: one model family per iteration, small Optuna budget (e.g. 20 trials), fixed CV folds
         simplicity (one-SE) rule picks the champion
   │ explore stop rule
   ▼
REFINE:  champion family only, narrower ranges, bigger budget (e.g. 100 trials)
         keep only if gain ≥ fold spread
   │ stop rule
   ▼
GATE (once): check runner with held-out mounted → score raw held-out rows with the saved Pipeline
   │        → held-out metrics, counter += 1, save GatePredictions (preds, targets, segment columns)
   ├─ pass → user approves → stage 4
   └─ fail → "Reopen features loop" / "Continue training loop" with the reasons
```

Tuning lives inside stage 3 (D47). The loop sees CV only; held-out evaluation is a one-time gate (D8). The gate loads the exact trained `Pipeline` inside the sandbox (D37) and saves predictions so that stage 4 never needs the held-out set again (D46).

### 5.5 Post-modelling analysis (stage 4) and reopening

```
Agent requests analyses / extra cuts ──► Check runner computes (never the agent):
    per-segment metrics (held-out: from GatePredictions only) · SHAP importance & dependence
    (training CV folds) · calibration or residuals by segment · importance stability across folds
   ▼
Red-flag checks: ID/forbidden column in top features · one dominant feature ·
                 segment beyond tolerance · unstable importances
   ▼
Agent writes analysis.md interpreting computed results (every number must match)
   ├─ Approve → stage 5 (model-summary)
   └─ "Reopen stage 2 / 3 with these findings" → stale cascade (5.6), findings in that stage's brief
```

The model is frozen in this stage. The held-out counter stays at 1 (D46).

### 5.6 Approval, handoff and stale cascade

On approval the stage's version is frozen and the next stage unlocks (D3). The master agent posts a handoff summary and note, and the session manager builds the next brief deterministically (D43, D45). If an earlier stage is re-approved, every later stage is marked **stale**. The platform automatically re-runs each stale stage's current approved code and checks on the new inputs, then waits for the user to re-approve (D30).

---

## 6. Storage & Persistence

| Data | Store | Why | Retention |
|---|---|---|---|
| Stage structures, versions, generated scripts, outputs | Project folder, numbered version directories (D20) | Agents and checks work with plain files; the file browser shows the store directly ([ADR-004](docs/adr/ADR-004-folders-and-jsonl-with-mlflow-mirror.md)) | Permanent |
| Experiments and chat revisions | One folder each plus `experiments.jsonl` (D24) | A complete, re-runnable record including failures | **Nothing is deleted** (D24) |
| Model files | joblib `Pipeline` in the experiment folder (D37) | Exact trained object for the gate and the registry | Only kept experiments and the champion; discarded ones are re-buildable from their folder (D25) |
| Held-out set | Platform-only folder outside all workspaces (D44) | Reachable only by the gate run (D5, D28) | Permanent for the project |
| GatePredictions | Platform-owned folder (D46) | Stage 4 analysis without re-reading held-out data | Permanent for the project |
| Loop state | Project folder, saved after every experiment (D31) | Resume after a crash or restart | Until the loop ends (then part of the record) |
| Chat history, approvals, check results, logs | JSONL logs in the project folder (D14, D20) | Single source of truth, inspectable | Permanent |
| Cost records | Written by the gateway into the experiment record (D36) | One trusted source for tokens and $ | Permanent |
| MLflow tracking and registry | MLflow server (Docker Compose service, or a local server in dev mode) (D24) | Familiar tracking UI and model registry | Mirror; rebuildable from the folders |
| API key | `.env`, git-ignored, outside every workspace, read only by the gateway (D36) | Never in a sandbox | — |

*Illustrative* project layout (folder names are not settled decisions):

```
projects/<name>/
  dataset/            imported copy, fingerprint, profile
  problem.yaml
  stages/<stage>/vNNN/        stage structure + generated script per version
  stages/<stage>/experiments/<id>/   proposal.json, code, diff.patch, verdict.json, run.log, …
  experiments.jsonl
  logs/               chat, approvals, events (JSONL)
platform/<name>/      held-out set, GatePredictions   ← never mounted into the agent container
```

---

## 7. Non-Functional Requirements

| Requirement | Target | Source |
|---|---|---|
| Users | One user, local machine | D15 |
| Dataset size | Fits in memory with pandas: up to about 1M rows and a few hundred columns | D21 |
| Concurrency | One loop at a time per project | D32, PRD §8.7 |
| Per-experiment limits | 10 min and 4 GB by default; 30 min in stage 3; editable | D40 |
| Loop cost | $ cap per loop, default $0.25 (Haiku 5.5), enforced per request | D35, D36 |
| Model | Claude Haiku 5.5 by default, configurable per stage and mode | D26 |
| Durability | Loop state saved after every experiment; resumable after a crash | D31 |
| Problem scope | Tabular supervised learning: regression and binary classification | D12 |
| Generality | Moving from house prices to churn needs only a new dataset and spec, with no changes to prompts, skills, loop runner or UI | D11, PRD §13 (P1-M3) |

**Not set (flagged, not assumed):** no latency targets were set for UI responsiveness, agent turns or check runs, and no availability target applies to a local single-user app. Container start-up time per check run is a known cost ([ADR-003](docs/adr/ADR-003-separate-check-runner-and-held-out-gate.md)), but no budget for it was agreed. Measure it in P1-M1.

---

## 8. Security & Compliance

| Area | Design | Source |
|---|---|---|
| Data sensitivity | Public datasets only | D16 |
| Network exposure | UI, backend and MLflow bind to `127.0.0.1`; no login. Gateway only on the internal Docker network (dev mode: `127.0.0.1`) | D41 |
| Agent sandbox | Agent process in a per-project container: workspace only, no internet, placeholder key | D27, [ADR-002](docs/adr/ADR-002-agent-runs-in-per-project-container.md) |
| Verifier isolation | Checks in a separate container: workspace read-only, no network, writes only `verdict.json` and platform outputs; data passed as function arguments | D28, [ADR-003](docs/adr/ADR-003-separate-check-runner-and-held-out-gate.md) |
| Held-out protection | Split by the platform after spec confirmation, stored outside all workspaces, mounted only for the gate; uses counted; stage 4 uses saved GatePredictions only | D5, D44, D46 |
| API key | Held only by the gateway; demo-only key with a monthly spend limit in the Anthropic Console; `sk-ant-…` masked in logs before saving or mirroring | D36, [ADR-007](docs/adr/ADR-007-llm-gateway-holds-api-key-and-meters-cost.md) |
| Authority | Agents cannot approve, advance, skip checks or edit verdicts; the master agent is guide-only | D3, D4, D42, [ADR-006](docs/adr/ADR-006-master-agent-is-guide-only.md) |
| Untrusted deserialisation | joblib model files are loaded only inside sandbox containers | D37 |
| Dev mode | Isolation is weaker (local subprocesses); a UI banner makes it visible; not used for demos or comparisons | D23 |
| Audit trail | Every message, file version, loop iteration, check result, approval (with reason when overriding a failed check), backend id and cost is logged | D14, D24 |

---

## 9. Deployment & Operability

- **How it ships:** a local app. `docker compose up` starts the UI, backend, LLM gateway and MLflow, and the backend starts agent and check-runner containers on demand (D17, D27, D28). `start.sh` runs the frontend, backend, gateway and MLflow as local processes in dev mode (D17, D23).
- **Images:** the agent container image must include the Python data-science stack the skills need, because the container has no internet ([ADR-002](docs/adr/ADR-002-agent-runs-in-per-project-container.md)). The check-runner image includes the checks, Optuna and SHAP (D46, D47).
- **Configuration:** the model and effort per stage and mode (D26); loop program defaults (budgets, $ cap, per-experiment limits) (D35, D40); the gateway's per-model price table ([ADR-007](docs/adr/ADR-007-llm-gateway-holds-api-key-and-meters-cost.md)); the API key in `.env` (D36).
- **Observability:**
  - the experiment record and JSONL logs, as the primary source;
  - MLflow, for browsing experiments and models;
  - the UI's budget meter, which shows live cost from the gateway;
  - stop banners that state why a loop stopped;
  - the held-out counter.
- **Rollback:** project state is versioned and nothing is deleted. "Rolling back" a stage means re-approving an earlier version, which triggers the stale cascade (D3, D20, D30). App rollback is redeploying a previous image or commit; there is no hosted deployment.

---

## 10. Failure Modes

| Failure | Type | Behaviour | Source |
|---|---|---|---|
| Model API error or rate limit | Hard dependency for agents | SDK retries with backoff. Then: experiment `error`, loop **paused** with Resume; chat shows Retry. Failed iterations don't consume the iteration budget; cost counts | D39 |
| LLM gateway down | Hard dependency for agents | Model calls fail and are handled as API errors (pause and resume) | [ADR-007](docs/adr/ADR-007-llm-gateway-holds-api-key-and-meters-cost.md) |
| $ cap reached | Expected stop | Gateway refuses further requests; loop runner stops the loop with that reason | D35, D36 |
| Agent writes a malformed or multi-change proposal | Degrades gracefully | Experiment `invalid`; loop continues | D22, D24 |
| Agent code crashes | Degrades gracefully | Experiment `error`; loop continues | D24 |
| Experiment exceeds time or memory | Degrades gracefully | Experiment `timeout`; loop continues | D40 |
| Backend crash or restart mid-loop | Recoverable | Loop shows **Interrupted**; Resume continues from the last finished experiment with the remaining budget; the in-progress experiment is logged `error` | D31 |
| MLflow down or out of sync | Degrades gracefully | Folders + JSONL remain the source of truth; the mirror can be replayed | D24, [ADR-004](docs/adr/ADR-004-folders-and-jsonl-with-mlflow-mirror.md) |
| Held-out gate fails | Expected outcome | Offer "Reopen features loop" or "Continue training loop" with the reasons | PRD §8.6 |
| Stage 4 red flag | Expected outcome | Offer "Reopen stage 2 / 3 with these findings"; stale cascade | D46, D30 |
| Master agent drafts a wrong spec | Mitigated | Form with agent-filled values highlighted; user must confirm before the split | D44, [ADR-006](docs/adr/ADR-006-master-agent-is-guide-only.md) |
| Upstream re-approval | Expected | Later stages stale, auto re-run and re-checked, then await re-approval | D30 |

---

## 11. Open Risks / Deferred Decisions

These were explicitly left open or surfaced while writing this design. None are assumed above.

| Item | Status | Where it is tracked |
|---|---|---|
| Loop budget defaults (stage 2: 15 iterations / 20 min; stage 3: 12 iterations / 30 min) — smaller for live demos? | Open product question | PRD Phase 1 §17.2 |
| Hints during a running loop — allowed, or hands-off? | Open product question (the PRD currently allows them) | PRD Phase 1 §17.3 |
| Exact `proposal.json` schemas per stage | To define in implementation | [ADR-001](docs/adr/ADR-001-coworker-shaped-agent-boundary.md) follow-up |
| Project folder layout and REST endpoint names | Implementation detail; §6 shows an *illustrative* layout only | — |
| Container start-up cost per check run | Accepted; to measure in P1-M1 | [ADR-003](docs/adr/ADR-003-separate-check-runner-and-held-out-gate.md) |
| Disk growth: nothing is deleted; model files are kept only for kept experiments and the champion | No retention limit set; acceptable at the demo's scale | D24, D25 |
| Gateway price table must be updated when model prices change | Accepted | [ADR-007](docs/adr/ADR-007-llm-gateway-holds-api-key-and-meters-cost.md) |
| Repeated CV iterations still overfit the folds slightly | Accepted and mitigated (min gain, one-SE rule, one-time gate) | PRD Phase 1 §8.6 |
| Haiku 5.5 may give weak proposals in some stages | Mitigated by per-stage model configuration | D26 |
| Subtle planted leaks (caught in stage 4) — exact design per dataset | To design with the answer keys | PRD Phase 1 §11.2 |
| Phase 2: may a coworker replace the master agent? Same-model comparisons on Haiku vs. a larger model? | Deferred to Phase 2 | PRD Phase 2 §15, Q6–Q7 |
| Multi-user, hosted deployment | Out of scope for Phase 1; the folder store (ADR-004) and the in-process job manager (D32) would need revisiting | D15 |

---

## 12. Decisions & ADRs

The complete decision record is [`decision-log.md`](decision-log.md) (47 decisions). It is not duplicated here.

Decisions that cleared the three-part ADR bar (hard to reverse, surprising without context, a real trade-off):

| ADR | Title | Decisions |
|-----|-------|-----------|
| [ADR-001](docs/adr/ADR-001-coworker-shaped-agent-boundary.md) | Agents exchange files with the platform, not tool calls | D10, D19, D22 |
| [ADR-002](docs/adr/ADR-002-agent-runs-in-per-project-container.md) | Run each project's agent process inside its own container | D27 |
| [ADR-003](docs/adr/ADR-003-separate-check-runner-and-held-out-gate.md) | Run checks in a separate check-runner container; only the stage 3 gate sees held-out data | D5, D28, D44 |
| [ADR-004](docs/adr/ADR-004-folders-and-jsonl-with-mlflow-mirror.md) | Plain folders and JSONL are the source of truth; MLflow is a mirror | D20, D24 |
| [ADR-005](docs/adr/ADR-005-shared-stage-structure-for-chat-and-loop.md) | Chat and loop modes edit one shared, itemised stage structure | D29 |
| [ADR-006](docs/adr/ADR-006-master-agent-is-guide-only.md) | The master agent guides but holds no authority over approvals, transitions or checks | D42, D45 |
| [ADR-007](docs/adr/ADR-007-llm-gateway-holds-api-key-and-meters-cost.md) | An LLM gateway holds the API key and meters cost per experiment | D36 |
