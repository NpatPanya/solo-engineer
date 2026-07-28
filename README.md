# solo

A **2 entry-point agents + multi-skill + 1 subagent** engineering plugin, adapted down from a
9-role reference design. There are two independent, human-invoked entry points — `engineer`
(`@solo:engineer`) and `builder` (`@solo:builder`, colloquially "the worker") — plus one subagent,
`deep-specialist`, that only `engineer` can dispatch. `engineer` and `builder` are peers, not a
caller and its subagent: `engineer` never dispatches `builder`, and the two are never active on the
same plan at the same time. The user decides which one to invoke for a given piece of work.

- **`engineer`** carries `plan-mode-protocol` embedded/preloaded, so it can autoplan and then
  execute the resulting plan itself, end to end, in one session — it's fully self-sufficient. It
  can also delegate isolated deep-dive work (large recon, independent review, exhaustive research)
  to its one subagent, `deep-specialist`.
- **`builder`** does not plan and never invokes `plan-mode-protocol`. It reads an already-saved,
  already human-approved plan (produced earlier by `engineer`) and executes it, one `Sub n.k` per
  invocation, on a cheaper/faster model — the token-optimized execution path for when you don't
  want `engineer`'s (more expensive) model running the mechanical build work.

## Roster

| Role | Agent name | Model | Effort | Responsibility |
|---|---|---|---|---|
| Engineer (entry point) | `engineer` (`@solo:engineer`) | sonnet | medium | Classifies requests, recons, autoplans, executes directly or delegates recon/review, reconciles results, reports to the user |
| Builder (entry point) | `builder` (`@solo:builder`) | haiku | low | Reads an already-approved plan and executes exactly one `Sub n.k` per invocation, token-optimized — never plans, never talks about anything but the assigned sub-plan |
| Specialist (subagent) | `deep-specialist` | sonnet | high | Isolated deep-dive work — large trace/recon, independent review, exhaustive research — dispatched by `engineer` one at a time, never talks to the user |

## Skills

| Skill | Preloaded on | Purpose |
|---|---|---|
| `plan-mode-protocol` | `engineer` | Mandatory 4-rule planning ruleset plus a human-approval checkpoint before any execution (see below). Embedded in `engineer`; `builder` never invokes it. |
| `agent-handoff-protocol` | `engineer`, `deep-specialist` | Structured packet format for every `engineer` ↔ `deep-specialist` dispatch. Not used by `builder` — it isn't dispatched, it's invoked directly. |
| `codebase-recon` | `engineer`, `deep-specialist` | Tracing existing code and blast-radius assessment |
| `safe-build` | `engineer`, `builder` | Implementation + mechanical refactor + test-after methodology, applied by whichever agent is executing a step |
| `deep-specialist-brief` | `engineer` | Decision rule for when to delegate to `deep-specialist`, and how to brief the dispatch |
| `coding-architecture-standards` | `engineer`, `builder` | Mandatory SRP + Hexagonal Architecture standard for any code written or reviewed; gates `safe-build`'s definition-of-done and informs `codebase-recon`'s blast-radius classification |

`deep-specialist` preloads only `agent-handoff-protocol` and `codebase-recon` — the subset it needs
for isolated recon/review work; it never sees `plan-mode-protocol`. `builder` preloads only
`safe-build` and `coding-architecture-standards` — the subset it needs to execute an already-written
sub-plan; it never sees `plan-mode-protocol` or `agent-handoff-protocol` either, since it isn't
dispatched by anything.

## Plan mode rules (mandatory, non-optional, embedded in `engineer`)

Whenever plan mode is invoked, `engineer`'s embedded `plan-mode-protocol` governs the output. Four
rules, plus a mandatory human checkpoint, always in force:

1. **Modular sub-division** — a complex plan is never written into one information file. Split by
   module/phase into an index file plus one file per sub-plan; each sub-plan file is
   self-contained but cross-referenced so the full set reads coherently together.
2. **Explicit priority** — every plan and sub-plan carries a visible `P0` (blocking) / `P1`
   (required) / `P2` (enhancement) marker. Priority is never left to be inferred from order.
