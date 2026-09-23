---
name: dotnet-feature
description: Adds a new feature (command or query) to a .NET 10 API following our CQRS architecture without MediatR. Use this skill whenever a new endpoint, command, query or handler has to be added in a project with the layers Domain, Application, Infrastructure and Api - for example "add an endpoint to complete a todo" or "create a query to fetch orders per customer". Do not use it for setting up a new project or for changes that only modify existing code.
---

# Adding a new .NET feature

Follows the architecture from `@.standards/dotnet/ARCHITECTURE.md`. Read that document
if you are in doubt about a choice; this skill is the execution of the checklist from
chapter 6, not a replacement for it.

> The `.standards/` paths below assume this project has the standards as a git submodule.
> Without it, the same documents are in the repository this skill was installed from.

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

Every property is `required`, on the DTO and on a command bound from a request body. See
chapter 4.3 of `@.standards/dotnet/ARCHITECTURE.md` for why: without it the OpenAPI
document makes every response field optional, and a missing number in a request silently
arrives as zero.

### 5. Infrastructure

Implement the repository method in `src/<Product>.Infrastructure/<Feature>/`.

### 6. EF configuration and migration

Only if the data model changes:

```bash
dotnet ef migrations add <Name> --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
```

Leave applying it to the user: restarting `aspire run` does that, through `.DbMigrator`.

### 7. Register the handler

In `src/<Product>.Api/Program.cs`: `builder.Services.AddScoped<{Name}Handler>();`

### 8. Add the endpoint

Also in `Program.cs`. The endpoint contains **no logic**: request in, call the handler,
HTTP result back.

- Create: `Results.Created(...)` with the DTO.
- Update: the id comes from the route, `command with { Id = id }`, see chapter 4.9 of
  `@.standards/dotnet/ARCHITECTURE.md`. `Results.Ok(...)` with the DTO, or
  `Results.NotFound()` when the handler returns `null`.
- Delete: `Results.NoContent()`, or `Results.NotFound()` when there was nothing to delete.
- Query: `Results.Ok(...)`, or `Results.NotFound()` if the result is `null`.
- A conflict, such as a name that must be unique, is a `ConflictException` thrown by the
  handler, never a status code chosen in the endpoint. See chapter 4.10.
- Pass the `CancellationToken` along.

Check before the first endpoint of a project that `BadRequestExceptionHandler` and
`ConflictExceptionHandler` are registered, and that JSON uses strict numbers and enums by
name. Chapter 4.10 and 4.11. Without the first, a missing field is a `500` in Development.

### 9. Regenerate the contract and the client

Build the API, which rewrites the committed OpenAPI document, and regenerate the frontend
client from it with the project's script (see its `CLAUDE.md`). Commit both with the code.
CI fails when either is out of date, and a frontend that compiles against an old client is
a frontend that breaks at runtime.

### 10. Tests

See `@.standards/dotnet/testing.md`. Mock the repository with Moq, call `HandleAsync`
directly, check the result **and** verify the interactions (for a command: something
was saved; for a query: nothing was saved).

Add an integration test for every endpoint, and one that proves another user's record is a
`404` for reading, changing and deleting, and is left untouched.

## Wrapping up

- Run `dotnet build` and `dotnet test`.
- Walk through the conventions table in chapter 5 of ARCHITECTURE.md.
- If there are architecture tests, they must be green - those guard the layer rules.
- Fixing a bug? Confirm the test fails without the fix before you call it done. A test
  that passes either way is testing something else.
- Report which files you added or changed, and which steps you skipped and why.
- Run the automated review over the diff before you hand it over, fix what it is right
  about, and report what it found and what you did with it. It is marking your own work, so
  it catches slips rather than blind spots; the pass that counts is the one the person who
  merges runs. See `@.standards/general/testing.md` chapter 3.
- Report the test evidence in the format from `@.standards/general/testing.md`: the
  command you ran, the summary line it actually printed, which tests you added and what
  they assert, what this change touches that no test covers, and anything you could not
  verify here. Those last two are not allowed to be empty, and a summary of a run that
  did not happen is worse than saying you could not run it.

