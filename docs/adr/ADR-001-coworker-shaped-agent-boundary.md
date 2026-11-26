# ADR-001: Agents exchange files with the platform, not tool calls

**Status:** Accepted
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

Phase 1 runs a skills-driven agent built on the Claude Agent SDK. Phase 2 must replace the stage agents with AI coworkers (OpenHands, Goose, Grok Bot) that bring their own harness and tools, with no UI changes (PRD Phase 1 §10.3, PRD Phase 2 §7). The platform, not the agent, must own the loop, apply changes, run checks and keep the log (decisions 4, 6). The question was how the agent and the platform exchange work so that a coworker can later sit where the skills-driven agent sits (grilled Q2, revised; Q4).

---

## Decision

We will make every agent "coworker-shaped". The agent sees only a workspace folder, a brief and the loop program, and it returns results as files in a fixed format. In loop mode it writes one `proposal.json` per iteration, describing exactly one change:
- stage 1: one cleaning-rule function;
- stage 2: add or remove one feature;
- stage 3: one search plan.

The agent gets no custom tools that call backend functions. The platform reads the files, applies the change, runs the checks and logs the result. The adapter interface (`start`, `send`, `propose`, `events`, `artifacts`, `stop`, `capabilities`) wraps this exchange.

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Files in, files out; no backend tools (chosen)** | Any coworker that can read and write a folder can plug in; the platform keeps full control of applying and judging; every proposal is a file that can be inspected and logged | The proposal format must be validated, and malformed proposals become `invalid` experiments; the agent can't ask the platform things mid-task (e.g. "run the checks now") |
| Custom tools exposed to the agent (e.g. `run_checks`, `apply_change`) through the SDK or MCP | Richer interaction; the agent gets feedback mid-task | Ties Phase 1 to the Claude Agent SDK's tool mechanism; every coworker would need the same tools, and many can't take custom tools; blurs who applies changes |
| Claude Code run headless | Less code to write | Harder to embed and stream; the same coupling problem for Phase 2 |

---

## Consequences

**Positive:**
- A Phase 2 coworker is a new adapter only. It runs in the same agent container and workspace and returns the same files.
- The loop runner stays the single owner of apply, check, log and stop. The agent cannot keep a change itself.
- Every proposal and its result are on disk for review, comparison and the MLflow mirror.

**Negative / Accepted Tradeoffs:**
- Feedback reaches the agent only between turns or iterations, as files or messages, not as live tool results.
- The platform owns a schema per stage for `proposal.json`, plus its validation.
- Chat mode edits files freely, so its changes are recorded as revisions, not one-change proposals.

---

## Follow-up Actions

- [ ] Define and version the `proposal.json` schema for stages 1–3.
- [ ] Review the interface against OpenHands and Goose before P1-M2 ends (PRD Phase 1 §16).
