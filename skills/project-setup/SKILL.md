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

Ten things cannot be inferred from an empty repository, and all ten are expensive to
change later. Ask the developer, one at a time, and do not guess.

Ask the questions in this order, because later answers depend on earlier ones.

**A plan is not an answer.** When the developer brings a plan that already proposes answers,
put each proposal to them as its own question, with what it means. Never fold several
into one "agree with all of these?". A proposal confirmed in a bundle is a proposal nobody
looked at, and the first project that did this found out halfway through the setup that it
was multi-tenant after all.

**Every question gets asked.** Do not skip one because the plan seems to answer it, and do
not wait for the developer to bring a topic up. A question that only gets asked when the
subject happens to come up gets asked too late.

### 1. Product name

Becomes the prefix of every project: `Billing.Domain`, `Billing.Api`. Short, no company
prefix, ideally the same as the repository name.

Getting this wrong means renaming every project, namespace and folder later. See
`@.standards/dotnet/solution-layout.md`.

### 2. Domain language

Are the domain concepts in the language of the users, or in English?

This is not the same question as which language the code is written in. Technical
vocabulary is always English. The domain concepts are a choice with two valid answers,
see `@.standards/general/language.md` chapter 2:

- **The language of the users**: a Dutch insurer gets `Polis` and `CreatePolisCommand`,
  and the route `/polissen/:polisId/edit`.
- **English**: a Dutch harbour gets `Member` and `CancelMembershipCommand`, and the route
  `/members/:memberId/edit`. Usual when the project continues an existing application or
  data model that is already English.

