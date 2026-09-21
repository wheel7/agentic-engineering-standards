---
name: project-setup
description: Sets up a new full-stack repository against these engineering standards - a .NET API and a React frontend in one repository - including the solution layout, frontend, database, container setup and CI/CD. Use when someone starts a new project, creates a new solution, or asks how to bootstrap a repository that follows the team standards - for example "set up a new application for billing" or "start a new project". Do not use for adding a feature to an existing project; use dotnet-feature or react-component for that.
---

# Setting up a new project

This skill turns the standards in `.standards/` into a working repository. It is the one
place where the decisions that cannot be derived from code get made and recorded.

> The `.standards/` paths below assume this project has the standards as a git submodule.
> Without it, the same documents are in the repository this skill was installed from.

## Ask first, scaffold second

Eight things cannot be inferred from an empty repository, and all eight are expensive to
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

### 3. Entry points

The stack is not a question. Every project is a .NET API in `src/<Product>.Api` and a
React frontend in `src/<Product>.Web`, in one repository. Do not ask whether the project
needs a frontend or a backend.

What you do ask: is there an entry point beyond those two, such as a worker? Each .NET
entry point is its own project and its own container image.

Ask the frontend technology choices too: bundler, routing, data access, state, styling,
forms and component library. The React standard leaves these open on purpose, see
`@.standards/react/ARCHITECTURE.md`, so the project has to decide them and record them.
An answer of "not decided yet" is fine for anything the first screen does not need.

### 4. Database

PostgreSQL unless there is a reason. If there is a reason, record it as a deviation with
the reason attached, otherwise someone will "fix" it later.

Ask which major version, because it has to match across the compose file, CI and
production.

### 5. Authentication provider

Kinde or Entra ID, and which tenant. Authentication is never built in-house and passwords
are never stored. See `@.standards/ops/database.md` for the identity model that goes with
it: our own `users` table keyed by our own UUID, and `user_identities` holding the
provider and its subject.

Ask for the system user id as well, or decide to generate one. Every row needs a
`created_by`, including the rows no logged-in person ever creates.

### 6. Local ports

Two fixed ports on `localhost`, one for the frontend and one for the API. Ask for both, and
do not take the defaults of the tools.

This comes right after the authentication provider because that is what makes it matter.
The provider only redirects to a callback URL that was registered with it, scheme and port
included, so `https://localhost:<frontend port>` ends up typed into Kinde or Entra ID. If
the port moves, signing in stops working, with an error page at the provider that says
nothing about ports. The API needs the same origin for its CORS allowlist.

The scheme is not a question: both run on HTTPS locally, with the ASP.NET development
certificate, and there is no plain HTTP listener beside it. Do not ask whether the project
wants HTTPS, and do not scaffold an `http://localhost` URL anywhere.

The defaults are the wrong choice for a second reason: every Vite project wants 5173 and
every container example says 8080, so two projects on one machine collide, and whichever
starts second silently gets another port.

Apply the answer everywhere it appears, see `@.standards/ops/containers.md` chapter 3:

- The frontend dev server and preview, on HTTPS, with the port strict, so it fails rather
  than moving. It reads the certificate from `.certs`, only when a server starts, and stops
  with the two `dotnet dev-certs` commands in the message when the files are missing. A
  build and a test run must work without them.
- The API under `dotnet run`, in `launchSettings.json`: one `https` profile, no `http` one.
- The API in compose: the host side of the port mapping, `ASPNETCORE_URLS` and the Kestrel
  certificate paths as environment variables, and `.certs` mounted read-only. The image
  itself stays on plain 8080.
- The API URL the frontend is configured with, and the CORS allowlist for development.
- `.certs/` in `.gitignore` and in `.dockerignore`, before the certificate is exported.
- The callback and logout URLs to register at the provider, in the project `CLAUDE.md`.

Export the certificate yourself while scaffolding, so that what you hand over runs:
`dotnet dev-certs https -ep .certs/localhost.pem --format Pem -np`. Trusting it,
`dotnet dev-certs https --trust`, shows a prompt on the developer's machine, so check with
`dotnet dev-certs https --check --trust` and ask the developer to run it when it is not
trusted yet.

### 7. Environments and hosting

Where does this run, and who is allowed to deploy to it? The answer drives the deploy
jobs in the CD workflow.

