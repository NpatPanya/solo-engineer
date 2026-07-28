---
name: safe-build
description: Implementation, mechanical-refactor, and test-after methodology engineer (@solo:engineer) applies internally while executing a sub-plan. Keeps building, refactoring, and testing as one in-place discipline instead of separate agent dispatches.
---

# Safe Build

Applied while executing any single `Sub n.k` from an active plan (see `plan-mode-protocol`).
Covers three disciplines in one pass so a normal implementation step doesn't need a second agent
dispatch for cleanup or testing.

## 1. Implement to the sub-plan's definition of done

Build only what that sub-plan's file specifies — not the whole plan, not adjacent sub-plans
(Rule 4 of plan-mode-protocol: one phase loaded and executed at a time). All implementation,
frontend or backend, follows `coding-architecture-standards` (SRP + Hexagonal Architecture) — run
its pre-done checklist before step 3's tests, not after.

## 2. Mechanical refactor, scoped and behavior-preserving

If the step requires touching existing code beyond the new addition, keep refactors mechanical
(renames, extractions, dead-code removal) and behavior-preserving. Anything that changes
observable behavior is a design decision, not a refactor — surface it back to the user rather
than folding it silently into a "cleanup."

## 3. Test after, edge-case-first

Write and run tests against the code just implemented (test-after, not test-first — a deliberate
workflow choice). Prioritize edge cases and failure paths over the happy path, since the happy
path is usually what got exercised while writing the code.

## Escalation

If implementation surfaces something outside this sub-plan's scope — a missing dependency, an
architectural question, a decision only the user can make — stop, do not guess, and report it
back per the "flag a gap, don't invent" rule in `agent-handoff-protocol`. Do not silently expand
the sub-plan's scope to cover it.
