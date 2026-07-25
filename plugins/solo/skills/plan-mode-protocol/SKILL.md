---
name: plan-mode-protocol
description: Governs how engineer (@solo:engineer) produces and executes any plan when plan mode is invoked. Enforces four mandatory rules — modular sub-division, explicit priority, strict sequencing, and incremental delivery gated on user approval. This is the canonical, always-preloaded ruleset for plan mode; it is never optional and never overridden by task-specific convenience.
---

# Plan Mode Protocol

This skill activates the instant plan mode is entered (`ExitPlanMode`, a user request to "plan
first," or any request whose scope requires a plan before execution). It applies regardless of
task domain — feature work, refactors, research, migrations, anything. Four rules are mandatory,
in order. None may be skipped, merged away, or reordered. If a task seems too small to need all
four, it is still small enough to satisfy all four cheaply — do not treat size as an exemption.

## Rule 1 — Modular Sub-Division (decouple information)

Never write a complex plan into a single information file. Split the plan by module or by
decoupled context: each split-off piece stands on its own — a reader opening only that piece
should understand what that piece is for and how to do it, without first reading every other
piece. At the same time, the pieces must remain **related when read together** — consistent
naming, consistent numbering, and cross-references back to the index file, so the full plan
reassembles cleanly from its parts.

Practical rule of thumb: split by module/component/phase boundary, not by arbitrary line count.
A split is correct when each piece answers "what is this piece responsible for" in one sentence,
and wrong when two pieces both need to be open at once to make sense of either.

Concretely, this means:

- One **index file** (the single "information file" the user reads first) that lists every
  sub-plan, its priority, and its sequence position — but does **not** inline the sub-plans'
  full working detail.
- One **sub-plan file per module/phase**, each self-contained: its own objective, its own steps,
  its own definition of done, its own inputs/outputs.
- The index cross-references sub-plan files by name/number; sub-plan files cross-reference the
  index and their immediate predecessor/successor, never the full set.

## Rule 2 — Explicit Priority

Every plan, and every sub-plan inside it, always carries a visible priority marker. Never leave
priority implicit in ordering alone — state it.

Use this scale unless the user specifies another:

- **P0 — Blocking**: nothing else can proceed until this is done.
- **P1 — Required**: needed for the stated goal, not blocking other P1s.
- **P2 — Enhancement**: improves the result but the goal is met without it.

Priority is marked on the index file per sub-plan, and restated at the top of each sub-plan file
itself, so the priority is visible whether the reader opens the index or the piece directly.

## Rule 3 — Strict Sequencing

Sub-plans execute in a strict, declared order — never in parallel, never "whichever is
convenient." The order is recorded in a **single index/notation file** using this exact shape:

```
Project #<n>
Plan <n>. <Plan title>
  -> Sub <n>.1  <one-line action> [P<priority>]
  -> Sub <n>.2  <one-line action> [P<priority>]
  -> Sub <n>.3  <one-line action> [P<priority>]
  ...
```

This notation file is the only place the *entire* sequence is visible end-to-end; it stays a
lightweight table of contents (satisfies Rule 1 by not inlining sub-plan detail). Each `Sub n.k`
line links to its own sub-plan file. Sequencing is a strict total order: `Sub n.2` never starts
before `Sub n.1` reports done, even if they touch unrelated modules — sequencing is about
execution order, not about topical independence (which Rule 1 already handles).

## Rule 4 — Incremental Delivery (phase-gated, permission-gated)

Never execute the whole sequence in one uninterrupted pass, and never load every sub-plan's full
context into the working context window at once.

For each `Sub n.k` in turn:

1. Load only that sub-plan's file (plus the index for orientation). Do not pre-load Sub n.k+1's
   detail.
2. Execute exactly that sub-plan's scope.
3. Report what was done, against that sub-plan's definition of done.
4. **Stop and ask the user for explicit permission/agreement before starting the next sub-plan.**
   A go-ahead is required every time — silence, a prior blanket "sounds good," or the user simply
   not objecting do not count as consent to proceed. If the user asked for the whole plan
   up-front, that authorized planning it, not executing every phase unattended.

Exception: the user may explicitly grant standing approval for a named run ("execute all of Plan
2 without stopping between sub-plans") — this must be an explicit, current statement, scoped to
that plan, and does not carry over to a different plan or a future session.

## Applying all four together (minimal worked shape)

```
plans/
  INDEX.md                 <- Rule 1 (single lightweight notation file) + Rule 3 (sequence table)
  plan-1-auth/
    00-overview.md          objective, priority P0, definition of done
    sub-1.1-schema.md       [P0]
    sub-1.2-api.md          [P0]
    sub-1.3-ui.md           [P1]
  plan-2-notifications/
    00-overview.md          objective, priority P1
    sub-2.1-queue.md        [P1]
    sub-2.2-templates.md    [P2]
```

`INDEX.md` contains the `Project #1 / Plan 1. -> Sub 1.1 ... / Plan 2. -> Sub 2.1 ...` notation
from Rule 3, with a `[P#]` tag per line per Rule 2. Each `sub-*.md` is independently readable per
Rule 1. Execution walks the notation top to bottom, one `Sub` at a time, stopping for permission
between each per Rule 4.

## Failure modes this prevents

- A single giant plan file that must be fully re-read on every turn (violates Rule 1 — this is
  the specific failure this ruleset exists to stop).
- Priorities the reader has to infer from paragraph order (violates Rule 2).
- Two sub-plans dispatched "in parallel because they're unrelated" (violates Rule 3 — relatedness
  of topic is not the same axis as order of execution).
- Running phase 2, 3, and 4 back-to-back because phase 1 went fine (violates Rule 4 — approval is
  per-phase, not earned once).
