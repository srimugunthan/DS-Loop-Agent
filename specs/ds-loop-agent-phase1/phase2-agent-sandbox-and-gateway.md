# Phase 2: Agent Sandbox and LLM Gateway

**Depends on:** `phase1-foundation.md`
**Status:** Not started

## Design components touched

LLM gateway (§2, §4 model calls / cost reporting; ADR-007, D35, D36), agent container (§2; ADR-002, D27), agent interface + skills-driven adapter (§4; ADR-001, D10, D19, D26), dev mode (§2; D23), security (§8).

## Objective

Replace the fake agent with a real skills-driven agent (Claude Agent SDK, Haiku 5.5 by default), running in a per-project container with no internet and only a placeholder API key. All model calls go through the LLM gateway, which holds the real key, records tokens and cost per experiment, and refuses requests once a cap is reached. The backend talks to the agent only through the agent interface, exchanging files. The same works without Docker in dev mode, behind a visible banner.

## Independently testable when

- In Docker mode, the user chats with a real agent in a sandbox stage workspace; the agent writes a file that appears in the file browser.
- From inside the agent container, any request to the internet fails, and the environment holds no real key.
- Each turn's tokens and cost appear in the UI and in the log; setting a tiny cap produces a clear "cap reached" refusal.
- `start.sh` dev mode runs the same chat with the "Unsandboxed — dev mode" banner.

## Tasks

### 2.1 LLM gateway: passthrough, key injection, metering

- **Type:** AFK
- **Blocked by:** 1.1
- **PRD reference:** §9 (API key handling)
- **System design reference:** §4 model calls, cost reporting; ADR-007, D36

**Scope:**
- **Forwarding:** a small FastAPI service. Requests to `/exp/<id>/v1/...` are forwarded to `api.anthropic.com` with the real key from `.env` replacing the placeholder; streamed replies pass through unbuffered.
- **Metering:** token usage is read from the reply (including streamed events), priced from a per-model price table in configuration, and recorded against the experiment id.
- **Cap:** requests are refused once the experiment's loop cap is reached.

Tests run against a mock upstream. Not included: the agent container (2.2).

**Acceptance criteria:**
- [ ] Streaming test: events arrive incrementally, not buffered.
- [ ] Usage and cost records are correct for streamed and non-streamed replies; non-model endpoints are refused.
- [ ] With a cap set, the request that crosses it completes and the next one is refused with a distinguishable error.
- [ ] Verifiable in isolation with the mock upstream.

### 2.2 Agent container and internal network

- **Type:** AFK
- **Blocked by:** 2.1
- **PRD reference:** §9 (sandbox)
- **System design reference:** §2 agent container; ADR-002, D27

**Scope:** Docker Compose with an internal network. The agent container image includes the Agent SDK and the Python data-science stack the skills need. Its only route out is the gateway. It gets `ANTHROPIC_BASE_URL=http://gateway:8080/exp/<id>`, a placeholder key, and a stage workspace mount. Not included: the adapter (2.3).

**Acceptance criteria:**
- [ ] From the container, requests to arbitrary internet hosts fail; requests to the gateway succeed.
- [ ] The container environment contains no real key; only the workspace is mounted.
- [ ] Verifiable in isolation.

### 2.3 Agent interface + skills-driven adapter: real chat end to end

- **Type:** AFK
- **Blocked by:** 1.2, 2.2
- **PRD reference:** §10.1, §10.3
- **System design reference:** §4 agent interface, agent ↔ platform exchange; ADR-001, D10, D19, D26

**Scope:**
- **Interface:** define `start`, `send`, `propose`, `events`, `artifacts`, `stop`, `capabilities`. `propose` is a stub until Phase 5.
- **Adapter:** the skills-driven adapter starts an Agent SDK session in the agent container with a brief file, relays messages, and maps SDK events (text, status, tool activity, file written, errors, done) onto the SSE stream.
- **Model setting:** the model and effort are configurable per stage and mode, defaulting to `claude-haiku-5-5`.

Not included: stage skills (later phases).

**Acceptance criteria:**
- [ ] A chat in a sandbox stage produces agent replies and a file the agent wrote, visible live in the file browser.
- [ ] The backend never calls a custom tool exposed to the agent: the exchange is files plus messages only.
- [ ] Every log entry records the backend id and version (D14).
- [ ] Demoable in isolation.

### 2.4 Cost in the UI and the cap-refusal path

- **Type:** AFK
- **Blocked by:** 2.1, 2.3
- **PRD reference:** §8.3 (dollar cap)
- **System design reference:** §4 cost reporting; D35, D36

**Scope:** Show cost per turn and per session from the gateway's records. When the gateway refuses a request, show a clear message in the chat. (The loop runner's handling of this comes in Phase 5.)

**Acceptance criteria:**
- [ ] Displayed cost matches the gateway's records.
- [ ] With a tiny cap, the refusal is shown plainly and logged.
- [ ] Demoable in isolation.

### 2.5 Dev mode without Docker

- **Type:** AFK
- **Blocked by:** 2.3
- **PRD reference:** §12 (dev mode note)
- **System design reference:** §2 dev mode; D17, D23

**Scope:** `start.sh` also starts the gateway as a local process on `127.0.0.1`, and the adapter runs the agent as a local subprocess restricted to the workspace folder. The UI shows an "Unsandboxed — dev mode" banner.

**Acceptance criteria:**
- [ ] The same chat as in 2.3 works with `start.sh`.
- [ ] The banner is visible on every screen in dev mode and absent in Docker mode.
- [ ] Demoable in isolation.

### 2.6 `.env` handling and key masking in logs

- **Type:** AFK
- **Blocked by:** 2.1
- **PRD reference:** §9 (API key handling)
- **System design reference:** §8; D36

**Scope:** `.env` is git-ignored and read only by the gateway. A masking filter replaces anything matching `sk-ant-…` in every log before it is saved, and later before MLflow mirroring. Document the use of a demo-only key with a monthly spend limit.

**Acceptance criteria:**
- [ ] Test: agent output containing a key-like string is masked in every saved log.
- [ ] The key appears nowhere in project folders.
- [ ] Verifiable in isolation.
