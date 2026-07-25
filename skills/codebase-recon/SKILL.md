---
name: codebase-recon
description: Methodology for tracing how existing code works and assessing the blast radius of a change before planning or building it. Used by engineer (@solo:engineer) directly, or delegated to deep-specialist for a deep, isolated trace when the surface area is large.
---

# Codebase Recon

Use before writing any plan sub-plan (see `plan-mode-protocol`) that touches existing code, and
before any bug-fix or refactor.

## Steps

1. **Locate** — find every file that defines, calls, or configures the thing in question. Prefer
   targeted search (grep/glob for the symbol or string) over reading whole directories.
2. **Trace** — follow the call graph or data flow far enough to answer: what depends on this, what
   does this depend on, and what would break if it changed.
3. **Classify blast radius** — LOW (isolated, one module, no external contract), MEDIUM (crosses
   module boundaries, no public API), HIGH (touches a public API, shared schema, auth, or data
   migration).
4. **Write findings to a durable file**, not just packet prose, when the trace is non-trivial —
   downstream steps (a sub-plan's own file, per Rule 1 of plan-mode-protocol) read the file once
   instead of a paraphrase degrading through relays.

## When to delegate to deep-specialist

Delegate the recon itself (not just the resulting write-up) when the trace spans many files or
an unfamiliar area and would otherwise consume most of the working context before any plan gets
written. Package the ask with `agent-handoff-protocol`, give `deep-specialist` a narrow objective
("trace all callers of X and classify blast radius"), and have it return the packet with
`produced_artifacts` pointing at its findings file.

Do not delegate recon that's small enough to finish in a few targeted searches — that overhead
isn't worth a second context window.
