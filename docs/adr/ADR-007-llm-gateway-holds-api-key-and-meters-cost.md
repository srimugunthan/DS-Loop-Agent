# ADR-007: An LLM gateway holds the API key and meters cost per experiment

**Status:** Accepted (revises the original answer to grilled Q20, which put the key in the agent container and deferred the gateway to Phase 2)
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

Agents run inside a per-project container that also runs the code they write (ADR-002). Any secret in that container's environment can be read, and printed to logs, by that code. Logs are saved in the experiment record, mirrored to MLflow and shown in the file browser during demos (ADR-004, decision 34). Each loop also has a dollar cap (decision 35). Without a single point that sees every model call, that cap could only be checked between experiments, so a loop could overshoot it. Phase 2 will run third-party coworkers in the same container slot, and they must never hold the key. The gateway costs nothing to run locally and takes about a day to build.

---

## Decision

We will build an LLM gateway in Phase 1: a small custom FastAPI container, about 150 lines with no database, that is the only component holding the Anthropic API key.

- **Agent side:** agent containers have no internet access and only a placeholder key. Their SDK points at `ANTHROPIC_BASE_URL=http://gateway:8080/exp/<experiment-id>`.
- **Forwarding:** the gateway forwards only model requests to `api.anthropic.com` and streams replies through unbuffered.
- **Metering:** it reads token usage from each reply, prices it and records tokens and cost against the experiment.
- **Cap:** it refuses requests once the loop's dollar cap is reached.

In dev mode it runs as a local process.

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Custom gateway (chosen)** | The key never enters any sandbox; the dollar cap is enforced per request; cost is recorded per experiment straight into our experiment record; no extra infrastructure; reused unchanged for Phase 2 coworkers | About a day to build and test; streaming passthrough and reading usage from the stream must be done carefully; a model price table to maintain |
| Key in the agent container, egress proxy allowing only the API (original answer) | Least work | Agent-written code can read the key and leak it into logs and MLflow; the cap can only be checked between experiments; not acceptable for Phase 2 |
| LiteLLM proxy | Off the shelf; about half a day of setup | Budgets are per virtual key, not per experiment, so mapping code is still needed; usually needs Postgres for spend tracking; another component to version and debug |

---

## Consequences

**Positive:**
- No real key exists in any agent or check-runner container, in either phase.
- The dollar cap stops a loop at the request that crosses it, instead of up to one experiment late.
- Token and cost figures in `experiments.jsonl` and MLflow come from one trusted place, not from what the agent reports.
- Comparisons in Phase 2 get the same cost metering for every coworker that uses the gateway.

**Negative / Accepted Tradeoffs:**
- The gateway is a single point of failure for model calls. When it is down, calls fail and are handled like API errors: the loop pauses with Resume (decision 39).
- Each request has an extra local hop, which is negligible next to model latency.
- Prices are hard-coded per model and must be updated when prices change.

---

## Compliance / Risk Notes

- Demo-only key with a monthly spend limit in the Anthropic Console, and masking of `sk-ant-…` strings in logs, are kept as defence in depth.
- The gateway listens only on the internal Docker network (dev mode: `127.0.0.1`).

---

## Follow-up Actions

- [ ] Build the gateway with tests for streaming passthrough, usage extraction and cap refusal (P1-M1).
- [ ] Keep a model price table (Haiku 5.5, Sonnet 5.5, Opus 5.5) in configuration.
- [ ] Phase 2: confirm each coworker's model client can be pointed at the gateway; for any that can't, decide case by case.
