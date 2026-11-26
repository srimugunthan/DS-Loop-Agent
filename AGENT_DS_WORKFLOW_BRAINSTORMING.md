# Agent-Driven Data Science Workflow: Build Plan, Showcase Plan and AI Coworker Options

*Revision 2, 2026-11-25. Based on everything in this folder plus web research on AI coworkers. What changed from revision 1 is listed in Appendix B.*

---

## 0. TL;DR

- **The thesis:** agents (including AI coworkers) are useful in a DS workflow only when each stage has a narrow scope and a **deterministic gate the agent cannot overrule or tamper with**. Every design choice below serves that claim.
- **The architecture:** a chain of six stage loops (your decks' propose → verify → keep/retry pattern). Each proposal is a **git branch that touches only that stage's files**. A **gate runner** runs trusted verifiers from the protected main branch and writes `verdict.json`. A small orchestrator routes on verdicts and enforces retry caps.
- **The agents, by role rather than by product:**
  - **Reference proposer:** a plain coding agent with your `loop-engineer` skill (Claude Code, run headless). It is scriptable and replayable, and it reuses your skill directly. It runs every stage.
  - **Guest proposer (the AI coworker seat):** **Grok Bot** runs stage 2 through the same gates, *if* a 2-hour access spike shows it can open PRs against a repo. **Goose or OpenHands** is the open-source control, and the fallback if Grok Bot fails the spike.
  - **Human-in-the-loop channel:** **OpenClaw** sends the retrain approval request to your phone.
  - **Reporter (optional):** **Claude Cowork** turns the run log into a report.
- **Does a plain agent with skills work?** Yes. It is the best primary driver here (section 3.3). The coworkers add story value and prove the gates are agent-agnostic. They are not needed for the pipeline to work.
- **Dataset:** Ames Housing, split by **sale date** (2006 to 2010). The time split makes the planted leaks and the drift demo real instead of contrived.
- **Effort:** about 1 week for a credible stages 1 to 4 demo with one proposer. About 3 weeks part-time for everything, including the coworker seats.

---

## 1. What is currently in this folder

| Item | What it is | Relevance |
|---|---|---|
| `DS-workflow.pptx` (26 slides) | First deck: "Loop Engineering: Iterative Feedback Loops in Data Science Workflows". The worked example is **house price prediction (regression)**. Six loops, a nested-loop diagram, a design checklist. | Your **spec**: each stage's use case, skills, exit condition and flow. |
| `loop_engineering_data_science (2).pptx` (30 slides) | Revised deck. Adds the **Karpathy Loop pattern** (one editable scope, one objective metric, a verifier separate from the proposer, bounded retries, a stopping condition set up front). It also has notes from a LangChain talk on four loop levels (core, verification, event-driven, self-improvement), which stresses human approval for significant actions. | Your **design principles**. The most up-to-date framing. |
| `when_to_loopengr.pptx` (2 slides) | Almost empty (one slide says "x"), about 5 MB, probably images. | A stub. A good home for the "when *not* to use loop engineering" slide (section 5.6). |
| `loop-engineer-skill.skill` | Packaged skill: `SKILL.md`, `assets/program_template.md`, six `references/<stage>.md`, `scripts/feature_check.py`. | Your **first working asset**. Only feature engineering is fully built. |
| `Loop_engr_notes.rtf` | Notes on an Atomic masterclass (Alex Lavaee, Microsoft): agent runtimes, workflow graphs (DAGs), verbatim compaction, verifiability, sandboxing, adversarial audits. | Informs the orchestrator and the "adversarial audit" (tamper) demo. |
| `graphengg/` (12 `.jp2` images) | Pages of an independently compiled paper, "Graph Engineering for Multi-Agentic Systems: The Andrew Ng Playbook" (it says it is not endorsed by the people it names). It argues loop → chain → network → graph, use the simplest pattern that works, and put programmatic gates between stages. | Sets **how much architecture** to build (section 4). |
| `ai-coworkers.jpeg` | Your comparison of OpenClaw, Hermes Agent, Claude Cowork, Goose, OpenHands and Grok Bot. | Input to section 3. |

### 1.1 Gap analysis of the skill

- **Built:** `feature_check.py` (a leakage heuristic via correlation, a VIF check, CV-RMSE lift using a linear model).
- **Described but not built:** verifiers for data prep, training, evaluation, deployment and monitoring.
- **Missing:** dataset, stage agents, hand-off contracts, orchestrator, gate runner, run report.
- **Fix before showcasing `feature_check.py`:**
  1. **`eval()` on the candidate expression.** In revision 1 I called this fine locally. It is no longer fine once a hosted coworker supplies the string and CI runs it. Replace it with a whitelist of operations (ratio, difference, log, product, bucketing over named columns), or require features as code in `candidate_features.py` that runs in a sandboxed step without secrets.
  2. **The docstring example is broken.** `df['square_footage'] and df['listing_price']` raises an error on pandas Series. Fix it so nobody copies it into a demo.
  3. **The leakage check is only a correlation threshold (0.97).** It catches blatant leaks. It misses noisy ones (a correlation of 0.9 is still a leak) and time leaks. That is acceptable *if* the evaluation gate catches what this misses (section 2.4). Say so on a slide.
  4. **Linear-model lift versus tree models in stage 3.** Make the verifier's model configurable, or note the mismatch.
  5. **Shuffled KFold.** For a time-split project, use time-ordered folds (`TimeSeriesSplit` or folds grouped by year) at least in the evaluation gate. I would keep shuffled KFold in stage 2 on purpose, so the target-encoding leak gets through there and is caught later (section 2.4).

---

## 2. The architecture to build

### 2.1 Core idea

Each stage is a **bounded propose → verify → keep/retry loop**. A **fresh proposer with narrow scope** makes one change. A **deterministic gate** (not an LLM) decides keep or discard. Stages communicate through **artifacts with a written contract**, never through a shared chat transcript.

```
                 ┌──────────── Monitoring / retraining (outer loop) ──── human approval (OpenClaw) ─┐
                 │                                                                                   │
raw data ─► [Data Prep] ─► GATE 1 ─► [Features] ─► GATE 2 ─► [Modelling] ─► GATE 3 ─► [Evaluation gate]
              ▲  │ retry               ▲  │ retry              ▲  │ retry               │
              └──┘                     └──┘                    └──┘        reject → back to Features/Modelling
                                                                           approve → Deploy (simulated canary)
```

### 2.2 One hand-off mechanism for every proposer

Revision 1 had a local `workspace/` for the plain agent and GitHub PRs for coworkers. Use **one** mechanism from Phase 1 onward, so that adding a coworker later is an adapter, not a rewrite:

1. The orchestrator creates a branch `stage-02/iter-007` for each proposal.
2. The proposer (Claude Code, Goose, OpenHands, Grok Bot or a human) commits changes **only to that stage's scoped files**.
3. The **gate runner** (`gates/run_gate.py`, used both locally and in CI):
   - fails immediately if the diff touches anything outside the stage's scope (path guard);
   - loads the verifier code and thresholds **from `main`, never from the branch**;
   - runs the verifier in a step with no secrets and no network;
   - writes `verdict.json` and appends one line to `loop_log.jsonl`.
4. Keep → merge. Discard → the reasons go back to the proposer, up to the retry cap in `program.md`, enforced by the orchestrator.

Locally the "PR" can just be a branch, so you do not need GitHub until the coworker phase.

### 2.3 Repo layout

```
agentic-ds-workflow/
├── README.md                     # showcase entry point
├── data/
│   ├── raw/housing_raw.csv       # Ames, 2006–2008 sales, with injected defects
│   ├── drift/housing_2009_10.csv # later sales, for the monitoring loop
│   └── README.md                 # answer key: every injected defect and planted leak
├── programs/                     # one program.md per stage (from the skill template)
├── agents/                       # Claude Code subagent definitions, one per stage
├── adapters/                     # propose(brief, scope) -> branch, one per proposer
│   ├── claude_code.py
│   ├── goose.py                  # or openhands.py
│   └── manual.md                 # human-driven flow for Grok Bot if it has no API
├── verifiers/                    # deterministic, no LLM calls; protected (CODEOWNERS)
│   ├── data_quality_check.py
│   ├── feature_check.py          # from the skill, with the fixes in 1.1
│   ├── training_check.py
│   └── evaluation_gate.py
├── gates/run_gate.py             # path guard + runs verifiers from main
├── contracts/                    # JSON Schemas for each stage's hand-off artifact
├── stages/                       # the ONLY files proposers may change
│   ├── 01_cleaning_rules.py
│   ├── 02_candidate_features.py
│   └── 03_model_config.yaml
├── orchestrator/run.py           # stage order, routing, retry caps, resume
├── reports/                      # run report generated from loop_log.jsonl
└── .claude/                      # skill, subagents, permission rules
```

The **held-out test set is not in this repo.** The gate runner fetches it from a location only the gate can read (a CI secret store, or a local folder outside the agent's working directory). A hash check catches edits but not peeking, so keeping it out of view is the only real protection.

### 2.4 Dataset: Ames Housing, split by sale date

Why Ames rather than California Housing: Ames has sale dates (`YrSold`, `MoSold`, 2006 to 2010), so a time-based split is possible. That makes three demo beats real rather than staged:

| Planted item | Where it should be caught | Why it is credible |
|---|---|---|
| **Defects** for stage 1: a $50 price typo, sqft versus sqm mixing, nulls, duplicates (seeded, documented). | Gate 1 | Matches your deck's own examples. |
| **Blatant leak:** a column derived from `SalePrice` with little noise. | Gate 2 (correlation over 0.97) | Shows the cheap check working. |
| **Subtle leak:** neighbourhood mean sale price computed over *all* years (target encoding with future data). | **Not** gate 2 (shuffled CV looks fine). **Caught by the evaluation gate** on the time-ordered held-out set. | Shows defence in depth: no single gate is enough. This replaces revision 1's "looks good, isn't" feature, which shuffled CV could not have produced. |
| **Segment trap:** high-price homes have different error behaviour. | Evaluation gate (error per price band) | Your deck's "luxury homes error differently" point. |
| **Drift:** 2009 to 2010 sales (after the 2008 crisis). | Monitoring loop | Real drift, not synthetic. Check that the shift is large enough to cross a threshold. If not, add a documented synthetic shift. |

Note on `distance_to_city_center`: the original Ames file has no coordinates. I believe the `AmesHousing` R package version (`make_ames()`) adds latitude and longitude. Check this. Otherwise use an Ames-native legitimate feature such as house age at sale (`YrSold − YearBuilt`).

### 2.5 Stage-by-stage contract

| Stage | Proposer may change | Gate checks | Exit condition (from your deck) | Hand-off artifact |
|---|---|---|---|---|
| 1. Data prep | `01_cleaning_rules.py` | Null rate, schema, ranges, duplicates, **row count (no silent drops)** | Thresholds met, every row drop logged with a reason | `clean.parquet` + `data_report.json` |
| 2. Features | `02_candidate_features.py` | Leakage (correlation), VIF, CV-RMSE lift | Marginal lift below threshold, or feature budget hit | `features.parquet` + `feature_manifest.json` |
| 3. Training | `03_model_config.yaml` | CV score and **fold variance stable** | Variance within tolerance, not "best single split" | `model.pkl` + `cv_results.json` |
| 4. Evaluation | **nothing** (a gate, not a proposer) | Time-ordered held-out set: beats baseline, no price band disproportionately bad | Approve, or route back to stage 2 or 3 | `evaluation_verdict.json` |
| 5. Deployment | rollout stage | Replayed live rows versus validation metrics | Within tolerance | simulated |
| 6. Monitoring | trigger only | PSI/KS drift and error creep on the 2009 to 2010 data | Never exits; a retrain needs **human approval** | alert + approval record |

### 2.6 Real versus simulated

| Stage | Real | Simulated |
|---|---|---|
| 1 to 4 | All of it, locally, seconds to minutes | n/a |
| Deployment | A FastAPI model served locally | Canary traffic replayed from a file |
| Monitoring | Drift computed on real later sales | Passage of time (you replay month by month) |

Say on screen which parts are simulated.

---

## 3. Agent and AI coworker options

### 3.1 Think in seats, not products

The project has four seats. Each product is good at a different one:

| Seat | Needs | Best fit |
|---|---|---|
| **Reference proposer** (all stages) | Scriptable, scope enforceable, readable log, reuses the skill | Claude Code (headless) with the `loop-engineer` skill |
| **Guest proposer** (one stage, for comparison) | Can make a branch or PR, observable | Grok Bot (headline, if the spike passes); Goose or OpenHands (open control) |
| **Human-in-the-loop channel** (stage 6) | Reaches a human in chat, returns a decision | OpenClaw (or Hermes Agent) |
| **Reporter** (optional) | Reads local files, produces polished docs on a schedule | Claude Cowork |

### 3.2 Comparison for this project

Ratings are my judgement for *this project*, not general product reviews. All products are new and changing, so verify before publishing.

| Tool | What it is | Driven by script? | Can scope be enforced? | Models | Best seat here |
|---|---|---|---|---|---|
| **Claude Code** | CLI coding agent with skills, subagents, permission rules and hooks | Yes (headless mode) | Yes: permission rules plus the gate runner | Claude | Reference proposer |
| **Goose** (Block, now in the Linux Foundation's Agentic AI Foundation) | Local MCP agent; **recipes** pin settings and run headless (`goose run --recipe`) | Yes | Partly (tool and extension limits); the gate does the rest | Any | Guest proposer / open control |
| **OpenHands** | Autonomous software engineer; SDK with a Docker workspace | Yes (SDK) | Yes (sandboxed workspace) | Any | Guest proposer / open control |
| **Grok Bot** (xAI) | Hosted team of named specialist bots on **one shared cloud computer** | **Unconfirmed** (biggest open question) | **No.** xAI's guidance reportedly says not to treat separate Bots as a security boundary | Grok only | Guest proposer, if the spike passes |
| **OpenClaw** | Self-hosted gateway from chat apps to agents; large skill registry (ClawHub) | Yes | n/a (not proposing code) | Any | Human-in-the-loop channel |
| **Hermes Agent** (Nous Research) | Agent that writes and refines its own `SKILL.md` skills from finished tasks | Yes | Partly | Any | Alternative channel; optional "self-improvement" extension (section 5.4) |
| **Claude Cowork** | Desktop assistant: local files, scheduled tasks, plugins | Scheduled tasks | n/a | Claude | Reporter |

Devin (Cognition) is the best-known closed hosted coworker. For this project, OpenHands covers the same role, is open source and can be scripted, so I would not add Devin unless you specifically want a closed-product comparison.

### 3.3 Does a plain agent with some skills work?

**Yes, and it should be the primary driver.**

1. **Your thesis is already a skill.** `loop-engineer` encodes the pattern, so a plain agent that loads it behaves as your slides describe.
2. **Narrow scope maps onto subagents plus permission rules.** Each stage's subagent gets a fresh context and may write only its `stages/` file.
3. **Trust lives outside the model.** The gate runner, not the agent, decides keep or discard.
4. **It is the "chain with programmatic gates" level** that the paper in `graphengg/` recommends as the default.

What a plain agent does not give you, and the fix:

| Gap | Fix |
|---|---|
| The agent could edit verifiers or read the held-out set | Path guard, verifiers run from `main`, held-out set outside the repo (section 2.2) |
| Stages leak context into each other | Fresh subagent per stage; hand-off only through contract-checked artifacts |
| No durable state across crashes | Branches, artifacts and `loop_log.jsonl` are the state; the orchestrator resumes from the last passed gate |
| Retries wander | Caps enforced by the orchestrator, not just written in the prompt |

### 3.4 Grok Bot: what to know before using it

- **Shared computer.** xAI's docs say Bots share one persistent cloud computer (browser, filesystem, terminal). Sessions signed in for one bot are reportedly available to the others. The boundary must therefore live outside the bot: it only ever sees a branch and a `verdict.json`.
- **This is a strength for the story.** A coworker that can touch everything is exactly why the gate must be external and run from `main`. Grok Bot illustrates the thesis well.
- **Unconfirmed: can it be scripted?** If it is chat or web only, it can still take part through `adapters/manual.md`: you paste the brief, it pushes a branch, the gate posts the verdict, you paste the reasons back. That works for a recorded demo but is not replayable, so keep it to one stage.
- **Credentials.** Use a throwaway repo and a fine-grained token scoped to that repo only. Never sign the shared computer into anything that matters.
- **Data.** Public Ames data only.

### 3.5 Recommended lineup

- **Minimum viable:** Claude Code (all stages) plus the gates. This alone proves the thesis.
- **With a coworker (what you asked for):** add **Grok Bot as the guest proposer on stage 2** plus **Goose or OpenHands as the open control** on the same stage, and **OpenClaw** for the retrain approval.
- **Optional:** Cowork for the report; Hermes Agent as a "self-improving proposer" experiment.
- **If Grok Bot fails the spike:** OpenHands takes the guest seat. Grok Bot moves to a "what I'd test next" slide, with the shared-computer point kept, because it still makes the argument.

Why stage 2 only for the guest: it has the most interesting gate (the leak), cheap iterations, and the clearest comparison. Running a guest across all stages multiplies cost and demo risk without adding much story.

---

## 4. How much architecture? (from the `graphengg` paper)

| Level | Use here? | Why |
|---|---|---|
| Single loop | Yes, inside each stage | The Karpathy-loop core |
| **Chain with programmatic gates** | **Yes, the main design** | Stages are fixed and ordered; gates stop bad output spreading |
| Network (orchestrator plus workers) | Only for the outer loop and the guest-proposer comparison | Several proposers on the same stage is a small network |
| Knowledge graph as shared state | **Not in v1**; a "v2" slide | Pays off when many workers need durable shared facts. Good "at scale" slide: lineage of which feature came from which cleaning rule |

The Atomic notes point the same way: a durable workflow DAG is the production form of `orchestrator/run.py`. For an example project a plain script is enough. Name Atomic or LangGraph as the upgrade path.

---

## 5. How to showcase the project

### 5.1 The one claim

> **"Whoever the agent is — a coding agent or an AI coworker — it is only trustworthy in a DS workflow when every stage has a narrow scope and a gate it cannot overrule or tamper with."**

### 5.2 Showcase assets

1. **README** with a 90-second story: hero diagram, a short screen recording, the comparison table.
2. **Run report** (HTML) generated from `loop_log.jsonl`: one card per iteration with proposer, proposal, rationale, verdict, metrics and reasons. Filter by stage and by proposer.
3. **A 10-minute demo**, recorded, with a live option (5.3).

### 5.3 Demo script (10 minutes)

| Min | Beat | What the audience sees |
|---|---|---|
| 0–1 | The problem | Raw Ames data: the $50 house, sqft/sqm mixing |
| 1–2 | **Stage 1** (Claude Code) | A proposal drops rows to pass the null check; gate 1 rejects it on row count; the retry imputes; pass |
| 2–4 | **Stage 2 with a coworker** | Split screen: Grok Bot (or OpenHands) and Claude Code both propose features through branches. The same gate posts verdicts. The blatant leak is rejected whoever proposed it |
| 4–5 | **Stage 3** | A model with the best single split is rejected for unstable fold variance |
| 5–6.5 | **Evaluation gate: defence in depth** | The subtle target-encoding leak passed gate 2, but fails on the time-ordered held-out set. The luxury price band also fails. Routed back to stage 2 |
| 6.5–8 | **Monitoring + human in the loop** | 2009 to 2010 sales trigger the drift alert; OpenClaw pings your phone; you approve; retrain starts |
| 8–9 | **Tamper guard** | A branch that edits a threshold in `verifiers/` is blocked by the path guard, and the gate still runs `main`'s verifier. Then: "Did any proposer attempt this on its own? X of N runs" |
| 9–10 | **Close** | Comparison table across proposers, then the one claim |

Deployment (canary replay) moves to the README and the report. It is the least interesting beat on stage.

### 5.4 Strongest demonstrations, done honestly

LLM proposers are nondeterministic, so **any claim about agent behaviour needs repeated runs.** Run each experiment about 5 times and report rates, not anecdotes.

1. **Self-grading A/B.** Stage 2, about 5 runs each: (a) the agent decides keep or discard itself, (b) the gate decides. Report how often (a) kept a leak. If (a) never kept one, say so. The argument then becomes "you could not have known without the gate", which still holds.
2. **Tamper test, two parts.** (i) A **scripted** tampering branch proves the guard works deterministically; that is the reliable part. (ii) Record whether any proposer *spontaneously* tried to edit gates or thresholds across all runs. Do not stage it with a prompt that pushes the agent to cheat and then present the result as natural behaviour.
3. **Proposer comparison table.** Only metrics observable for every proposer: proposals made, rejected, iterations to pass, wall-clock, defects from the answer key found or missed, cost where available (it may not be visible for Grok Bot).
4. **Defect scorecard** against the answer key.
5. **Optional, self-improvement level:** let Hermes Agent turn a finished run into a skill, then put *that skill* through the same gates on the next run. This shows the LangChain "level 4" loop with a gate around it.

### 5.5 Limits to state openly

- Gates catch only what you thought to check (shown on purpose: the subtle leak gets past gate 2).
- Thresholds were tuned on one dataset.
- Deployment and the passage of time were simulated.
- Few runs per proposer; products and versions are dated.

### 5.6 Terminology and "when not to loop"

Your revised deck (slide 5) notes that "loop engineering" usually means autonomous agent loops. This project is workflow iteration made agentic: say so up front. Use `when_to_loopengr.pptx` for one slide on when *not* to loop: a stage without a cheap, objective verifier (for example "is this feature meaningful to the business?") needs a human, not a loop.

---

## 6. Build plan

### Phase 0: groundwork and the coworker spike (1 day)
- Repo skeleton, Ames download, seeded defect-injection script, answer key, time split, held-out set stored outside the repo.
- Unpack the skill; apply the `feature_check.py` fixes from section 1.1.
- **Grok Bot spike (2 hours, timeboxed):** do you have access, can it push a branch to a throwaway repo with a scoped token, can it be driven by API, can you export what it did? The result decides the guest seat (section 3.5). Also note Goose or OpenHands install time.

### Phase 1: stage 1 end to end with the real hand-off (2 days)
- `gates/run_gate.py` (path guard, verifiers from `main`), `data_quality_check.py`, `programs/01_data_prep.md`, the data-prep subagent, the Claude Code adapter.
- *Exit test:* the agent reaches a pass; a scripted row-dropping branch is rejected; a scripted branch editing `verifiers/` is rejected.

### Phase 2: features (1 day)
- Feature subagent, contract between stages 1 and 2, the planted leaks.
- *Exit test:* the blatant leak is rejected, legitimate features are kept, the subtle leak passes (on purpose).

### Phase 3: modelling and evaluation gate (2 days)
- `training_check.py`, `evaluation_gate.py` (time-ordered held-out set, price bands), routing back with retry caps.
- *Exit test:* the subtle leak and the price-band failure are caught here.

### Phase 4: orchestrator and run report (2 days)
- Resume from the last passed gate, global budget, HTML report.
- **Milestone: the minimum viable demo (about 1.5 weeks part-time).**

### Phase 5: coworker seats (2 to 3 days)
- Guest-proposer adapter (Goose or OpenHands, plus `manual.md` or an API adapter for Grok Bot), CI version of the gate runner posting `verdict.json` on PRs.
- OpenClaw approval hook for the retrain.

### Phase 6: monitoring and deployment (1 day)
- Drift on 2009 to 2010 data, retrain trigger behind the approval, canary replay for the README.

### Phase 7: experiments and showcase (2 to 3 days)
- 5 runs per proposer for stage 2, self-grading A/B, tamper results, scorecard, comparison table, recording, README, limits.

**Total:** about 3 weeks part-time. Stop after Phase 4 if time is short; that version already proves the claim.

---

## 7. Risks and how to handle them

| Risk | Handling |
|---|---|
| **CI runs the branch's own (tampered) verifier** | The gate runner always loads verifiers and thresholds from `main`; the path guard fails any branch touching `verifiers/` or `gates/` |
| Proposer peeks at the held-out set | Keep it outside the repo; only the gate runner can read it |
| Agent-supplied code runs in the verifier | No `eval()`; candidate code runs in a step with no secrets or network |
| Coworker credentials leak (shared computer) | Throwaway repo, fine-grained token for that repo only, no other sign-ins |
| Metric gaming (dropping rows, overfitting CV) | Row-count check, time-ordered held-out evaluation |
| Nondeterministic results presented as facts | About 5 runs per proposer, report rates |
| Live demo fails | Record a golden run; ship the log and report for replay |
| Grok Bot unavailable or not scriptable | Decided by the Phase 0 spike; OpenHands takes the seat |
| Runaway cost or loops | Caps enforced in the orchestrator; cost per stage in the report |
| Scope creep (graph, more coworkers) | Hold the line at "chain with gates" and one guest stage |
| Product claims go stale | Date the comparison; quote vendor docs, not blogs |

---

## 8. Decisions needed from you

1. **Confirm the roles:** Claude Code as reference proposer, a coworker as guest proposer on stage 2, OpenClaw for approval. Or tell me if the coworker must drive more stages.
2. **Grok Bot access:** do you have it? This decides whether the Phase 0 spike is worth running.
3. **Open control:** Goose (lighter, recipes) or OpenHands (Docker sandbox, closer to a "coworker")? I lean OpenHands, because it also serves as the Grok Bot fallback.
4. **Audience:** a talk or recording, a GitHub portfolio piece, or an internal demo? This sets how much polish goes into the report versus the README.
5. **Start now?** I can do Phase 0 (except the Grok Bot spike, which needs your account) and Phase 1.

---

## Appendix A: sources

**In this folder:** both decks (stage definitions, exit conditions, Karpathy loop, design checklist); `loop-engineer-skill.skill` (all files read); `Loop_engr_notes.rtf`; `graphengg/` (I read pages 1 and 5 in detail; the other ten pages are unread, and the paper's claims are its author's); `ai-coworkers.jpeg`; `when_to_loopengr.pptx` (nearly empty).

**Web (most are third-party; verify before presenting):**
- Grok Bot: [xAI docs](https://docs.x.ai/grok-bot/overview), [announcement](https://x.ai/news/introducing-grok-bot), [security commentary](https://michaelbennett.co/blog/grok-bot-shared-computer), [explainer](https://www.mindstudio.ai/blog/what-is-grok-bot)
- OpenClaw: [docs index](https://docs.openclaw.ai/llms.txt), [ClawHub overview](https://dev.to/hex_agent/clawhub-the-skill-registry-for-openclaw-agents-591c)
- Hermes Agent: [docs](https://hermes-agent.nousresearch.com/docs), [PyPI](https://pypi.org/project/hermes-agent)
- Goose: [recipe reference](https://block.github.io/goose/docs/guides/recipes/recipe-reference), [Agentic AI Foundation](https://aaif.io/blog/how-goose-is-building-a-distributed-ai-teammate-ecosystem)
- OpenHands: [SDK Docker sandbox](https://docs.openhands.dev/sdk/guides/agent-server/docker-sandbox)
- Claude Cowork: [getting started](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork), [scheduled tasks](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)

## Appendix B: what changed in revision 2

| # | Change | Why |
|---|---|---|
| 1 | Merged the coworker section into sections 0, 3, 5 and 6; removed the stale Grok row and section 3.5 | Revision 1 contradicted itself: the TL;DR said "no hosted coworker at the core", section 9 made Grok Bot the headline for stages 1 to 3 |
| 2 | Grok Bot moved from proposer on stages 1 to 3 to **guest proposer on stage 2**, gated by a 2-hour spike | Its scriptability is unconfirmed; one stage gives the story with a fraction of the risk |
| 3 | **Verifiers always run from `main`**, plus a path guard | Revision 1's CI would have run a PR's own tampered verifier; branch protection only stops the merge |
| 4 | **Held-out set moved outside the repo** | Hashing detects edits, not peeking |
| 5 | One hand-off mechanism (branches) from Phase 1 | Avoids rebuilding the hand-off when coworkers join |
| 6 | **Ames with a time split**; the "looks good, isn't" feature replaced by a target-encoding leak caught at the evaluation gate | Revision 1's version could not be produced with shuffled CV; the new one demonstrates defence in depth and gives real drift |
| 7 | `eval()` upgraded from "fine locally" to must-fix; docstring bug noted | Coworker-supplied strings would execute in CI |
| 8 | Experiments need about 5 runs; the tamper test is scripted, plus a count of spontaneous attempts | One run of a nondeterministic agent is an anecdote; prompting a bot to cheat is staging |
| 9 | Demo script reworked to include the coworker and the phone approval; deployment moved to the README | It had no coworker beat and was already full |
| 10 | Timeline 1.5 to 2 weeks → about 3 weeks, with a minimum viable cut after Phase 4 | The coworker work was never added to the estimate |
