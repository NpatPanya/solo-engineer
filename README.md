# solo

A **1 agent + multi-skill + 1 subagent** engineering plugin, adapted down from a 9-role reference
design. `engineer` — invoked directly as **`@solo:engineer`** — is the single entry point and the
only role that talks to the user; it carries 5 preloaded skills and can dispatch exactly one
subagent, `deep-specialist`, for isolated deep-dive work.

## Roster

| Role | Agent name | Model | Effort | Responsibility |
|---|---|---|---|---|
| Engineer (entry point) | `engineer` (`@solo:engineer`) | sonnet | medium | Classifies requests, plans, executes directly or delegates, reconciles results, reports to the user |
| Specialist (subagent) | `deep-specialist` | sonnet | high | Isolated deep-dive work — large trace/recon, independent review, exhaustive research — dispatched one at a time, never talks to the user |

## Skills (all preloaded on `engineer`)

| Skill | Purpose |
|---|---|
| `plan-mode-protocol` | Mandatory 4-rule ruleset applied whenever plan mode is used (see below) |
| `agent-handoff-protocol` | Structured packet format for every dispatch to/from `deep-specialist` |
| `codebase-recon` | Tracing existing code and blast-radius assessment |
| `safe-build` | In-place implementation + mechanical refactor + test-after methodology |
| `deep-specialist-brief` | Decision rule for when to delegate, and how to brief the dispatch |

`deep-specialist` itself preloads only `agent-handoff-protocol` and `codebase-recon` — the subset
it needs for isolated recon/review work; it never sees `plan-mode-protocol` because it never
produces a plan or talks to the user.

## Plan mode rules (mandatory, non-optional)

Whenever plan mode is invoked, `plan-mode-protocol` governs the output. Four rules, always in
force:

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
   loaded into context (never the whole plan at once), and `engineer` stops to get explicit user
   permission before starting the next sub-plan. No phase runs unattended off a prior blanket
   approval unless the user has explicitly scoped a standing approval to that run.

Full detail: `skills/plan-mode-protocol/SKILL.md`.

## How it works

`engineer` classifies each request. Small, well-understood work it executes directly, using
`codebase-recon` and `safe-build` in place. Work that would consume most of its own working
context, or that benefits from an unbiased second look, it delegates to `deep-specialist` per
`deep-specialist-brief`'s criteria — one dispatch at a time, packet in, packet out, using the
shape in `agent-handoff-protocol`:

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

`deep-specialist` never talks to the user and never opens a further subagent — `engineer` remains
the single point of contact throughout.

## Install & invoke

Distributed as a `.plugin` file. Install through Cowork's plugin installer, or unzip manually into
a project's `.claude/` directory if using Claude Code directly. Agents live under `agents/`;
skills live under `skills/<name>/SKILL.md`. Discovery is automatic.

Once installed, call the agent directly with **`@solo:engineer`** followed by your request.

## Customizing

- `engineer`'s `tools:` line and `deep-specialist`'s `tools:` line each restrict what that role
  can touch — adjust per your workflow.
- Adding a skill to `engineer`'s `skills:` list costs tokens on every dispatch (preloaded at
  startup); keep the set to what's actually needed every time.
- `plan-mode-protocol` is intentionally the one skill that is never conditional — it is preloaded
  and always in force the moment plan mode is used, regardless of task domain or size.
