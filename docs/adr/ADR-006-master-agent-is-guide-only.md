# ADR-006: The master agent guides but holds no authority over approvals, transitions or checks

**Status:** Accepted
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

The user added a master agent: they chat with it first, it takes the problem and the dataset location, and it moves them through the stage agents (now five) (grilled Q26–Q29). An obvious design would make it an orchestrator that runs the workflow. But Phase 1's claim depends on three things (decisions 3, 4, 5):
- every stage advances only on human approval;
- checks are platform-owned and can't be overruled;
- the held-out set is hidden from every agent.

An orchestrating AI would be signing off on its own team's work.

---

## Decision

We will make the master agent **guide-only**:
- it runs intake and drafts `problem.yaml`;
- it explains check verdicts in plain language;
- it writes a short handoff note and suggests when a stage looks ready.

It cannot approve, advance, skip or re-run a check, or change a verdict. Approvals and stage transitions belong to the user, through the platform.

Supporting rules:
- **Intake:** the platform imports, fingerprints and profiles the dataset. The master agent sees only the profile and a row sample. The held-out split happens after the user confirms the spec (decision 44).
- **Handoffs:** the platform builds each stage's brief deterministically and attaches the master agent's note as a labelled note (decision 45).
- **Interface and UI:** the master agent uses the same file-based interface as the stage agents (ADR-001), on a new Project step in the stepper (decision 43).

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Guide only (chosen)** | Keeps human approval and the separate verifier intact; the master agent adds explanation and continuity without becoming a judge | Less automation: the user still clicks Approve at every stage |
| Orchestrator that advances a stage when all checks pass | Faster end-to-end runs | Breaks decision 3; an AI decides that AI work is good enough; weakens the demo's central claim |
| One master conversation, with stage agents as hidden sub-agents | A single chat thread | The user can't review or steer each stage directly; stage agents stop being separate plug-in points for Phase 2 coworkers |
| Master agent reads the dataset directly from the user's path | Simpler intake | Breaks the sandbox (any path on the laptop); the agent would see rows that later become held-out data |

---

## Consequences

**Positive:**
- The demo opens with a strong moment: give a problem and a path, and get a pre-filled spec to confirm. The trust model is unchanged.
- Essential brief contents never depend on what the master agent remembers to include.
- Phase 2 swaps only the stage agents, so comparisons stay focused on checked output (PRD Phase 2, open question 6).

**Negative / Accepted Tradeoffs:**
- The master agent can still draft a wrong spec, such as a random split on time-ordered data or a leaky column not forbidden. This is mitigated by the confirmation form with agent-filled values highlighted, and by the platform's profile flagging likely time columns.
- One more agent session per project adds a little cost. It uses the default model, Haiku 5.5 (decision 26).
