# Phase 8: MLflow Mirror and Export

**Depends on:** `phase6-training-tuning-and-gate.md` (can run in parallel with `phase7-analysis-and-summary.md`)
**Status:** Not started

## Design components touched

MLflow mirror and server (§2, §4; ADR-004, D24), model logging and registry (D24, D37), exporter (§2; D38), key masking before mirroring (D36).

## Objective

Everything in the experiment record is mirrored to MLflow by the platform. The agent never writes to MLflow. Mapping:
- project + stage → MLflow experiment;
- loop → parent run;
- experiment or revision → child run, with params, metrics, status tags and the folder as artefacts.

Kept and champion models are logged with the sklearn flavour, and the approved model is registered. The mirror can be replayed from `experiments.jsonl`. Export produces a take-away bundle whose `pipeline.py` runs stages 1–3 end to end on a raw CSV.

## Independently testable when

- After a stage 2 and a stage 3 loop on Ames, the MLflow UI shows the loops as parent runs with one child run per experiment, including failures, with the correct status tags.
- The approved model appears in the registry.
- Wiping MLflow and replaying rebuilds the same runs.
- `python pipeline.py raw.csv` from the export reproduces the Pipeline's predictions.

## Tasks

### 8.1 MLflow service and mirror, with replay

- **Type:** AFK
- **Blocked by:** 5.3
- **PRD reference:** §8.8 (MLflow mirror)
- **System design reference:** §4 MLflow mirror, §10; ADR-004, D24

**Scope:**
- **Service:** MLflow runs as a Docker Compose service, or a local server from `start.sh`, bound to localhost.
- **Mirror:** applies the mapping above. Logs are masked before mirroring (2.6).
- **Replay:** rebuilds MLflow from `experiments.jsonl`.
- **Failure handling:** a failing mirror never blocks the loop.

**Acceptance criteria:**
- [ ] Every status (`kept`, `discarded`, `error`, `timeout`, `invalid`) appears as a tagged child run.
- [ ] Stopping MLflow mid-loop doesn't stop the loop; replay catches up.
- [ ] Demoable in isolation.

### 8.2 Model logging and registration

- **Type:** AFK
- **Blocked by:** 8.1, 6.5
- **PRD reference:** §8.8
- **System design reference:** D24, D37

**Scope:** Log kept and champion Pipelines with MLflow's sklearn flavour. On stage 3 approval, register the approved model with the gate metrics and the held-out counter as metadata.

**Acceptance criteria:**
- [ ] The registered model loads and scores raw rows identically to the stored Pipeline.
- [ ] Discarded experiments have no logged model.
- [ ] Demoable in isolation.

### 8.3 Export bundle with `pipeline.py`

- **Type:** AFK
- **Blocked by:** 6.4
- **PRD reference:** §7 step 10
- **System design reference:** §2 exporter; D38

**Scope:** Export the approved artefacts as a bundle:
- the per-stage generated scripts;
- `pipeline.py`, which runs stages 1–3 end to end on a raw CSV using the approved model;
- `requirements.txt`;
- metrics, the summary (when stage 5 exists), loop logs and the review log.

**Acceptance criteria:**
- [ ] In a clean virtualenv, `pip install -r requirements.txt` followed by `python pipeline.py raw.csv` reproduces the Pipeline's predictions.
- [ ] The bundle contains no held-out data and no API key.
- [ ] Demoable in isolation.
