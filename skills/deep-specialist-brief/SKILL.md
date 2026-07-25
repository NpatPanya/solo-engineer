---
name: deep-specialist-brief
description: Decision rule for when engineer (@solo:engineer) should delegate to deep-specialist versus handle work itself, and how to brief that dispatch. This is the only delegation path in the project — there is one subagent.
---

# Deep Specialist Brief

`engineer` (`@solo:engineer`) is the only role that talks to the user. `deep-specialist` is the
only subagent it can dispatch to. This skill governs that one decision point.

## When to delegate

Delegate a `Sub n.k` (or an ad-hoc request) to `deep-specialist` when at least one is true:

- The work would consume most of the remaining working context on its own (a large trace, a wide
  multi-file investigation, a long research pass) and `engineer` still needs context left to
  reconcile results and talk to the user.
- The work benefits from a clean, unbiased context — e.g. an independent second look at a design
  or a diff, where `engineer`'s own accumulated assumptions would bias the result.
- The task is explicitly deep/isolated by nature (deep research, an exhaustive audit, a large
  refactor trace) rather than a normal incremental build step.

## When NOT to delegate

- Small, well-understood steps `engineer` can finish in a few tool calls — dispatch overhead
  (fresh context, a full handoff packet, re-reading inputs) costs more than it saves.
  `codebase-recon` and `safe-build` cover this in-place work.
- Anything that requires back-and-forth with the user mid-task — `deep-specialist` never talks to
  the user; it only exchanges packets with `engineer`.

## How to brief it

1. Compose a full `agent-handoff-protocol` packet: one-sentence objective, explicit inputs,
   explicit constraints (what NOT to touch), and a concrete `definition_of_done`.
2. Set `sub_plan_ref` if this dispatch is executing a numbered sub-plan, so the returned packet
   can be reconciled against the right phase.
3. Dispatch once, wait for the returned packet — never dispatch a second dispatch before the first
   resolves (strict sequencing applies to subagent calls too, not just to sub-plans).
4. On return, reconcile: if `status: complete`, fold `produced_artifacts` into the sub-plan's
   record and proceed to the Rule 4 permission gate. If `status: blocked` or
   `needs_clarification`, surface that to the user verbatim rather than smoothing it over.
