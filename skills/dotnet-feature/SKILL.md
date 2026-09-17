---
name: dotnet-feature
description: Adds a new feature (command or query) to a .NET 10 API following our CQRS architecture without MediatR. Use this skill whenever a new endpoint, command, query or handler has to be added in a project with the layers Domain, Application, Infrastructure and Api - for example "add an endpoint to complete a todo" or "create a query to fetch orders per customer". Do not use it for setting up a new project or for changes that only modify existing code.
---

# Adding a new .NET feature

Follows the architecture from `@.standards/dotnet/ARCHITECTURE.md`. Read that document
if you are in doubt about a choice; this skill is the execution of the checklist from
chapter 6, not a replacement for it.

`<Product>` in the paths below is the product name of the solution, for example
`TodoApp` in `src/TodoApp.Domain/`. See `@.standards/dotnet/solution-layout.md`.

## Before you start

1. Read `.standards/dotnet/ARCHITECTURE.md` (in particular chapter 5, conventions) and
   `.standards/dotnet/solution-layout.md` (project names and paths).
2. Read the project's `CLAUDE.md` for repo-specific deviations.
3. Determine: is this a **command** (changes state) or a **query** (reads state)?
   Both in one handler is not an option - split it.
4. Look at an existing feature in the same repo and follow that style. The conventions
   below apply, but the existing project wins when in doubt.

## Steps

Work through them in this order. Skip steps that do not apply, but do say that you
are skipping them.

### 1. Domain

Add the entity or the behavior in `src/<Product>.Domain/<Feature>/`.

- Private setters; state changes only through methods with a meaningful name.
- No public parameterless constructor (a private one is fine, for EF Core).
- No references to other layers. Domain depends on nothing.

### 2. Repository interface

Add the required method to `I<Entity>Repository` in `src/<Product>.Application/<Feature>/`.

- Every method gets a `CancellationToken`.
- If a command has to change an existing entity, a separate method is needed
  **without** `AsNoTracking()` - otherwise changes are not saved.

### 3. Command or query + handler

In `src/<Product>.Application/<Feature>/`:

- Naming: `{Verb}{Entity}Command` / `Get{Entity}By{Criterion}Query`, the handler
  is `{Name}Handler`.
- Everything `sealed`, properties with `init`.
- One public method: `HandleAsync(request, CancellationToken)`.
- Constructor injection of the repository, no service locator.
- Command: calls `SaveChangesAsync`. Query: never, and uses `AsNoTracking()`.

### 4. DTO

Reuse `{Entity}Dto` if it already exists, otherwise create it in the same folder.
Never return a domain entity from the API.

### 5. Infrastructure

Implement the repository method in `src/<Product>.Infrastructure/<Feature>/`.

### 6. EF configuration and migration

Only if the data model changes:

```bash
dotnet ef migrations add <Name> --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
```

Leave running `database update` to the user.

### 7. Register the handler

In `src/<Product>.Api/Program.cs`: `builder.Services.AddScoped<{Name}Handler>();`

### 8. Add the endpoint

Also in `Program.cs`. The endpoint contains **no logic**: request in, call the handler,
HTTP result back.

- Command: `Results.Created(...)` or `Results.NoContent()`.
- Query: `Results.Ok(...)`, or `Results.NotFound()` if the result is `null`.
- Pass the `CancellationToken` along.

### 9. Unit test

See `@.standards/dotnet/testing.md`. Mock the repository with Moq, call `HandleAsync`
directly, check the result **and** verify the interactions (for a command: something
was saved; for a query: nothing was saved).

## Wrapping up

- Run `dotnet build` and `dotnet test`.
- Walk through the conventions table in chapter 5 of ARCHITECTURE.md.
- If there are architecture tests, they must be green - those guard the layer rules.
- Report which files you added or changed, and which steps you skipped and why.