Ask it explicitly, with both options, and ask in which language the users work. A team
that never decides ends up with both, which is the one outcome the standard rules out.
Record the answer in the project's `CLAUDE.md`. For English, ask for the main business
words too and write the glossary (users' word -> code word) right away. Route action
segments (`new`, `edit`) are English either way.

### 3. Architecture: domain-driven design, unless the domain is only lists and forms

Domain-driven design is the default: `@.standards/dotnet/DDD.md` on top of
`@.standards/dotnet/ARCHITECTURE.md`. That means aggregates that guard their own rules, no
public setters, one repository per aggregate root, a `DomainRuleException` for a rule that
says no, and an audit log of every change. With tenants it adds one query filter per tenant.

Ask it every time, as a question of its own, right after the domain language. Ask it before
tenants, roles or the stack come up. Never skip it because the plan looks simple, and never
wait until tenants or a business rule come up in conversation. The first project that did
that rebuilt its backend to DDD halfway through the setup.

Put it as the default with one way out, and explain both in these terms:

- **Domain-driven design**, the default. Take it unless the developer gives a reason for the
  lighter variant.
- **ARCHITECTURE.md alone**: entities with private setters and a few intent methods, and the
  rules mostly in the handlers. Less code per feature. It only fits a domain of lists and
  forms with field validation: no rule that spans fields, no state that moves in steps, no
  records that are only consistent together, and no tenants. Record it as a deviation, with
  the reason.

Chapter 1 of DDD.md says when the lighter variant fits. Record the choice in the project
`CLAUDE.md`.

With DDD the scaffold changes in four places: a `<Product>.Domain.UnitTests` project, the
`DomainRuleException` and its handler, the extra architecture test of DDD.md chapter 8, and,
with tenants, the tenant interface with the query filter in the `DbContext`. Import
`@.standards/dotnet/DDD.md` in the project `CLAUDE.md`.

### 4. Tenants and who signs in

Does one installation serve several businesses, clubs or practices whose data must stay
apart, and do people from those sign in themselves? Ask it on its own, and ask it even when
the plan says "one user": the owner alone today and a customer's own staff next month is
exactly how this goes wrong. Adding tenants later touches every query, the roles table and
the way a request knows who it is for.

When the answer is yes, ask four more things, because each one changes the scaffold:

- **What is the tenant called** in the business: a club, a customer, a practice. That word
  becomes `I<Tenant>Owned` and `<Tenant>Id`.
- **Is there a role above all tenants**, someone who sees and manages every one of them,
  usually the owner? That role has no tenant in its `user_roles` row.
- **Can one person hold a role in more than one tenant**, a bookkeeper for two clubs? Then
  ask whether they see their tenants together, or choose one at a time with a switcher. The
  second keeps "one tenant per request" from DDD.md chapter 4 intact and is the usual answer.
- **Who may sign in**: only people an admin invited, linked on their first sign-in by a
  verified email address, or anybody, who then gets no rights until somebody grants them.
  Invited only is the usual answer, and a login nobody invited gets a 403 `no-access`.

Then name the roles per tenant at question 7. Record all of it in the project `CLAUDE.md`
under Tenants. A multi-tenant project is always DDD. If question 3 chose the lighter
variant, go back to it now and say why.

### 5. Entry points

The stack is not a question. Every project is a .NET API in `src/<Product>.Api` and a
React frontend in `src/<Product>.Web`, in one repository. Do not ask whether the project
needs a frontend or a backend.

What you do ask: is there an entry point beyond those two, such as a worker? Each .NET
entry point is its own project and its own container image.

Ask the frontend technology choices too: bundler, routing, data access, state, styling,
forms and component library. The React standard leaves these open on purpose, see
`@.standards/react/ARCHITECTURE.md`, so the project has to decide them and record them.
An answer of "not decided yet" is fine for anything the first screen does not need.

### 6. Database

PostgreSQL unless there is a reason. If there is a reason, record it as a deviation with
the reason attached, otherwise someone will "fix" it later.

Ask which major version, because it has to match across the compose file, CI and
production.

### 7. Authentication provider

Kinde or Entra ID, and which tenant. Authentication is never built in-house and passwords
are never stored. See `@.standards/ops/database.md` for the identity model that goes with
it: our own `users` table keyed by our own UUID, and `user_identities` holding the
provider and its subject.

Ask for the system user id as well, or decide to generate one. Every row needs a
`created_by`, including the rows no logged-in person ever creates.

Ask which roles the application needs, and start with as few as the answer allows. With
tenants, ask for each role whether it holds per tenant or above all of them, and name what
each may do; the answers of question 4 already say whether there is a role above all
tenants and whether one person can hold roles in several. Where
they live is not a question: in our own database, in `user_roles`, never as roles or
permissions at the provider. See `@.standards/general/security.md` chapter 2 and
`@.standards/ops/database.md` chapter 4. Do not tell the developer to create roles or
permissions in Kinde or Entra ID. The first admin comes from `Authorization:AdminSubjects`
per environment; ask for the developer's subject at the provider, or leave the
development setting empty and say where to find it after the first sign-in. Scaffold the
`/me` endpoint that returns the user's roles, so the frontend can show what they may do,
and the integration test that a role claim inside the token gives a 403. With tenants, `/me`
also returns the roles per tenant with the tenant's name and the tenant the request works in,
so the frontend can offer the switcher, and the integration tests cover a switch to somebody
else's tenant being refused.

The access token supplies the subject and the email address, nothing else. For Kinde, tell
the developer to switch on only **Email** under the application's Tokens, "Customize access
token". The name comes from the ID token: scaffold `POST /me/sync`, which the frontend calls
after signing in with the given and family name from the provider's SDK, and a domain
method that fills in an empty name only. See `@.standards/ops/database.md` chapter 4.

### 8. Local ports

Two fixed ports on `localhost`, one for the frontend and one for the API, and the dashboard one
below the frontend. Do not take the defaults of the tools, and do not invent a pair: read
`@.standards/ops/ports.md`, propose the next free block of ten, and ask whether that is fine.
Add the project to the table in that file in a standards pull request, or record the block in
the project `CLAUDE.md` so the next standards pull request picks it up.

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
- The API under Aspire and `dotnet run`, in `launchSettings.json`: one `https` profile, no
  `http` one.
- The frontend endpoint in `AppHost.cs`: the frontend port, HTTPS, not proxied. The
  dashboard in the AppHost's `launchSettings.json`: the port below the frontend's.
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

### 9. Environments and hosting

Where does this run, and who is allowed to deploy to it? The answer drives the deploy
jobs in the CD workflow. See `@.standards/ops/environments.md`.

**Development is local** (localhost, Aspire) and gets no hostname: do not ask for a
`dev.<domain>`. Ask which deployed environments there are: production always, test when
there is somewhere to run it (a home cluster usually means from the start), acceptance only
when somebody outside the team signs off. For each deployed environment, ask:

- The hostnames, nested under the environment label rather than prefixed:
  `test.<domain>` and `api.test.<domain>`, production without a prefix.
- How a non-production environment is kept private: reachable only from a private network
  such as Tailscale, or an IP allowlist. Not basic authentication, which breaks the API calls
  of the frontend; environments.md chapter 2 says why.
- Whether the database is rebuilt on every deploy or persists. Test is rebuilt,
  acceptance persists. Getting this backwards costs you either reproducibility or
  realism, and you will not notice which until you need it.
- Who approves a deploy to it.

**Application registrations at the identity provider** follow from the team, so ask
whether the project has more than one developer here, not only at branch protection:

- **One developer**: one registration for every environment, the standard for solo
  projects, on the condition that authorization lives in the application's own database
  (users and roles per environment), not in the token. Do not ask; record it, with where
  authorization lives. **With an API per environment**: audiences `<product>-local`,
  `<product>-test` and `<product>-production`, the application switched on for each API at
  the provider, and the audience in the configuration of both frontend and API per
  environment. Tell the developer to create all three APIs now. See environments.md
  chapter 4.
- **More than one developer**: ask, one registration for every environment or one per
  environment, and explain the difference in the terms of environments.md chapter 4: with
  one, a token from test is accepted by production. Recommend one per environment when
  others sign in to test, when test holds data others should not see in production, or when
  the token carries roles or permissions.

If none of this is decided yet, say so in the project `CLAUDE.md` rather than inventing
something.

### 10. Documentation language

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
   `users`, `user_identities` and `user_roles` tables, the seeded system user, `IAuditableEntity` with
   its interceptor, and the first migration.
5. **Frontend** in `src/<Product>.Web/` per `@.standards/dotnet/solution-layout.md` and
   `@.standards/react/ARCHITECTURE.md`: its own `package.json`, TypeScript, the feature
   folder layout, lint, typecheck and test scripts, and one feature that calls the example
   slice from step 3. Not in the `.sln`, and no `package.json` in the repository root.
6. **Containers** per `@.standards/ops/containers.md`: Dockerfile, `.dockerignore`, a
   compose file with the database and a health check, and the API on HTTPS on its own port.
   The frontend does not get an image.
7. **Running locally** per `@.standards/ops/containers.md` chapter 3 and
   `@.standards/dotnet/solution-layout.md` chapter 2: `.AppHost` with Postgres, pgAdmin,
   the migrator, the API and the frontend; `.DbMigrator`; `.ServiceDefaults`, called by the
   API; `aspire.config.json` in the root. Check that the Aspire CLI is installed with
   `aspire --version`, and ask the developer to install it with
   `dotnet tool install --global aspire.cli` when it is not.
8. **CI/CD** per `@.standards/ops/ci-cd.md`: the workflows, with `submodules: recursive`
   in every checkout. CD triggers on a successful CI run, never on push, or a red test
   will not stop a deploy.
9. **Branch and protection** per `@.standards/general/git-workflow.md`: one long-lived
   branch, `main`, protected, with the required checks on it. There is no `develop`. Whether
   the project has more than one developer, asked at question 9, decides whether a review
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

   Protection is a ruleset, applied from a file and only after the scaffold pull request is
   merged, because the checks it requires do not exist before CI has run once:

   ```bash
   gh api -X POST repos/<owner>/<repo>/rulesets --input .standards/templates/ruleset-main.json
   ```

   With more than one developer, set `required_approving_review_count` to 1 first. Then
   prove that it took: make an empty commit on `main`, push it, and expect "push declined
   due to repository rule violations". Reset the local commit afterwards. From that moment
   you cannot push to `main` either, and that is the point.

   GitHub refuses this on a private repository on a free plan, with "Upgrade to GitHub Pro
   or make this repository public". When that happens, do not work around it. Say so, record
   it in the project `CLAUDE.md` as a deviation with the condition under which it gets
   turned on, and leave the choice between a paid plan and a public repository to the
   developer. Secret scanning is a separate case: on a private repository of a personal
   account it is not available on any plan, so record that as a deviation and do not
   present it as something an upgrade fixes.

   The very first CD run deserves a look as well. It only happens after the merge, so a
   mistake in `cd.yml` cannot show on the pull request. Check that it went green and that
   the log says "pushing manifest" for the image.
10. **Architecture tests** per `@.standards/dotnet/testing.md`, so the layer rules are
   enforced from the first commit rather than from the first review that notices.
11. **End-to-end tests** in `tests/<Product>.E2ETests/`: Playwright with its own
    `package.json`, and one journey that goes through the frontend to the example slice.
12. **A pull request template** carrying the test evidence block from
    `@.standards/general/testing.md`, in `.github/pull_request_template.md`.
13. **Project `CLAUDE.md`**: copy `@.standards/templates/CLAUDE.project.md` and fill in
    every answer from the questions above. Leave no placeholder behind.

## Wrapping up

- Run `dotnet build` and `dotnet test`. Both green before you hand over.
- In `src/<Product>.Web`, run the lint, the typecheck, the tests and the build. All green.
- Run `aspire run` and confirm the application starts and reaches the database:
  `https://localhost:<API port>/health/ready` answers 200, with the certificate verified and
  not skipped. A 401 on `/` is the API working, not a fault: every endpoint requires a
  signed-in user unless it opts out, and nothing lives at `/`. Tell the developer, because
  it is the first thing they will open in a browser.
- Confirm the frontend is on `https://localhost:<frontend port>`, served by Vite itself with
  `--port <frontend port>` and not by a leftover dev server, and that the API accepts a CORS
  preflight from that origin. Check that pgAdmin opens from the dashboard, and that
  `git status` shows no changed `package-lock.json`.
- Stop Aspire and run `docker compose up` once as well: the image has to start on its own,
  because that is what the journeys and a deployed host run.
- Check that the project `CLAUDE.md` contains no remaining `<placeholder>`.
- Report which decisions were made, which steps you skipped, and anything the developer
  still has to decide.
