# ADR-005: Chat and loop modes edit one shared, itemised stage structure

**Status:** Accepted
**Date:** 2026-11-27
**Deciders:** Project owner (DS-Loop-Agent)

---

## Context

Each of stages 1–3 can be worked in chat mode (the agent edits code freely) or loop mode (exactly one change per iteration, applied by the platform; decisions 7, 22). Users switch between modes mid-stage, and both modes must end in the same review and approval step (PRD Phase 1 §8.7). The mockup shows a single readable script per stage (`data_cleaning.py`). If the two modes stored code differently, every switch would need a conversion, and loop proposals couldn't be applied deterministically to free-form chat code (grilled Q12).

---

## Decision

We will store each stage as an itemised structure that both modes edit:
- stage 1: a folder of cleaning rules, one file per rule, plus an ordered list;
- stage 2: one file per feature, plus a list;
- stage 3: a model config.

Chat mode edits these files freely. Loop mode adds or removes one item per iteration. The platform generates the combined readable scripts (`data_cleaning.py`, `features.py`, `train.py`) for viewing and export. Each rule or feature becomes a function in list order, with a `run(df)` entry point and a header naming the version and content hash. Export adds `pipeline.py` and `requirements.txt` (decision 38).

---

## Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **One shared itemised structure (chosen)** | Switching modes needs no conversion; the platform applies a loop change by adding or removing one file; every change (chat or loop) is recorded the same way; generated scripts still match the mockup | The agent must work within the structure in chat mode, not write one big script; the script generator is extra platform code |
| Free-form script in chat; the agent splits it into the structure when switching to loop | Natural for chat | Splitting is agent-driven and error-prone, and needs its own check; two representations |
| Separate modes: loops ignore chat code | Simple to build | Breaks "a loop starts from the stage's current version"; work is lost when switching |

---

## Consequences

**Positive:**
- "One change per iteration" can be enforced and diffed precisely.
- Undoing a change means not promoting it. Nothing is rewritten.
- The exported `pipeline.py` and the saved scikit-learn `Pipeline` (decision 37) are built from the same items.

**Negative / Accepted Tradeoffs:**
- The skills have to teach the agent the structure, and malformed items are rejected.
- The generated scripts are read-only views. Editing them by hand would not round-trip back into the structure.
