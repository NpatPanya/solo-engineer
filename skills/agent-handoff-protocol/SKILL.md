---
name: agent-handoff-protocol
description: Structured handoff-packet format used whenever engineer (@solo:engineer) dispatches work to deep-specialist, and the format deep-specialist uses to report back. Replaces free-form delegation prose. Includes the "flag a gap, don't invent" rule. (builder is a separate, standalone entry-point agent and is not part of this handoff relationship — see agents/builder.md.)
---

# Agent Handoff Protocol

There is exactly one subagent in this project: `deep-specialist`. Every dispatch to it, and every
report back from it, uses this one packet shape — nothing is composed freehand.

```yaml
handoff:
  objective: <one sentence>
  from: engineer | deep-specialist
  to: deep-specialist | engineer | user
  status: dispatched | complete | blocked | needs_clarification | rejected
  priority: P0 | P1 | P2
  sub_plan_ref: <e.g. "Plan 1 / Sub 1.2", or "n/a" for ad-hoc work>
  inputs: [<file path or inline reference>]
  constraints: [<explicit "do NOT" boundary>]
  produced_artifacts: [{path, description}]
  definition_of_done: <concrete, checkable criterion>
  notes: <deviations, open questions, escalation target if blocked>
```

## Rules

- `engineer` (`@solo:engineer`) fills this in to dispatch; `deep-specialist` fills in the same
  shape to report back. One vocabulary in both directions — a handoff is forwarded, not
  re-composed at each relay.
- `sub_plan_ref` ties every dispatch back to the plan-mode-protocol notation (Rule 3's `Sub n.k`
  numbering) when the work originated from a plan. This is what lets `engineer` know which phase
  just closed and which permission gate (Rule 4) to pause at next.
- **Flag a gap, don't invent.** If `deep-specialist` hits missing information, an ambiguous
  requirement, or a decision only the user can make, it sets `status: needs_clarification` and
  states exactly what's missing in `notes` — it does not guess and proceed. `engineer` does the
  same when it can't fill in the packet's `objective` or `definition_of_done` confidently.
- Dispatches are **strictly sequential** — one active `deep-specialist` invocation at a time, its
  packet returned and reconciled before the next is opened. This mirrors Rule 3 of
  plan-mode-protocol; there is no parallel or background dispatch in this project.
- `deep-specialist` never talks to the user directly and never opens a second-order subagent — it
  only exchanges packets with `engineer`, which remains the single point of contact with the
  user.
- `builder` is not part of this protocol at all — it's a separate entry-point agent the user
  invokes directly, not something `engineer` dispatches or receives packets from. Don't compose a
  handoff packet for it.
