---
name: coding-architecture-standards
description: Mandatory architecture and design standards for writing or reviewing any application code, frontend or backend. Enforces the Single Responsibility Principle (SRP) and Hexagonal Architecture (ports & adapters) so code stays maintainable, testable, and framework-agnostic at its core. Use this whenever engineer or builder is about to write new code, add a module/component/class/function, wire up a new dependency, design an API or data layer, or refactor existing code — not just when the user says "architecture." If a task involves creating or touching source files in any language or framework, this skill applies. Language-agnostic core; see references/backend.md and references/frontend.md for concrete layer layouts and worked examples.
---

# Coding Architecture Standards

Two non-negotiable rules govern every unit of code this team writes, on either side of the stack:

1. **Single Responsibility Principle (SRP)** — every module, class, function, or component has
   exactly one reason to change.
2. **Hexagonal Architecture (ports & adapters)** — a framework-free domain/application core,
   surrounded by explicit interfaces ("ports") that the core defines, implemented by
   swappable "adapters" on the outside. Dependencies point inward, always.

These are not style preferences. They are the definition-of-done gate referenced by `safe-build`
(implementation step) and the blast-radius classifier in `codebase-recon`. Code that violates
either rule is not "done," even if it passes its own tests.

## 1. Single Responsibility Principle

**Definition**: a unit of code should have one reason to change — one actor, one concern, one job.

**Smells that mean SRP is broken** (check before calling anything done):

- A file imports both a persistence/HTTP library **and** a UI/presentation library.
- A function or method name contains "and" (`validateAndSave`, `fetchAndRender`) — it's doing two
  jobs; split it.
- A class or component touches more than one of: data access, business rule, orchestration,
  presentation.
- Adding a feature to layer A requires editing a file whose primary job is layer B.
- A test for "the business rule" also has to mock a database, an HTTP client, and a router.

**Fix pattern**: extract until each unit answers "what is this responsible for" in one clause with
no "and." One responsibility per file/class/function/component; compose them, don't merge them.

## 2. Hexagonal Architecture (Ports & Adapters)

**Definition**: the domain and application logic form a core with zero framework imports. The
core defines the interfaces ("ports") it needs from the outside world. Everything
framework-specific — HTTP frameworks, ORMs, UI libraries, queue clients, third-party SDKs — is an
"adapter" that implements a port. Dependencies always point inward: adapters depend on the core;
the core never imports an adapter or a framework.

```
            ┌─────────────────────────────┐
 inbound    │        Adapters (outer)      │   outbound
 adapters → │  ┌───────────────────────┐   │ → adapters
 (HTTP,     │  │   Application layer    │   │  (DB repo,
  CLI,      │  │  (use cases, orchestr- │   │   external
  UI event) │  │   ation, defines ports)│   │   API client,
            │  │  ┌──────────────────┐  │   │   storage)
            │  │  │   Domain core     │  │   │
            │  │  │ (entities, rules, │  │   │
            │  │  │  no framework)    │  │   │
            │  │  └──────────────────┘  │   │
            │  └───────────────────────┘   │
            └─────────────────────────────┘
```

**Ports are owned by the core, not by the adapter.** The application layer defines the interface
it needs (e.g. `UserRepository.findById(id)`); an adapter (a Postgres repo, an in-memory fake, a
REST client) implements that interface. The core never reaches out and imports a concrete
adapter — it receives one via dependency injection at the composition root.

**Test this directly**: can the application/domain layer be unit-tested with a hand-written fake
adapter and zero framework, database, or network spin-up? If writing that test requires spinning
up a framework or a real dependency, a boundary is leaking and the code isn't hexagonal yet.

This applies on both sides of the stack — a backend service and a frontend app are both "adapters
around a core" from this skill's point of view:

- **Backend**: HTTP controllers, DB repositories, message consumers, and third-party clients are
  all adapters. The domain/use-case layer between them has no `import express`, no ORM decorator,
  no HTTP status code.
- **Frontend**: UI components, API-client implementations, and browser storage are adapters. The
  domain/state logic behind them (validation rules, business calculations, orchestration) is
  plain, framework-agnostic code that doesn't import the UI framework.

See `references/backend.md` and `references/frontend.md` for concrete layer layouts, naming
conventions, and worked examples — this file stays the shared, framework-agnostic contract both
build on.

## Pre-done checklist

Run this before reporting any code-writing step complete (feeds the Rule 4 permission-gate report
in `plan-mode-protocol`):

- [ ] Domain/application code has no framework or infrastructure imports.
- [ ] Every new/edited unit has one responsibility, statable in one clause with no "and."
- [ ] Dependencies point inward only — an adapter may depend on the core; the core never depends
      on an adapter.
- [ ] A new capability that talks to the outside world does so through an existing port, or a
      newly defined one — never an ad-hoc call to a concrete library from inside the core.
- [ ] The core is unit-testable with a fake adapter, no real framework/DB/network required.

If any box is unchecked, that's a gap to flag per the "flag a gap, don't invent" rule in
`agent-handoff-protocol` — not something to silently patch over to close out the step.
