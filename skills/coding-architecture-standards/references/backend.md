# Backend Reference — Hexagonal Architecture & SRP

Read this when writing, reviewing, or refactoring any backend/server-side code. Builds on the
shared rules in `SKILL.md` — read that first if you haven't.

## Layer layout

```
src/
  domain/         entities, value objects, domain rules — pure, no framework imports
  application/    use cases / services; orchestrate domain; DEFINE ports (interfaces)
  adapters/
    inbound/      HTTP controllers, CLI commands, queue/event consumers, GraphQL resolvers
    outbound/     DB repositories, external API clients, file storage, email/SMS senders
  composition/    wiring only — this is the ONE place concrete adapters get constructed
                  and injected into use cases (a.k.a. composition root, main.ts, app factory)
```

Naming isn't mandatory (`domain/` vs `core/`, `application/` vs `usecases/` — match the
codebase's existing convention if one exists), but the four responsibilities must stay separate
files/modules. Never collapse "domain rule" and "route handler" into the same file because it's
convenient for a small feature — the layering pays for itself the moment the feature grows.

## Ports belong to the core, not the adapter

The application layer declares the interface it needs; the adapter directory implements it.

```
// application/ports/user-repository.ts   <- port, owned by application layer
interface UserRepository {
  findById(id: string): Promise<User | null>
  save(user: User): Promise<void>
}

// application/use-cases/register-user.ts  <- depends only on the port
class RegisterUser {
  constructor(private users: UserRepository) {}
  async execute(input: RegisterInput): Promise<User> { /* pure orchestration + domain rules */ }
}

// adapters/outbound/postgres-user-repository.ts  <- implements the port, imports the ORM
class PostgresUserRepository implements UserRepository { /* ORM-specific code lives ONLY here */ }
```

`RegisterUser` never imports `PostgresUserRepository` directly. The composition root decides
which implementation to hand it. This is what makes the use case swappable and unit-testable.

## Inbound adapters stay thin

A controller/handler's job is: parse the request, call one use case, map the result to a
response. If a controller contains a business rule (a discount calculation, a validation rule
beyond "is this field present/well-formed"), that rule belongs in `domain/` or `application/`,
not in the HTTP layer. Test: could this controller be replaced by a CLI command or a queue
consumer calling the same use case, unchanged? If not, logic leaked into the adapter.

## Outbound adapters implement, never leak

A repository/client adapter's job is: satisfy its port's interface using a specific technology.
It should not contain business rules, and its method signatures should speak the domain's
language (`findActiveSubscriptions(userId)`), not the underlying tech's language
(`runQuery(sql: string)`). If callers need to know it's Postgres vs. Mongo vs. an HTTP API, the
abstraction has a hole in it.

## Dependency Injection — composition root only

Concrete adapters get constructed and wired to use cases in exactly one place per application
(the composition root — `main.ts`, `app.py`'s bootstrap, a DI container's config module). A use
case or domain function must never contain `new SomeRepository()`, `require('pg')`, or a direct
framework import. If you find yourself instantiating an adapter inside application/domain code to
"save time," that's the leak this rule exists to prevent — wire it at the root instead, even for
a one-off script.

## SRP at backend layer level

- **domain/**: one entity or value object's rules per file. A `User` entity validating its own
  invariants (email format, password strength) is fine; a `User` entity that also knows how to
  hash a password using a specific library is not (hashing is an adapter concern — inject a
  `PasswordHasher` port).
- **application/**: one use case per file/class (`RegisterUser`, `DeactivateUser`), not a
  `UserService` god-class with fifteen unrelated methods. If a "service" file keeps growing
  because unrelated features keep landing in it, split by use case.
- **adapters/**: one adapter implements one port. Don't let a single repository class quietly grow
  into "the class that also sends emails because it was convenient" — that's two responsibilities
  and two reasons to change.

## Common violations to catch in review

| Symptom | Likely cause |
|---|---|
| A domain entity imports an ORM decorator (`@Entity`, `@Column`) | Domain and persistence model merged — split into a domain entity + a separate persistence mapper/DTO |
| A use case test needs a real database to pass | Port not actually being used — use case is calling a concrete adapter directly |
| Business logic duplicated in the HTTP controller and the CLI command | Logic lives in the adapter instead of the use case; move it up, call from both |
| A "utils" or "helpers" file with unrelated functions accumulating over time | No real home for these — decide if each belongs in domain, application, or a named adapter, and place it there instead |
| Adding a field requires touching the controller, the service, the repository, and the domain entity for what should be one change | Sign that layering itself is likely fine, but worth checking these aren't tightly coupled sibling copies of the same shape that should be one DTO passed through |

## Testability check

The fastest signal a backend module is correctly hexagonal: the application layer's tests run
with an in-memory fake for every port (`InMemoryUserRepository` implementing `UserRepository`)
and complete in milliseconds, with no DB container, no HTTP server, no network call. If a "unit
test" for a use case needs `docker-compose up` first, the boundary isn't real yet — fix the
architecture, not the test setup.
