# Phase 4: Checks and Chat-Mode Stages 1–2

**Depends on:** `phase3-intake-and-problem-spec.md`
**Status:** Not started

## Design components touched

Check-runner container + orchestrator (§2, §4 check run; ADR-003, D28), shared stage structure (§3 StageStructure; ADR-005, D29), script generator (D38), experiment/revision record (§3; D24), approvals and versions (§5.6; D3), handoff (§5.6; D45), stale cascade (D30).

## Objective

Stages 1 (data-prep) and 2 (features-gen) work in chat mode on Ames:
- The agent edits the itemised stage structure; the platform generates the readable script.
- Every code-changing turn becomes a revision that the check runner judges.
- The Checks tab shows the verdict separately from the agent's claims.
- The user approves, which freezes a version, or requests changes.
- The master agent's handoff note appears in the next stage.
- Re-approving stage 1 makes stage 2 stale, and stage 2 is automatically re-run and re-checked.

## Independently testable when

- In stage 1 chat, a planted Ames defect is flagged by the Checks tab even when the agent says it's done.
- Approving stage 1 freezes a version and opens stage 2 with the handoff note.
- In stage 2 chat, the planted neighbourhood-mean leak fails the leakage check.
- Re-approving stage 1 marks stage 2 stale, re-runs its checks, and waits for re-approval.

## Tasks

### 4.1 Check-runner container

- **Type:** AFK
- **Blocked by:** 2.2, 3.5
- **PRD reference:** §9 (objective checks run outside the agent)
- **System design reference:** §2 check runner, §4 check run; ADR-003, D28

**Scope:** The check orchestrator starts a container per check run:
- the workspace is mounted read-only, and there is no network;
- the development data and the check suite plus thresholds come from the spec;
- agent code is imported and called with data as **function arguments**;
- results are written to `verdict.json` in a platform-owned folder.

Dev mode uses a separate local subprocess. Not included: the held-out mount (6.5).

**Acceptance criteria:**
- [ ] Test: agent code that tries to open files outside its arguments or reach the network fails.
- [ ] The check code and thresholds are not readable from the agent container.
- [ ] Verifiable in isolation with a trivial check suite.

### 4.2 Stage 1 in chat: rules structure and generated `data_cleaning.py`

- **Type:** AFK
- **Blocked by:** 4.1
- **PRD reference:** §6.2 (stage 1), §7
- **System design reference:** §5.2; ADR-005, D24, D29, D38

**Scope:**
- **Stage structure:** a folder of rule files plus an ordered list.
- **`ds-data-prep` skill:** profiling, a cleaning-rule catalogue, "never drop rows silently", fit-on-train-only.
- **Agent behaviour:** in chat, the agent adds and edits rules.
- **Platform:** generates `data_cleaning.py` (functions in list order, `run(df)`, a header with the version and hash), and snapshots each code-changing turn as a revision folder with `diff.patch`.

**Acceptance criteria:**
- [ ] Generated script output equals applying the rules in order (test).
- [ ] Each revision folder holds the full code as it ran, plus its diff.
- [ ] Demoable in isolation.

### 4.3 Stage 1 checks and the Checks tab

- **Type:** AFK
- **Blocked by:** 4.2
- **PRD reference:** §6.2, §9
- **System design reference:** §5.2; D4

**Scope:** Stage 1 checks, configured from the spec: schema, null rates, ranges and outliers, duplicates, row count (no silent drops), imputers fitted on training data only, and the target untouched. The Checks tab shows pass/fail with reasons, visually separate from the agent's message. A failed check is prominent even if the agent says "done".

**Acceptance criteria:**
- [ ] On Ames, each planted stage-1 defect in the answer key is caught, or left flagged until it's fixed.
- [ ] Check results are logged per revision.
- [ ] Demoable in isolation.

### 4.4 Approve, request changes, send check results

- **Type:** AFK
- **Blocked by:** 4.3
- **PRD reference:** §7 step 8, §9
- **System design reference:** §5.6; D3

**Scope:**
- **Approve:** freezes the current version and labels it approved (using 1.6). Approving over a failed check requires a reason, which is logged.
- **Request changes:** returns to chat.
- **"Send check results to agent":** passes the failures to the agent as a message.

**Acceptance criteria:**
- [ ] An approved version is immutable; later edits create new versions.
- [ ] An override reason is required and logged.
- [ ] Demoable in isolation.

### 4.5 Handoff: platform-built brief plus the master agent's note

- **Type:** AFK
- **Blocked by:** 4.4
- **PRD reference:** §5.0, §6.1 (handoffs)
- **System design reference:** §5.6; D43, D45

**Scope:** On approval, the session manager builds stage 2's brief deterministically from the approved artefacts, `problem.yaml` and the last verdict. The master agent writes a short handoff note, which is shown at the top of stage 2's chat and attached to the brief as a labelled note. The master agent can also explain verdicts on request.

**Acceptance criteria:**
- [ ] The brief's essential contents are identical whether or not the note mentions them (test by removing the note).
- [ ] The note is visibly labelled as the master agent's.
- [ ] Demoable in isolation.

### 4.6 Stage 2 in chat: feature structure, `features.py` and checks

- **Type:** AFK
- **Blocked by:** 4.5
- **PRD reference:** §6.2 (stage 2), §6 check table (regression)
- **System design reference:** ADR-005; D29, D38

**Scope:**
- **Stage structure:** one file per feature plus a list.
- **`ds-features` skill:** a feature-idea catalogue, leakage pitfalls, the one-feature protocol.
- **Platform:** generates `features.py`.
- **Checks:** leakage (target association, forbidden columns, time leakage), redundancy (VIF), and CV lift on the spec's metric. Check code reads its metric and task type from `problem.yaml`.
- **Reuse:** generalise `feature_check.py` from the existing skill: remove `eval()`, fix its docstring example, make the model configurable.

**Acceptance criteria:**
- [ ] The planted neighbourhood-mean leak fails the leakage check.
- [ ] No `eval()` anywhere in the checks; no dataset names, columns or metrics hardcoded (grep test).
- [ ] Demoable in isolation.

### 4.7 Stale cascade

- **Type:** AFK
- **Blocked by:** 4.6
- **PRD reference:** §6.2 (going back), §11.3
- **System design reference:** §5.6; D30

**Scope:** Reopening and re-approving stage 1 marks every later stage stale. The platform automatically re-runs each stale stage's current approved code and checks on the new inputs, as a background job, and then waits for re-approval.

**Acceptance criteria:**
- [ ] Stage 2 shows *stale*, then *awaiting review*, with fresh verdicts.
- [ ] Nothing advances without a new approval.
- [ ] Demoable in isolation.

### 4.8 Chat retry nudge

- **Type:** AFK
- **Blocked by:** 4.3
- **PRD reference:** §9 (chat-mode retry nudge)
- **System design reference:** §5.2

**Scope:** After 5 consecutive revisions with failing checks in chat mode, the UI suggests the user step in or switch to loop mode with a clearer goal.

**Acceptance criteria:**
- [ ] The nudge appears after exactly 5 failed attempts and resets on a pass.
- [ ] Demoable in isolation.
