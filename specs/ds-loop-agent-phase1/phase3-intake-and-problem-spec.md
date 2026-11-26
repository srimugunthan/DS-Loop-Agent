# Phase 3: Intake and Problem Spec

**Depends on:** `phase2-agent-sandbox-and-gateway.md`
**Status:** Not started

## Design components touched

Dataset import / profiler / splitter (§2, §5.1; D44), master agent (§2; D42, D43; ADR-006), problem spec (§3 ProblemSpec; D11), held-out protection (§8; ADR-003, D5).

## Objective

The Project step works end to end:
1. The user tells the master agent what to predict and where the data is.
2. The platform imports, fingerprints and profiles the dataset.
3. The master agent drafts `problem.yaml` from the profile and a small row sample.
4. The user reviews it in a form, with agent-filled values highlighted, and confirms.
5. The platform splits off the held-out set into a folder no agent can reach, and stage 1 unlocks.

The Ames dataset with planted defects and its answer key exists for later phases to test against.

## Independently testable when

- With the raw Ames CSV, the master agent drafts a spec with a time-ordered split on `YrSold`/`MoSold`, the user confirms, and stage 1 unlocks.
- The held-out set exists only in the platform folder: it is absent from every workspace, absent from the agent container's mounts, and never shown in the file browser.
- The answer key lists every planted defect.

## Tasks

### 3.1 Ames dataset with planted defects and answer key

- **Type:** AFK
- **Blocked by:** None
- **PRD reference:** §11.2
- **System design reference:** §11 (answer keys)

**Scope:** Build the demo Ames dataset following `AGENT_DS_WORKFLOW_BRAINSTORMING.md` §2.4. Plant the defects: a $50 price typo, sqft/sqm mixing, nulls, duplicates, and the neighbourhood-mean-price leak. Write an answer key listing each defect, its location and the expected catching check. Not included: the subtle stage-4 leak (7.1).

**Acceptance criteria:**
- [ ] Dataset and answer key are versioned under a fixtures folder.
- [ ] A script reproduces the dataset from the public source.
- [ ] Verifiable in isolation.

### 3.2 Import from a path or upload, with fingerprint and profile

- **Type:** AFK
- **Blocked by:** 1.3, 3.1
- **PRD reference:** §6.1 (dataset intake), §7 steps 1–2
- **System design reference:** §5.1; D21, D44

**Scope:** The platform, not the agent, copies a CSV from a given path or an upload into the project and records a fingerprint. It profiles the data (types, missing values, cardinality, likely id/time/target columns) and shows the profile in the right pane. Datasets up to about 1M rows load in memory.

**Acceptance criteria:**
- [ ] The import happens in backend code; the agent never opens the user's path.
- [ ] The fingerprint is stable across re-imports of the same file; the profile flags `YrSold`/`MoSold` as likely time columns for Ames.
- [ ] Demoable in isolation.

### 3.3 Master agent drafts `problem.yaml`

- **Type:** AFK
- **Blocked by:** 2.3, 3.2
- **PRD reference:** §5.0, §6.1
- **System design reference:** §5.1; D42, D43; ADR-006

**Scope:**
- **Project step:** the master agent's chat, with its agent card.
- **`ds-master` skill:** intake questions, drafting from a profile, leak-prone columns, split choice, and the "guide only" rule.
- **Flow:** the master agent asks for the problem and the data location. The platform runs the import (3.2) and puts the profile and a small sample in the master agent's workspace. The agent writes a draft `problem.yaml` as a file.

Not included: the confirmation form (3.4).

**Acceptance criteria:**
- [ ] The draft for Ames sets regression, `SalePrice`, `rmse` and a time-ordered split.
- [ ] The master agent has no way to approve or advance: no such capability exists in its interface.
- [ ] Demoable in isolation.

### 3.4 Spec form: highlight, edit, validate, confirm

- **Type:** AFK
- **Blocked by:** 3.3
- **PRD reference:** §5.0, §11.1
- **System design reference:** §3 ProblemSpec; D11, D44

**Scope:** The right pane shows the draft spec as a form, with agent-filled values highlighted. The user edits and confirms. Validation covers: the target exists, the metric fits the task type, the split's time column exists, forbidden columns exist, and segments exist. The confirmed spec is saved and shown on review screens.

**Acceptance criteria:**
- [ ] Invalid specs can't be confirmed, with a clear reason given.
- [ ] Edits and confirmation are logged.
- [ ] Demoable in isolation.

### 3.5 Held-out split on confirmation; stage 1 unlocks

- **Type:** AFK
- **Blocked by:** 3.4
- **PRD reference:** §6.1, §7 step 3
- **System design reference:** §5.1, §6, §8; ADR-003, D5, D44

**Scope:** On confirmation, the platform splits the data per the spec (time-ordered: last period held out; stratified random: e.g. 20%). The development set goes into the workspace; the held-out set goes into the platform-only folder. The held-out counter is initialised at 0, and stage 1 becomes active.

**Acceptance criteria:**
- [ ] Test: the held-out folder is not mounted in the agent container and not listed by the file endpoints.
- [ ] The split is reproducible (seed recorded) and matches the spec's strategy.
- [ ] Demoable in isolation.
