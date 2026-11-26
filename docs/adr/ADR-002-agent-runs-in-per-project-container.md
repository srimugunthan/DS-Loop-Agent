# ADR-002: Run each project's agent process inside its own container

**Status:** Accepted
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

The Agent SDK's built-in tools (read, write, edit, run commands) act on whatever machine the agent process runs on. Agent code must run in a sandbox: no network, no secrets, and only the project's files (PRD Phase 1 §9, decision 9). The backend holds the held-out set location and the check code, and in Phase 2 it will hold coworker credentials. The app is local and single-user, started with Docker Compose, plus a `start.sh` dev mode that runs without Docker (decisions 17, 23).

---

## Decision

In Docker mode, we will run the agent process itself, not just the code it writes, inside one agent container per project:
- only the stage workspace is mounted;
- there is no internet access: the only network route is to the LLM gateway ([ADR-007](ADR-007-llm-gateway-holds-api-key-and-meters-cost.md));
- the backend starts, stops and talks to the container.

The container holds only a placeholder API key. The real key stays in the gateway (decision 36). In dev mode the agent runs as a local subprocess restricted to the workspace, and the UI shows an "Unsandboxed — dev mode" banner.

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **The agent process runs inside a per-project container (chosen)** | Everything the agent does, including its own shell commands, is confined; matches how coworkers such as OpenHands run, so Phase 2 plugs into the same slot | Container lifecycle to manage; the backend talks to the agent across a process boundary; model calls depend on the gateway being up |
| Agent inside the backend, built-in file and shell tools off, code runs through a custom tool that uses a container | Simple agent hosting | Needs a custom backend tool, which ADR-001 rules out; the agent process sits next to the backend's secrets |
| Agent inside the backend, built-in tools limited by permission rules | No containers for the agent | Relies on rules inside the same process that holds the held-out data and check code; weak isolation |

---

## Consequences

**Positive:**
- The agent cannot read the held-out set, the check code or other projects, because they aren't in its filesystem.
- Phase 2 coworkers run in the same container slot with the same mounts.

**Negative / Accepted Tradeoffs:**
- The agent can reach the outside world only through the gateway, so any tool or package the agent needs at run time must already be in the container image.
- Dev mode gives up isolation. This is accepted because it is labelled in the UI and not used for demos or comparisons.

---

## Compliance / Risk Notes

- Public datasets only (decision 16). A sandbox escape would expose public data, not sensitive data, and no real API key is inside the container.

---

## Follow-up Actions

- [ ] Bake the Python data-science stack the skills need into the agent container image, since it has no internet access.
