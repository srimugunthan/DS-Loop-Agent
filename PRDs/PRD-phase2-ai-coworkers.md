# PRD Phase 2: DS-Loop-Agent with AI Coworkers

| | |
|---|---|
| **Status** | Draft v2: aligned with Phase 1 Draft v3 (master agent, file browser, five stages) and the Phase 1 system design (`decision-log.md`) |
| **Date** | 2026-11-27 (v1: 2026-11-26) |
| **Phase** | 2 of 2. Builds on `../PRD-phase1-skills-agent.md` |
| **Prerequisite** | Phase 1 exit criteria met (Phase 1 PRD, section 13) |
| **Inputs** | Phase 1 PRD, `AGENT_DS_WORKFLOW_BRAINSTORMING.md` (revision 2, section 3), `ai-coworkers.jpeg`, web research on coworkers (sources at the end) |

---

## 1. Summary

Phase 1 delivers DS-Loop-Agent with a **skills-driven agent**: our harness, our skills, our tools, in chat mode and loop mode, behind a documented agent interface. A **master agent** handles project intake and guides the user between stages, and five **stage agents** do the work.

Phase 2 lets the user **swap the agent behind any of the five stages for an AI coworker**, a product that already has its own harness, tools and planner (OpenHands, Goose, Grok Bot, Hermes Agent). The user then **compares** it with the skills-driven agent on the same task, under the same checks, the same loop programs and the same review step. The master agent stays the skills-driven agent (proposed; see open question 6): it guides and explains but produces no output that is checked, so there is nothing to compare.

Phase 2 proves:
1. **The checks are agent-agnostic.** The same gates catch the same problems whoever the agent is.
2. **Swapping agents costs only an adapter**, with no changes to the UI, checks or loop runner.
3. **A fair, repeatable comparison** of AI coworkers on real DS work, reported as rates over repeated runs rather than single anecdotes.

> *Claim for Phase 2:* whoever the agent is — our skills-driven agent or an AI coworker with its own harness — its output is only trustworthy when each stage has a narrow scope, a human review step, and objective checks the agent cannot overrule.

---

## 2. Problem

- AI coworker products are multiplying, each with its own harness, tools and claims. There is no easy way to compare them on the *same* data science task, with the *same* goal and the *same* standard of review.
- Coworkers differ in **who enforces the boundaries**. Some run in a sandbox we control; some share one computer across all their bots. A comparison is only meaningful if the checks sit outside every agent.
- A one-off demo ("look, a coworker built a model") says nothing about reliability. Agents are nondeterministic, so claims need repeated runs.

---

## 3. Users

| User | Goal |
|---|---|
| **Data scientist** | Pick the agent that works best for each stage; try a coworker without changing how they work |
| **Evaluator of AI coworkers** | Run the same stage and loop program with different agents and compare results, rejections, effort and cost |
| **Demo presenter (you)** | Show that the gates hold for any agent, and show a fair coworker comparison |

---

## 4. Goals and non-goals

### Goals
1. **Coworker adapters** implementing the Phase 1 agent interface for at least one coworker in full (OpenHands first), plus more where feasible.
2. **Agent selector** on the stage agent card: choose the skills-driven agent or a coworker per stage.
3. Coworkers usable in **chat mode**, and in **loop mode** where their capability level allows (section 7).
4. A **comparison view** and an **experiment protocol** for fair, repeated comparisons (sections 8 and 9).
5. **Safety for third-party agents:** sandboxing, scoped credentials, checks run from a trusted copy, and every coworker's model calls routed through the Phase 1 **LLM gateway** (section 10).
6. A Phase 2 showcase (section 12).

### Non-goals (Phase 2)
- Changes to the workflow, stages, checks or problem spec from Phase 1, except fixes the comparison exposes.
- Coworkers with no programmatic access being driven in loop mode (they are compared in chat mode only).
- Mixing agents inside one loop (one backend per loop run).
- Replacing the master agent with a coworker (proposed; open question 6).
- Ranking coworker products in general. Results are for these tasks and datasets only, and are dated.
- Deployment, monitoring and multi-user features (still future work).