3. **Strict sequencing** — sub-plans execute in one strict, declared order, recorded in a single
   lightweight notation file:
   ```
   Project #1
   Plan 1. -> Sub 1.1  do this  -> Sub 1.2  do that  -> Sub 1.3  do that ...
   ```
4. **Incremental delivery** — work executes one sub-plan at a time. Only that sub-plan's file is
   loaded into context (never the whole plan at once), and permission to proceed is required
   before every next sub-plan. No phase runs unattended off a prior blanket approval unless the
   user has explicitly scoped a standing approval to that run.
5. **Checkpoint 0 — human approval before any execution.** The moment the plan is saved to disk,
   stop. Planning it is not authorization to execute it — not even for `engineer` to continue in
   the same session. `engineer` presents the saved plan and waits for an explicit go-ahead before
   `Sub 1.1` starts. Once cleared, the user picks the execution path: `engineer` keeps going in
   place, or the user separately invokes `builder` — never both.

Full detail: `skills/plan-mode-protocol/SKILL.md`.

## How it works

**Planning and (optionally) building, in one `engineer` session:** `engineer` classifies each
request, recons existing code with `codebase-recon` when relevant, and — if the scope calls for a
plan — autoplans via its embedded `plan-mode-protocol`, saving the result to disk and stopping for
Checkpoint 0 approval. Once approved, `engineer` can keep going and execute the plan itself with
`safe-build`, gated on `coding-architecture-standards`. For isolated work that would otherwise eat
most of `engineer`'s own context (a wide recon, an independent review, exhaustive research),
`engineer` dispatches `deep-specialist` instead, one at a time, packet in and packet out per
`agent-handoff-protocol`:

```yaml
handoff:
  objective: <one sentence>
  from: engineer | deep-specialist
  to: deep-specialist | engineer | user
  status: dispatched | complete | blocked | needs_clarification | rejected
  priority: P0 | P1 | P2
  sub_plan_ref: <e.g. "Plan 1 / Sub 1.2", or "n/a">
  inputs: [<file path or inline reference>]
  constraints: [<explicit "do NOT" boundary>]
  produced_artifacts: [{path, description}]
  definition_of_done: <concrete, checkable criterion>
  notes: <deviations, open questions, escalation target if blocked>
```

`deep-specialist` never talks to the user or opens a further subagent — `engineer` remains the
single point of contact throughout its own session.

**Executing a saved plan cheaply, separately:** instead of (or after) letting `engineer` execute
its own plan, the user can invoke `@solo:builder` directly, pointing it at the saved plan on disk.
`builder` reads the index + the relevant `Sub n.k` file, executes exactly that one step with
`safe-build` and `coding-architecture-standards`, reports, and stops — the next `Sub n.k` only runs
on the user's next explicit invocation of `builder`. `builder` never talks to `engineer` and never
edits the plan itself; if the plan looks missing or stale, it says so and points back to
`@solo:engineer` rather than improvising.

`engineer` and `builder` are never active on the same plan at the same time — the human decides
which one to run for a given step, every time.

## Install & invoke

Distributed as a `.plugin` file. Install through Cowork's plugin installer, or unzip manually into
a project's `.claude/` directory if using Claude Code directly. Agents live under `agents/`;
skills live under `skills/<name>/SKILL.md`. Discovery is automatic.

Once installed, call either entry point directly: **`@solo:engineer`** to classify/recon/plan/build
or delegate to `deep-specialist`, or **`@solo:builder`** to execute an already-approved plan on a
cheaper model.

## Customizing

- `engineer`'s, `deep-specialist`'s, and `builder`'s `tools:` lines each restrict what that role
  can touch — adjust per your workflow.
- Adding a skill to an agent's `skills:` list costs tokens on every invocation (preloaded at
  startup); keep each set to what's actually needed every time.
- `builder`'s `model:` is the token-optimization lever — swap it for a different cheap/fast model
  if you'd rather not use haiku, but keep it a lighter model than `engineer`'s, since the whole
  point is running well-scoped, already-approved sub-plans cheaply.
- `plan-mode-protocol` is intentionally embedded only on `engineer` and never optional there — it
  is always in force the moment plan mode is used, regardless of task domain or size. It is
  deliberately absent from `builder`, which only ever executes what's already been planned.