Start from two, development and production, and only add more when something needs them.
See `@.standards/ops/environments.md`. For each deployed environment, ask:

- The hostnames, nested under the environment label rather than prefixed.
- Whether the database is rebuilt on every deploy or persists. Test is rebuilt,
  acceptance persists. Getting this backwards costs you either reproducibility or
  realism, and you will not notice which until you need it.
- Whether it has its own application registration at the identity provider. It has to.
  One registration shared across environments means a test token that production accepts.
- Who approves a deploy to it.

If none of this is decided yet, say so in the project `CLAUDE.md` rather than inventing
something.

### 8. Documentation language

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
4. **Database** per `@.standards/ops/database.md`: Npgsql, snake_case naming, the
   `users` and `user_identities` tables, the seeded system user, `IAuditableEntity` with
   its interceptor, and the first migration.
5. **Frontend** in `src/<Product>.Web/` per `@.standards/dotnet/solution-layout.md` and
   `@.standards/react/ARCHITECTURE.md`: its own `package.json`, TypeScript, the feature
   folder layout, lint, typecheck and test scripts, and one feature that calls the example
   slice from step 3. Not in the `.sln`, and no `package.json` in the repository root.
6. **Containers** per `@.standards/ops/containers.md`: Dockerfile, `.dockerignore`, a
   compose file with the database and a health check, and the API on HTTPS on its own port.
   The frontend does not get an image.
7. **CI/CD** per `@.standards/ops/ci-cd.md`: the workflows, with `submodules: recursive`
   in every checkout. CD triggers on a successful CI run, never on push, or a red test
   will not stop a deploy.
8. **Branch and protection** per `@.standards/general/git-workflow.md`: one long-lived
   branch, `main`, protected, with the required checks on it. There is no `develop`. Ask
   whether this project has more than one developer, because that decides whether a review
   is required or whether the checks carry it alone. Working alone changes who approves, not
   whether the protection is on, and not whether a change goes through a pull request.

   Two repository settings go with this, and neither is the GitHub default: squash merging
   only, with the pull request title as the commit title, and deleting the branch after a
   merge. Set them before the first pull request, or it goes in as a merge commit.

   The very first commits are the exception to "nobody pushes to `main`", because an empty
   repository has nothing to open a pull request against. Put the scaffold on a feature
   branch and open a pull request for it all the same: that run is the first proof that CI
   works on a clean runner, which it usually does not. After the merge, check with
   `gh workflow list` that GitHub knows all four workflows.

   Tell the developer how it works from here, because it is two decisions and people
   expect one: merging a pull request puts code in `main` and builds an image, and nothing
   goes live until they run the Promote workflow with the commit they want.

   GitHub refuses branch protection on a private repository on a free plan. When that
   happens, do not work around it. Say so, record it in the project `CLAUDE.md` as a
   deviation with the condition under which it gets turned on, and leave the choice between
   a paid plan and a public repository to the developer.
9. **Architecture tests** per `@.standards/dotnet/testing.md`, so the layer rules are
   enforced from the first commit rather than from the first review that notices.
10. **End-to-end tests** in `tests/<Product>.E2ETests/`: Playwright with its own
    `package.json`, and one journey that goes through the frontend to the example slice.
11. **A pull request template** carrying the test evidence block from
    `@.standards/general/testing.md`, in `.github/pull_request_template.md`.
12. **Project `CLAUDE.md`**: copy `@.standards/templates/CLAUDE.project.md` and fill in
    every answer from the questions above. Leave no placeholder behind.

## Wrapping up

- Run `dotnet build` and `dotnet test`. Both green before you hand over.
- In `src/<Product>.Web`, run the lint, the typecheck, the tests and the build. All green.
- Run `docker compose up` and confirm the application starts and reaches the database:
  `https://localhost:<API port>/health/ready` answers 200, with the certificate verified and
  not skipped. A 401 on `/` is the API working, not a fault: every endpoint requires a
  signed-in user unless it opts out, and nothing lives at `/`. Tell the developer, because
  it is the first thing they will open in a browser.
- Start the frontend dev server and confirm it is on `https://localhost:<frontend port>` and
  that the API accepts a CORS preflight from that origin.
- Check that the project `CLAUDE.md` contains no remaining `<placeholder>`.
- Report which decisions were made, which steps you skipped, and anything the developer
  still has to decide.