---

## 5. What changes in the UI

Everything from Phase 1 stays. Additions:

| Area | Element | Behaviour |
|---|---|---|
| Stage agent card | **Agent selector** (▾) | Lists the skills-driven agent and installed coworkers, with a capability badge (Full / Partial / Manual). Changing it for a stage starts a new session; earlier runs stay in history. Not shown on the Project step (master agent) |
| Master agent | Handoff notes name the backend | e.g. "Stage 2 was done by OpenHands; it passed 5/6 checks", so the user knows who produced what |
| File browser | Backend shown per version folder and experiment | Mixed histories stay readable in the tree as well as the chat |
| Agent card | Backend badge | Shows which agent produced the current output (e.g. "OpenHands · model X") |
| Chat and timeline | Backend tag on each message or iteration | So mixed histories stay readable |
| Right pane | **Compare** tab | Section 8 |
| Right pane, manual mode | **Upload result** panel | Section 7.3 |
| Loop mode | **"Re-run this program with…"** | Runs the exact same loop program with another backend |

```
┌───────────────────────────────────────────┬──────────────────────────────────────────────┐
│ (icon) features-gen-agent    ● Looping    │ COMPARE — stage 2 loop program "feat-01"      │
│   Mode: [ Chat | Loop ]                   │ ┌ Chart ──────────────────────────────────┐ │
│   Agent: [ OpenHands (Full) ▾ ]           │ │ CV RMSE   ── skills agent  ── OpenHands │ │
│     • Skills-driven agent (Full)          │ │  ●──●──●──●            (5 runs each,    │ │
│     • OpenHands (Full)                    │ │  ●──●─────●──●          median + range) │ │
│     • Goose (Full)                        │ └─────────────────────────────────────────┘ │
│     • Grok Bot (Manual)                   │ ┌ Table ──────────────────────────────────┐ │
│                                           │ │            Skills agent   OpenHands     │ │
│                                           │ │ Goal met       4/5           3/5        │ │
│                                           │ │ Leak proposed  1/5           2/5        │ │
│                                           │ │ Leak kept      0/5           0/5        │ │
│                                           │ │ Iterations     9 (7–12)     12 (9–15)   │ │
│                                           │ └─────────────────────────────────────────┘ │
└───────────────────────────────────────────┴──────────────────────────────────────────────┘
```
(The numbers above are illustrative placeholders, not results.)

---

## 6. Coworker shortlist

Based on `ai-coworkers.jpeg` and the research in `AGENT_DS_WORKFLOW_BRAINSTORMING.md` section 3. These products are new and change quickly; **verify every capability before building**.

| Coworker | What it is | Expected level | Role in Phase 2 | Order |
|---|---|---|---|---|
| **OpenHands** | Open-source (MIT) autonomous software engineer; SDK with local, Docker and remote workspaces; any model | Full | **First coworker.** Sandboxed and scriptable, so it can run loop mode | 1 |
| **Goose** | Open-source (Apache 2.0) local MCP agent; headless "recipes"; any model | Full (to verify) | Second coworker. A recipe maps well onto a stage brief or loop program | 2 |
| **Grok Bot** (xAI) | Hosted team of named bots on **one shared cloud computer**; Grok models only | Manual unless an API exists | **Headline comparison if access allows.** Its shared computer illustrates why checks must sit outside the agent | 3 (after an access spike) |
| **Hermes Agent** | Open-source (MIT); writes and refines its own `SKILL.md` skills from finished tasks; any model | Partial or Full (to verify) | Optional experiment: put its self-written skills through the same loop programs and checks (the "self-improvement" loop level from your deck) | 4 (optional) |
| **OpenClaw**, **Claude Cowork** | Chat gateway; desktop assistant | n/a (not stage agents) | Out of scope for Phase 2 as stage agents. Possible later: "loop finished" notifications and approvals in chat (OpenClaw); report generation (Cowork) | — |

