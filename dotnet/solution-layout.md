# Solution layout and project naming: .NET

Goes with [`ARCHITECTURE.md`](ARCHITECTURE.md). That describes the layers and what the
code inside them looks like; this document describes how the solution is laid out and
what projects, assemblies and namespaces are called.

> Assembly name, root namespace and folder name are always identical.

That is the most important rule. It makes navigating predictable and stops namespaces
from drifting away from the place where the files are.

---

## 1. The pattern

```
<Product>.<Layer>
```

- `<Product>` is the name of the application or service, for example `TodoApp` or
  `Billing`. No company prefix (`Wheel7.`): it adds nothing and makes every namespace
  longer.
- `<Layer>` is one of the layers from `ARCHITECTURE.md`: `Domain`, `Application`,
  `Infrastructure`, `Api`.
- Layers are singular: `.Domain`, not `.Domains`. Plural only for something that really
  is a collection (`.Migrations`).

Why not bare names like `Domain` or `Application`? They collide as soon as two products
reference each other or share packages, `Application` rubs against framework types, and
an architecture test that filters on namespace `"Application"` hits more than you mean.
With `TodoApp.Application` such a filter is exact.

### Multiple bounded contexts

If a single solution holds more than one bounded context, the context comes before the
layer: `Shop.Ordering.Domain`, `Shop.Catalog.Domain`. Only do this when there really are
multiple contexts; one product with one context gets plain `<Product>.<Layer>`.

---

## 2. Standard layout

```
TodoApp.sln
├── src/
│   ├── TodoApp.Domain/
│   ├── TodoApp.Application/
│   ├── TodoApp.Infrastructure/
│   └── TodoApp.Api/
└── tests/
    ├── TodoApp.Application.UnitTests/
    ├── TodoApp.Api.IntegrationTests/
    └── TodoApp.ArchitectureTests/
```

The entry point is named after what it is: `.Api`, `.Web`, `.Blazor`, `.Worker`. A
solution with an API and a worker therefore has two entry points side by side.

### Test projects

Test projects take the name of the project they test, plus a suffix that says what kind
of test it is:

| Suffix | Test | Example |
|---|---|---|
| `.UnitTests` | one class in isolation, dependencies mocked | `TodoApp.Application.UnitTests` |
| `.IntegrationTests` | the real chain, with host and database | `TodoApp.Api.IntegrationTests` |
| `.ArchitectureTests` | layer rules and conventions across the whole solution | `TodoApp.ArchitectureTests` |

A suffix and not a prefix, so that a test project sits next to the project it tests in
the Solution Explorer. A bare `.Tests` is not enough once you have both unit and
integration tests, and that moment always comes.

The architecture tests do not test one layer but the solution, so there is no layer in
that name.

---

## 3. Optional projects

Only create these when the situation calls for it. A project that starts out empty "for
later" becomes a dumping ground.

### `.Contracts`: shared types with a .NET client

The standard layout is API-only. If a .NET client is added that calls the same API
(Blazor WebAssembly, a console tool, another service), both sides need the same request
and response types. That client must not reference `.Application` and `.Domain`: that
puts entities and repositories in the browser.

That is what `.Contracts` is for. The name says what belongs in it, and with that also
what does not:

- **Yes**: request and response types, the enums they carry, constants that travel with
  them (routes, header names).
- **No**: services, extension methods, helpers, interfaces from the application layer.

`.Contracts` has no project references at all. The client references `.Contracts` and
nothing else.

**When to and when not to.** A React frontend is no reason for `.Contracts`: it speaks
JSON and reads the contract from the OpenAPI specification, see
[`../general/api-contracts.md`](../general/api-contracts.md). You introduce `.Contracts`
only when a second .NET project calls the API, not earlier.

**Consequence for the CQRS pattern.** `ARCHITECTURE.md` binds the command from
`.Application` directly as the request body in the endpoint. Once `.Contracts` exists
that is no longer possible, because the client must not know `.Application`. So:

1. Request types go in `.Contracts` (`CreateTodoRequest`).
2. Response types move from `.Application` to `.Contracts` (`TodoDto` or
   `TodoResponse`).
3. The endpoint maps request to command, in one line. That is still "no logic": it is
   the only place where the two touch.
4. `.Application` references `.Contracts` (allowed, because `.Contracts` has no
   dependencies). `.Domain` does not reference `.Contracts`.

Add an architecture test as well: `.Contracts` has no dependency on another layer.

### `.Persistence` and `.Migrations`

Migrations belong with persistence. As long as only the application runs them, they live
in `.Infrastructure`, as `ARCHITECTURE.md` describes.

If `.Infrastructure` grows to the point where EF Core, HTTP clients and messaging get in
each other's way, split off `.Persistence` for everything that has to do with the
database.

If you want to run migrations separately from the application as well (in a pipeline, by
an administrator), give them their own executable project `.Migrations`. `.Persistence`
then stays a library.

Name the project after the task, not after the tool: `.Migrations`, not `.DbUp`,
`.FluentMigrator` or `.EfMigrations`. See also the rule about technology names below.

---

## 4. Names we do not use

| Name | Why not | What instead |
|---|---|---|
| `.Core` | Means something different in every codebase: domain, application layer or plumbing. We already have a name per layer. | `.Domain` or `.Application`, depending on what you mean |
| `.Common`, `.Shared`, `.Utils`, `.Helpers` | Mean nothing, so everything fits in them and the project fills up. | Name it after the content: `.Abstractions`, `.Logging`, `.Validation` |
| `.Extensions` | Extension methods are a language feature, not a category. This is how `.Utils` gets reinvented. | Put them with what they extend: mapping with the mapping, registration in the project that registers, display extensions in the UI |
| `.BLL`, `.DAL` | Date the codebase and say nothing to anyone who did not learn them in 2010. | `.Application`, `.Infrastructure` |
| `.EntityFramework`, `.SqlServer`, `.RabbitMq` | Technology in a project name means renaming projects when you replace the tool. | Technology is a folder inside `.Infrastructure`: `Infrastructure/Data/`, `Infrastructure/Messaging/` |

An extension method on a framework type that fits nowhere is usually a sign that the
logic belongs somewhere more concrete.

---

## 5. Checklist: new solution

1. Pick the product name. Short, without a company prefix, and the same as the name of
   the repository if you can.
2. Create the four layers as `<Product>.<Layer>` in `src/`. See chapter 3 of
   `ARCHITECTURE.md` for the commands.
3. Check that assembly name, root namespace and folder name are identical. `dotnet new`
   with `-n <Product>.<Layer> -o src/<Product>.<Layer>` takes care of that.
4. Create test projects with the right suffix in `tests/`.
5. Do not add an optional project until the situation from chapter 3 arises.
