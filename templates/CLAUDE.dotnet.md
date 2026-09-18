# CLAUDE.md - <PROJECT NAME>

<!--
Copy this file to the root of your .NET project as CLAUDE.md.

Precondition: the standards are added as a submodule in .standards
    git submodule add https://github.com/wheel7/agentic-engineering-standards.git .standards

The skills come from the same repository, installed once as a plugin rather than
per project:
    /plugin marketplace add wheel7/agentic-engineering-standards
    /plugin install wheel7@agentic-standards

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
@.standards/ops/database.md
@.standards/ops/containers.md
@.standards/ops/environments.md
@.standards/ops/ci-cd.md

Updating to the latest version:

```bash
git submodule update --remote .standards
```

Do that in a PR of its own, so the change in the standards is visible in the diff.

---

## Specific to this repo

### Solution

<!-- The product name determines the project names: <Product>.Domain, <Product>.Api, etc.
     See .standards/dotnet/solution-layout.md. -->

- **Product name**: <Product>
- **Entry points**: <Product>.Api <and optionally .Worker, .Blazor>

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
- **External systems we integrate with**: <...>

### Authentication

<!-- Authentication runs through an external provider. The identity model, the users and
     user_identities tables and the audit columns are described in
     .standards/ops/database.md. Only record what is specific to this project here. -->

- **Provider**: <Kinde / Entra ID>
- **Tenant or environment**: <...>
- **System user id**: <the seeded UUID used for writes with no logged-in user>

### Database

<!-- Which database, where does it run, how do you get to it? -->

- **Type**: PostgreSQL <major version, same as docker-compose.yml and production>
- **Local connection string**: comes from `docker-compose.yml`; if this project deviates
  from that, note here how.
- **Migrations**:

```bash
dotnet ef migrations add <Name> --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
dotnet ef database update       --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
```

- **Test data / seeding**: <...>

### Running locally

```bash
# Everything in containers, including the database:
docker compose up

# Or only the database in a container and the API outside it:
docker compose up -d db
dotnet run --project src/<Product>.Api
```

- **Local URL**: <https://localhost:xxxx>
- **Required secrets**: <which ones, and how do you set them up - e.g. dotnet user-secrets>
- **Dependencies that must be running**: <database, message broker, mock services>
- **Running tests**: `dotnet test`

### Hosting and environments

<!-- Where does this run, how does it get there, and how do you get it back when it goes
     wrong? The generic conventions are in .standards/ops/environments.md; here only what
     differs per project. Drop the rows for environments this project does not have. -->

- **Runs on**: <own Debian host with Docker / ... >
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
