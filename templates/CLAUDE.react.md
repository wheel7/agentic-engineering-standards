# CLAUDE.md - <PROJECT NAME>

<!--
Copy this file to the root of your React project as CLAUDE.md.

Precondition: the standards are added as a submodule in .standards
    git submodule add https://github.com/wheel7/agentic-engineering-standards.git .standards

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
@.standards/react/ARCHITECTURE.md
@.standards/react/testing.md
@.standards/ops/environments.md

> Note: the React standards are a **draft** at the moment. If this project deviates from
> them, note that below - that is valuable input for the review.

Updating to the latest version:

```bash
git submodule update --remote .standards
```

Do that in a PR of its own, so the change in the standards is visible in the diff.

---

## Specific to this repo

### Language

<!-- Technical vocabulary is always English. Domain concepts follow the language the
     business itself uses. See .standards/general/language.md. Decide this once, here,
     because a team that never decides ends up with both. -->

- **Domain concepts**: <English / Dutch / ...>
- **Agreed domain terms**: <Polis, Schademelding, ... - the words that must not be translated>
- **Documentation in this repo**: <English / Dutch>
- **Commit messages and PRs**: English

### Domain

<!-- What is this application about? Which screens and concepts do you need to know? -->

- **Key concepts**: <...>
- **Most important screens / flows**: <...>

### Technology choices

<!-- The React standard deliberately leaves these choices open. Fill in what THIS project does. -->

- **Framework / bundler**: <Vite / Next.js / ...>
- **Routing**: <...>
- **Server state / data access**: <TanStack Query / fetch in hooks / ...>
- **Client state**: <...>
- **Styling**: <Tailwind / CSS Modules / ...>
- **Forms and validation**: <...>
- **Component library**: <...>

### Backend / API

- **Local API URL**: <http://localhost:xxxx>
- **Where the backend repo lives**: <...>
- **Authentication**: <how does the frontend log in, where does the token come from>
- **Is the client generated from OpenAPI?**: <yes/no, and with which command>

### Running locally

```bash
# <Fill in the commands that actually work here>
npm install
npm run dev
```

- **Local URL**: <http://localhost:5173>
- **Required environment variables**: <which ones, and where do you set them - e.g. .env.local>
- **Running tests**: `npm test`
- **Lint and typecheck**: `npm run lint` / `npx tsc --noEmit`

### Deviations from the standard

<!-- Does this project deliberately deviate from .standards? Note that here WITH the reason. -->

- <No known deviations.>

### Other

<!-- Pitfalls, oddities that grew over time, things everyone trips over. -->

- <...>
