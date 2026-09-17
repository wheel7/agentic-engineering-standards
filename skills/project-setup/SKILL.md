---
name: project-setup
description: Sets up a new repository against these engineering standards, including the solution layout, database, container setup and CI/CD. Use when someone starts a new project, creates a new solution, or asks how to bootstrap a repository that follows the team standards - for example "set up a new API for billing" or "start a new React project". Do not use for adding a feature to an existing project; use dotnet-feature or react-component for that.
---

# Setting up a new project

This skill turns the standards in `.standards/` into a working repository. It is the one
place where the decisions that cannot be derived from code get made and recorded.

## Ask first, scaffold second

Six things cannot be inferred from an empty repository, and all six are expensive to
change later. Ask the developer, one at a time, and do not guess.

Ask the questions in this order, because later answers depend on earlier ones.

### 1. Product name

Becomes the prefix of every project: `Billing.Domain`, `Billing.Api`. Short, no company
prefix, ideally the same as the repository name.

Getting this wrong means renaming every project, namespace and folder later. See
`@.standards/dotnet/solution-layout.md`.

### 2. Domain language

Which language do the people who pay for this software use for their own concepts?

This is not the same question as which language the code is written in. Technical
vocabulary is always English. Domain concepts follow the business, so a Dutch insurer
gets `Polis` and `CreatePolisCommand`. See `@.standards/general/language.md`.

Ask it explicitly. A team that never decides ends up with both, which is the one outcome
the standard rules out.

### 3. Stack and entry points

.NET API, React frontend, or both. More than one entry point, such as an API plus a
worker? Each entry point is its own project and its own container image.

### 4. Database

PostgreSQL unless there is a reason. If there is a reason, record it as a deviation with
the reason attached, otherwise someone will "fix" it later.

Ask which major version, because it has to match across the compose file, CI and
production.

### 5. Hosting

Where does this run, and who deploys it? The answer drives the deploy step of the CD
workflow. If it is not decided yet, say so in the project `CLAUDE.md` rather than
inventing something.

### 6. Documentation language

The team's call, but it has to be one language. Record it.

## Then scaffold

Work through these in order. Skip what does not apply, but say which steps you skipped
and why.

1. **Solution and projects** per `@.standards/dotnet/solution-layout.md`. Assembly name,
   root namespace and folder name identical.
2. **Test projects** with the right suffixes: `.UnitTests`, `.IntegrationTests`,
   `.ArchitectureTests`.
3. **Architecture** per `@.standards/dotnet/ARCHITECTURE.md`: the four layers, the
   project references pointing inwards, one vertical slice as an example.
4. **Database** per `@.standards/ops/database.md`: Npgsql, snake_case naming, the first
   migration.
5. **Containers** per `@.standards/ops/containers.md`: Dockerfile, `.dockerignore`, a
   compose file with the database and a health check.
6. **CI/CD** per `@.standards/ops/ci-cd.md`: both workflows, with `submodules: recursive`
   in every checkout.
7. **Architecture tests** per `@.standards/dotnet/testing.md`, so the layer rules are
   enforced from the first commit rather than from the first review that notices.
8. **Project `CLAUDE.md`**: copy the matching template from `@.standards/templates/` and
   fill in every answer from the questions above. Leave no placeholder behind.

## Wrapping up

- Run `dotnet build` and `dotnet test`. Both green before you hand over.
- Run `docker compose up` and confirm the application starts and reaches the database.
- Check that the project `CLAUDE.md` contains no remaining `<placeholder>`.
- Report which decisions were made, which steps you skipped, and anything the developer
  still has to decide.