Devin (Cognition) is a closed alternative to OpenHands. Add it only if a closed-product comparison is wanted.

---

## 7. Integrating coworkers

### 7.1 Adapter

Each coworker gets an adapter implementing the Phase 1 interface. Phase 1 already uses a **coworker-shaped boundary**: the agent gets a workspace folder, a brief and the loop program, and returns results as files (`proposal.json` in loop mode), with no tools that call the backend. A coworker therefore plugs in the same way: it runs in the same per-project agent container, sees the same workspace, and the platform applies, checks and logs exactly as in Phase 1.

| Operation | What the adapter must do for a coworker |
|---|---|
| `start(stage, project_context)` | Translate the platform-built stage brief (approved artefacts, problem spec, last verdict, the master agent's handoff note) into the coworker's own format (e.g. a task description, an OpenHands workspace, a Goose recipe) |
| `send(message)` | Forward chat messages and check results |
| `propose(loop_state)` | Ask for exactly **one** change per call, written as `proposal.json` in the Phase 1 format. If the coworker returns more than one change or a malformed proposal, the experiment is logged with status `invalid` |
| `events()` | Map the coworker's own events (where available) to ours: text, status, tool calls, file written, errors, done |
| `artifacts()` | Collect the files it changed, **restricted to the stage workspace**. Anything outside is rejected |
| `stop()` | Stop the coworker's session or container |
| `capabilities()` | Report its level (7.2) |

### 7.2 Capability levels

| Level | What works | Loop mode? | Expected fit (to verify) |
|---|---|---|---|
| **Full** | Streamed chat and events; files appear live; runs in our sandbox | Yes | OpenHands, Goose |
| **Partial** | Final answer and files only, no live events | Yes, if it can be called once per iteration from a script | Command-line agents without streaming; possibly Hermes |
| **Manual** | The user works with the coworker in its own app; the result comes back by upload or git branch | **No.** Chat-style comparison only | Grok Bot, unless the access spike shows an API |

In every level **checks, review and approval are identical to Phase 1**, and every coworker experiment goes into the same experiment record (folders, `experiments.jsonl`, the MLflow mirror) tagged with its backend and version.

### 7.3 Manual mode flow

For coworkers with no programmatic access:
1. DS-Loop-Agent generates a **brief** (stage, scope, problem spec, goal) for the user to paste into the coworker.
2. The coworker works in its own environment, using only the public dataset.
3. The user brings the result back via the **Upload result** panel (files, or a git branch name).
4. The platform enforces scope (only stage-folder files accepted), runs the checks, and shows results exactly as in Phase 1. Uploaded files appear in the file browser like any other version.
5. Feedback ("Send check results") produces text for the user to paste back.

This keeps the comparison honest but cannot be replayed automatically. It is used for chat mode only, on one stage.

---

## 8. Comparison view

A **Compare** tab in the right pane, available for any stage where more than one backend has run.

| Element | Contents |
|---|---|
| Chart | For loop runs of the same program: metric over iterations per backend (median and range across runs) |
| Table | Per backend: goal met (x of N), iterations to stop, proposals discarded, **leaks or defects proposed**, **leaks or defects kept** (should be 0 for everyone), held-out result of the champion, user turns (chat mode), wall-clock, cost where available |
| Defect scorecard | Against the answer key: found, fixed, missed, wrongly "fixed", per backend |
| Drill-down | Click any cell to open the runs and their timelines |

Only metrics **observable for every backend** go in the headline table. Cost may not be visible for hosted coworkers; that is shown as "n/a", not estimated.

---

## 9. Experiment protocol

| Rule | Detail |
|---|---|
| **Same inputs** | Same dataset version, problem spec, approved upstream stages and loop program for every backend |
| **Repeated runs** | About 5 runs per backend per loop program; results reported as rates and ranges, not single runs |
| **Same model where possible** | For "any model" coworkers (OpenHands, Goose, Hermes), run them **with the same underlying LLM as the skills-driven agent** (Phase 1 default: Claude Haiku 5.5; whatever the stage is configured to use at the time). This isolates the harness's effect from the model's. Optionally add a second set with each coworker's recommended default model |
| **Same budgets** | Same iteration, time, per-experiment and **dollar** caps, enforced by the loop runner. Dollar caps can be enforced for any coworker whose model calls go through the LLM gateway (section 10) |
| **No special prompting** | Coworkers get the generated brief and program, nothing tuned per product beyond what the adapter needs |
| **Fixed seeds and folds** | Same CV folds and tuner seeds across backends |
| **Dated results** | Every comparison records product versions and the date |
| **Spontaneous-tamper count** | Across all runs, count attempts to touch anything outside scope (checks, held-out data, other stages). Never stage this with prompts that push an agent to cheat |

---

## 10. Safety for third-party agents

Coworkers are code we don't control, so Phase 1's protections are tightened:

| Risk | Requirement |
|---|---|
| A coworker edits checks or thresholds | Checks always run from the platform's trusted copy, never from anything the agent touched. Any change outside the stage folder is rejected before checks run |
| A coworker reads the held-out set | The held-out set is never in any workspace a coworker can reach |
| A coworker runs unsafe code | Full and Partial backends run inside the Phase 1 per-project agent container. Network access only where the coworker itself needs it to reach its model API, and nothing else |
| A coworker reads or leaks the model API key | Reuse the Phase 1 **LLM gateway** (decision 36, ADR-007). Coworker containers get no key and no internet; their SDK or model client is pointed at the gateway, which forwards only model requests, meters tokens per experiment and enforces the dollar cap. Phase 2 adds keys for any other model providers a coworker needs. A coworker whose model client can't be pointed at the gateway is handled case by case (manual mode at most) |
| Hosted coworkers (e.g. Grok Bot's shared computer) | Public data only. A **throwaway repo and a token scoped to that repo** if git is used. Never sign the shared computer into anything else. xAI's guidance reportedly says not to treat separate bots as a security boundary, so don't |
| Coworker-supplied strings executed by checks | No `eval()` anywhere in checks (already a Phase 1 fix) |
| Adapter credentials | Stored by the platform (or the gateway), never written into a coworker's workspace |

**Tamper test** (for the showcase, from the plan): a **scripted** change that edits a check is rejected deterministically. That is the reliable demonstration. Separately, report the spontaneous-tamper count from the protocol (section 9).

---

## 11. Milestones

| Milestone | Scope | Exit criteria |
|---|---|---|
| **P2-M0: Access spikes** (about 1 day) | Install OpenHands and Goose; check each against the interface, including whether its model client can be pointed at the LLM gateway. **Grok Bot spike (2 hours):** access, API or not, can it push to a throwaway repo with a scoped token, can its actions be exported | A capability level assigned to each coworker, with evidence |
| **P2-M1: First coworker** | Point OpenHands' model client at the LLM gateway; OpenHands adapter; agent selector; backend tags; stage 2 in chat and loop modes | The Phase 1 stage 2 loop program runs unchanged with OpenHands; the checks behave identically; no UI changes beyond the selector and tags |
| **P2-M2: Comparison** | Compare tab; experiment protocol tooling (batch runs, same-model setting); defect scorecard per backend | About 5 runs each for the skills-driven agent and OpenHands on the stage 2 loop, for both problems, reported as rates |
| **P2-M3: More coworkers** | Goose adapter; Grok Bot in manual mode (or API if the spike allows); optionally Hermes; stage 3 loop comparison for Full-level coworkers | Each added coworker needs only a new adapter. Results added to the comparison |
| **P2-M4: Phase 2 showcase** | Recording, comparison report, README update | Showcase flow (section 12) recorded; comparison report dated with versions |

---

## 12. Phase 2 showcase flow (about 6 minutes, after the Phase 1 demo)

| Min | Beat | What the audience sees |
|---|---|---|
| 0–1 | Swap the agent | On stage 2, the agent selector changes from the skills-driven agent to OpenHands. Nothing else in the UI changes |
| 1–3 | Same loop, different agent | "Re-run this program with OpenHands": the same goal, budget and checks. The leaky feature is discarded *whoever proposed it* |
| 3–4 | The comparison | Compare tab: two curves, the results table over 5 runs each, the defect scorecard |
| 4–5 | A hosted coworker | Grok Bot in manual mode (or API): brief out, result uploaded, the same checks run |
| 5–5.5 | Tamper guard | A scripted change to a check is rejected; spontaneous attempts: "x of N runs" |
| 5.5–6 | Close | The Phase 2 claim |

---

## 13. Success metrics

| Metric | Target |
|---|---|
| Swap cost | Each new coworker needs only an adapter: zero changes to UI, checks, loop runner or skills |
| Gate consistency | 0 planted leaks or defects kept, for every backend |
| Comparison quality | About 5 runs per backend per program, same model where possible, reported with ranges and dated versions |
| Coverage | At least one coworker at Full level on stages 2 and 3; at least one Manual coworker compared in chat mode |
| Tamper resistance | Scripted tamper attempts rejected 100% of the time |

---

## 14. Risks

| Risk | Mitigation |
|---|---|
| Coworkers expose little or no API | Capability levels; manual mode keeps them comparable in chat mode |
| A coworker won't make "one change per iteration" | The adapter enforces it; multi-change proposals are rejected and counted. The rate is itself a comparison result |
| Model differences hide harness differences | Same-model runs for "any model" coworkers (section 9) |
| Comparison seen as a product ranking | Scope and dates stated with every result; only these tasks and datasets |
| Hosted coworker leaks credentials or data | Public data only, throwaway repo, scoped token, no other sign-ins |
| Coworker cost is unpredictable or invisible | Budgets enforced by the loop runner; cost shown as "n/a" where not reported |
| Product changes break adapters | Pin versions per experiment; record them in the log |
| Nondeterminism makes the demo vary | Golden-run recording; repeated runs for claims |

---

## 15. Open questions

1. **Grok Bot:** do you have access? This decides whether P2-M0's spike is worth running.
2. **Second coworker:** Goose next, or another from your list?
3. **Same-model runs:** should the headline comparison use the same LLM for every "any model" coworker (fairer), each one's default model (more realistic), or both?
4. **Hermes experiment:** in or out of Phase 2?
5. **Chat-based notifications** (OpenClaw "loop finished, approve?"): Phase 2 or later?
6. **Master agent:** keep it as the skills-driven agent in Phase 2 (proposed), or let a coworker take that role too? The proposal keeps comparisons focused on the stage agents, whose output is what the checks judge.
7. **Same model vs. coworker strength:** Phase 1 defaults to Claude Haiku 5.5. Some coworkers may work poorly on a small model, so the same-model comparison may need a larger model (e.g. Sonnet 5.5) for every backend.

---

## Sources (coworker research; most are third-party; verify before presenting)

- Grok Bot: [xAI docs](https://docs.x.ai/grok-bot/overview), [announcement](https://x.ai/news/introducing-grok-bot), [security commentary](https://michaelbennett.co/blog/grok-bot-shared-computer)
- OpenHands: [SDK Docker sandbox](https://docs.openhands.dev/sdk/guides/agent-server/docker-sandbox)
- Goose: [recipe reference](https://block.github.io/goose/docs/guides/recipes/recipe-reference)
- Hermes Agent: [docs](https://hermes-agent.nousresearch.com/docs)
- OpenClaw: [docs index](https://docs.openclaw.ai/llms.txt)
- Claude Cowork: [getting started](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork)
