# Frontend Reference — Hexagonal Architecture & SRP

Read this when writing, reviewing, or refactoring any frontend/UI code. Builds on the shared
rules in `SKILL.md` — read that first if you haven't. The backend layering in `backend.md` maps
onto the frontend almost directly; the main difference is that the "framework" being kept out of
the core is the UI library itself (React, Vue, etc.), not just the data layer.

The most common frontend anti-pattern this guards against: business logic, validation, and
orchestration living *inside components*, so the UI framework becomes the de facto core of the
application instead of a replaceable shell around one.

## Layer layout

```
src/
  domain/          pure business rules, validation, calculations — no UI framework import,
                    no fetch/axios import, just plain functions/classes and types
  application/      use cases / orchestration hooks; DEFINE ports (interfaces) for data access
                    and side effects; framework-light (may use the framework's state primitives,
                    but contains no markup and no direct fetch/storage calls)
  adapters/
    ui/            components — presentation only, receive data + callbacks as props
    api/           concrete API clients implementing application-defined ports
    storage/       localStorage/IndexedDB/cookie implementations of storage ports
  composition/      where a concrete adapter gets wired to a hook/use case — app root,
                    provider setup, or a DI-ish factory/context
```

Small apps don't need every folder from day one, but the *separation of concerns* still applies
even in a single-file prototype: don't let one file both call `fetch` and return JSX for the
result of unrelated business logic.

## Ports belong to the core, not the component

The application layer declares the interface it needs from the outside world (usually "how do I
get/save this data"); a concrete adapter implements it. Components never call `fetch`/`axios`
directly for anything beyond the most trivial one-off page — they call a hook that depends on a
port.

```
// application/ports/user-repository.ts        <- port
interface UserRepository {
  getById(id: string): Promise<User>
  update(user: User): Promise<void>
}

// application/use-cases/use-update-profile.ts  <- depends only on the port
function useUpdateProfile(users: UserRepository) {
  // orchestration + validation calls into domain/, no JSX, no fetch()
}

// adapters/api/rest-user-repository.ts         <- implements the port
class RestUserRepository implements UserRepository { /* fetch()/axios calls live ONLY here */ }

// adapters/ui/ProfileForm.tsx                  <- presentation only
function ProfileForm({ user, onSave }: Props) { /* markup + local UI state only */ }
```

`ProfileForm` receives `user` and `onSave` as props; it doesn't know or care whether the data
came from REST, GraphQL, or a mock. Swapping `RestUserRepository` for a test double doesn't touch
the component at all.

## Components stay on one side of the container/presentational split

- **Orchestrating** components/hooks: fetch data, call use cases, manage what-to-show decisions.
  No markup beyond thin wiring.
- **Presentational** components: receive data + callbacks via props, render markup, own local UI
  state (an input's draft value, a dropdown's open/closed state) — but not business rules.

A component that both fetches data *and* contains the validation rule for that data is doing two
jobs. Split: a hook/container owns the fetch + orchestration, a plain component owns rendering,
and the validation rule itself lives in `domain/` where both the form and, say, a bulk-import
flow can reuse it without duplicating the rule.

## SRP at component/hook level

- One component = one rendering responsibility. A `UserCard` that also contains checkout logic is
  two components wearing a trenchcoat.
- One custom hook = one piece of orchestration. `useUserProfile` fetching a user is fine;
  `useUserProfile` also handling unrelated notification-badge logic is not — split the hook.
- Shared business rules (a discount calc, a password-strength rule, a date-range validator) live
  in `domain/` as plain functions, imported by whichever component/hook needs them — never
  copy-pasted between components because "it's just three lines."

## Common violations to catch in review

| Symptom | Likely cause |
|---|---|
| A component file has both `fetch(...)` calls and JSX for unrelated concerns | No adapter/port boundary — pull the fetch into an API adapter behind a port |
| The same validation logic appears in two different form components | Rule lives in a component instead of `domain/` — extract and share |
| A "smart" component keeps growing every time a new feature touches this screen | Orchestration and presentation merged — split container from presentational component |
| Unit-testing a component requires mocking a UI framework's router, a real HTTP client, and browser storage all at once | Business/orchestration logic is embedded in the component instead of a separately-testable hook/use case |
| Business rule changes require editing multiple component files across the app | Rule wasn't centralized in `domain/` — consolidate to one source of truth |

## Testability check

The application layer (hooks/use cases) and domain logic should be unit-testable with plain
function calls or a lightweight hook-testing utility and a hand-written fake adapter — no real
network, no real browser storage, no rendering a full component tree required. If testing "what
happens when the user submits invalid data" requires mounting the whole page and mocking a
network layer, the validation rule isn't actually isolated in `domain/` yet.

## Framework-primitive note

Using the UI framework's own state/effect primitives (`useState`, `useReducer`, signals, etc.)
inside the application layer is fine — that's orchestration, not presentation, and these tools
are how orchestration is expressed in most modern frontend frameworks. The line this skill draws
is about *markup and framework-specific rendering APIs* staying out of `domain/` and
`application/`, not about banning framework primitives everywhere outside `adapters/ui/`.
