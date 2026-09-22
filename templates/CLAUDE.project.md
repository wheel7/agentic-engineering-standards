# CLAUDE.md - <PROJECT NAME>

<!--
Copy this file to the root of your project as CLAUDE.md.

Every project under these standards is full stack: a .NET API and a React frontend in one
repository. That is why there is one template and not one per stack.

Precondition: the standards are added as a submodule in .standards
    git submodule add https://github.com/wheel7/agentic-engineering-standards.git .standards

The skills come from the same repository, installed once as a plugin rather than
per project:
    claude plugin marketplace add wheel7/agentic-engineering-standards
    claude plugin install wheel7@agentic-standards

Keep this file THIN. Anything that applies to other projects too does not belong here,
but in the agentic-engineering-standards repo. Replace all <placeholders> below.
-->

## Standards

These apply to this project. They live in the submodule `.standards`; changes to them
go through a PR in that repo, not here.

@.standards/general/language.md
@.standards/general/testing.md
@.standards/general/git-workflow.md
@.standards/general/security.md
@.standards/general/api-contracts.md
@.standards/dotnet/ARCHITECTURE.md
@.standards/dotnet/solution-layout.md
@.standards/dotnet/testing.md
@.standards/react/ARCHITECTURE.md
@.standards/react/testing.md
@.standards/ops/database.md
@.standards/ops/containers.md
@.standards/ops/environments.md
@.standards/ops/ci-cd.md

> Note: the React standards are a **draft** at the moment. If this project deviates from
> them, note that below - that is valuable input for the review.

Updating to the latest version:

```bash
git submodule update --remote .standards
```

Do that in a PR of its own, so the change in the standards is visible in the diff.

---

## Specific to this repo

### Solution

<!-- The product name determines the project names: <Product>.Domain, <Product>.Api, etc.
     The React frontend is an entry point like any other and lives in src/<Product>.Web.
     See .standards/dotnet/solution-layout.md. -->

- **Product name**: <Product>
- **Entry points**: <Product>.Api, <Product>.Web <and optionally .Worker>

### Language

<!-- Technical vocabulary is always English. Domain concepts follow the language the
     business itself uses. See .standards/general/language.md. Decide this once, here,
     because a team that never decides ends up with both. -->

- **Domain concepts**: <English / Dutch / ...>
- **Agreed domain terms**: <Polis, Schademelding, ... - the words that must not be translated>
- **Documentation in this repo**: <English / Dutch>
- **Commit messages and PRs**: English

### Domain

<!-- What is this application about? Which key concepts do you need to know?
     For example: entities, how they relate to each other, and the most important
     business rules that cannot be read from the code. -->

- **Key concepts**: <...>
- **Most important business rules**: <...>
- **Most important screens / flows**: <...>
- **External systems we integrate with**: <...>

### Authentication

<!-- Authentication runs through an external provider. The identity model, the users and
     user_identities tables and the audit columns are described in
     .standards/ops/database.md. Only record what is specific to this project here. -->

- **Provider**: <Kinde / Entra ID>
- **Tenant or environment**: <...>
- **System user id**: <the seeded UUID used for writes with no logged-in user>
- **How the frontend logs in**: <which flow, and where the token comes from>
- **Registered for development**: <https://localhost:xxxx> as callback URL and as logout URL

### Database

<!-- Which database, where does it run, how do you get to it? -->

- **Type**: PostgreSQL <major version, same in the AppHost, docker-compose.yml and production>
- **Local connection string**: set by the AppHost under Aspire, and in `docker-compose.yml`
  for the compose stack; if this project deviates from that, note here how.
- **Looking in the database**: pgAdmin, through its link in the Aspire dashboard.
- **Migrations**: `aspire run` applies them through `.DbMigrator`. Adding one, and applying it
  to the compose database:

```bash
dotnet ef migrations add <Name> --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
dotnet ef database update       --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
```

- **Test data / seeding**: <...>

### Frontend technology choices

<!-- The React standard deliberately leaves these choices open. Fill in what THIS project does. -->

- **Framework / bundler**: <Vite / Next.js / ...>
- **Routing**: <...>
- **Server state / data access**: <TanStack Query / fetch in hooks / ...>
- **Client state**: <...>
- **Styling**: <Tailwind / CSS Modules / ...>
- **Forms and validation**: <...>
- **Component library**: <...>
- **Is the API client generated from OpenAPI?**: <yes/no, and with which command>

### Running locally

```bash
# Once per machine:
dotnet tool install --global aspire.cli

# Everything: database with pgAdmin, migrations, API and frontend, with the dashboard.
aspire run

# The compose stack, for the journeys. It takes the API port too, so not next to Aspire.
docker compose up -d --wait
```

<!-- Two fixed ports, chosen during setup and not the defaults of the tools, both on HTTPS.
     The identity provider only redirects to a registered callback URL, scheme and port
     included, so the frontend has to stay where it was registered.
     See .standards/ops/containers.md chapter 3. -->

- **Aspire dashboard**: <https://localhost:xxxx - 1>
- **Local frontend URL**: <https://localhost:xxxx>, fixed and strict
- **Local API URL**: <https://localhost:yyyy>, under Aspire and `dotnet run`, and as the
  host side of the port mapping in compose.
- **Local HTTPS certificate**: the ASP.NET development certificate, once per machine, from
  the repository root. `.certs/` holds a private key and is not in git.

```bash
dotnet dev-certs https --trust
dotnet dev-certs https -ep .certs/localhost.pem --format Pem -np
# Linux and macOS only:
chmod 644 .certs/localhost.pem .certs/localhost.key
```

- **Required secrets**: <which ones, and how do you set them up - e.g. dotnet user-secrets>
- **Required frontend environment variables**: <which ones, and where do you set them - e.g. .env.local>
- **Dependencies that must be running**: <database, message broker, mock services>
- **Running backend tests**: `dotnet test`
- **Running frontend tests**: `npm test` in `src/<Product>.Web`
- **Lint and typecheck**: `npm run lint` / `npx tsc -b` in `src/<Product>.Web`
- **Running the journeys**: `npx playwright test` in `tests/<Product>.E2ETests`

### Hosting and environments

<!-- Where does this run, how does it get there, and how do you get it back when it goes
     wrong? The generic conventions are in .standards/ops/environments.md; here only what
     differs per project. Drop the rows for environments this project does not have. -->

- **API runs on**: <own Debian host with Docker / ... >
- **Frontend is served from**: <static hosting / CDN / ... >
- **Deploy**: <which workflow, and what triggers it>
- **Rolling back**: <how, and who is allowed to>
- **Logs and alerting**: <where do you look when it goes wrong>

| Environment | URLs | Database | App registration | Deploy approved by |
|---|---|---|---|---|
| Test | <test.example.com, api.test.example.com> | rebuilt on deploy | <name> | automatic |
| Acceptance | <acc.example.com, api.acc.example.com> | persists | <name> | <who> |
| Production | <example.com, api.example.com> | persists | <name> | <who> |

- **Seed data for test**: <where it lives, and what is in it>
- **Acceptance refresh**: <how often, and where the anonymization runs>
- **Secrets**: <which GitHub Environment holds what>

### Deviations from the standard

<!-- Does this project deliberately deviate from .standards? Note that here WITH the reason.
     Without a reason it gets accidentally "fixed" later on. If nothing is listed here,
     the standard applies in full. -->

- <No known deviations.>

### Other

<!-- Pitfalls, oddities that grew over time, things everyone trips over. -->

- <...>
