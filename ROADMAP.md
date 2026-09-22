# What is not done yet

Everything in this repository that is still open, in one place, so that picking it up
again does not depend on anybody remembering a conversation.

Two rules for keeping this file useful:

- **Close things by moving them out.** When a question gets answered, the answer goes in
  the document that owns it and the line disappears from here. This file gets shorter by
  being resolved, not by being tidied.
- **It is not a feature backlog.** Only open decisions, missing documents, and the things
  that have to be revisited when something changes.

One caveat that applies to all of it: none of these standards has been through a real
project yet. The first one will find things that reading cannot.

---

## 1. Missing documents

**`ops/hosting.md`.** How the host itself is set up: reverse proxy, TLS, unattended
upgrades, firewall, monitoring, and who patches the kernel.
[`ops/environments.md`](ops/environments.md) covers what differs between environments and
explicitly assumes this document exists. The target is a Debian host running Docker.

**`SECURITY.md`.** The repository is public, so it should say where to send a security
report and what to expect back. Tracked as an open question in
[`general/security.md`](general/security.md).

---

## 2. Open decisions

Eighteen open items sit in the documents themselves, as `- [ ]` lines. Grouped by how
ready they are:

### Ready to decide, just not decided

| Question | Where |
|---|---|
| Contract tests on top of the committed OpenAPI document, or is the diff enough | [`general/api-contracts.md`](general/api-contracts.md) |
| A periodic security review or pentest, and who pays | [`general/security.md`](general/security.md) |
| Who is the contact for a security report | [`general/security.md`](general/security.md) |
| Authentication inside the compose stack, so the journeys have something to log in to | [`ops/ci-cd.md`](ops/ci-cd.md) ch. 4 |
| How the frontend is served in the compose stack the journeys run against. The job points `BASE_URL` at the API, and nothing in the stack serves the React build | [`ops/ci-cd.md`](ops/ci-cd.md) ch. 4, [`ops/containers.md`](ops/containers.md) ch. 7 |
| How the frontend gets deployed. CD builds and ships the API image; the static build output has no job, no target and no rollback | [`ops/ci-cd.md`](ops/ci-cd.md) ch. 5, [`ops/environments.md`](ops/environments.md) |

The authentication one has three options written out with what each costs. It matters because one
of them puts a test-only authentication bypass into production code, which is not a thing
to end up with by accident.

### Waiting on a trigger, not on a decision

| Question | Decide when |
|---|---|
| A `.NET` client and therefore `.Contracts` | a second .NET project calls the API |
| Supporting a customer on an older version | it actually happens, and it changes the branching model |

### Deliberately parked: React

Thirteen of the eighteen are React, eight in
[`react/ARCHITECTURE.md`](react/ARCHITECTURE.md) and five in
[`react/testing.md`](react/testing.md). Both documents carry a draft banner. They cover
data access, state, styling, forms, routing, the component library, folder structure,
Vitest versus Jest, MSW, coverage, and whether there is a React equivalent of the .NET
architecture tests.

They stay open on purpose. There is no frontend yet, and answering them now is deciding
without a reason to. Every project is full stack, so the first real project will force
most of them. Decide them there, record them in that project's `CLAUDE.md`, and move what
turns out to be generic back here.

What is settled is where the frontend lives, `src/<Product>.Web/`, and where the journeys
live, `tests/<Product>.E2ETests/`. See
[`dotnet/solution-layout.md`](dotnet/solution-layout.md).

The Playwright policy is part of this. The end-to-end job exists in
[`ops/ci-cd.md`](ops/ci-cd.md) and runs against the compose stack, but which journeys are
worth covering, how many, and what to do about a flaky one is not written down anywhere.

---

## 3. Known design questions

These are not marked in any document, because they are about the shape of the repository
rather than about a rule inside it.

### Every endpoint in Program.cs

[`dotnet/ARCHITECTURE.md`](dotnet/ARCHITECTURE.md) puts every endpoint in `Program.cs`. The
first project had thirteen after its second feature, and `Program.cs` becomes the one file
every feature edits. The usual way out is a static class per feature with a
`Map<Feature>Endpoints(this IEndpointRouteBuilder)` extension and a `MapGroup` per resource,
which also puts the route prefix and the OpenAPI tag in one place. Decide it before the
third feature of a project, not after the tenth.

### The eager imports are heavy

A `CLAUDE.md` built from [`templates/CLAUDE.project.md`](templates/CLAUDE.project.md)
pulls in fourteen documents through `@` imports, and those load into context at the start
of every session, before anything is asked.

| | |
|---|---|
| Words | 18,039 |
| Rough estimate | ~31,000 tokens |

The four `ops/` documents are over 7,500 words of that, and they only matter when somebody
is working on the pipeline, the database or the environments. That is the profile of
something that should be a skill, which loads on demand, rather than an import, which
always loads.

Splitting them that way is a real restructure: the content would move from a document a
person reads into a skill an agent invokes, and those are not the same thing. Worth doing,
worth thinking about first.

### Skills and standards travel differently

The skills reach a project as a plugin, installed once. The standards reach it as a
submodule, pinned per project. That is the right split, and it means a project can have
one without the other, at which point the skills point at `.standards/` paths that are not
there. Each skill says so at the top, which is a note rather than a solution.

---

## 4. Revisit when something changes

The documents are full of rules that are correct now and will stop being correct later.
They are collected here because a conditional written inside a document is a conditional
nobody goes back to read.

| When this happens | Then | Written in |
|---|---|---|
| A second developer joins | required reviewers on, self-approval off, PR size starts mattering | [`general/git-workflow.md`](general/git-workflow.md) ch. 4 |
| A second .NET project calls the API | introduce `.Contracts` | [`dotnet/solution-layout.md`](dotnet/solution-layout.md) ch. 3 |
| The first breaking API change | `/v2` in the path, both versions side by side | [`general/api-contracts.md`](general/api-contracts.md) ch. 1 |
| Deep pages get slow | move to cursor-based pagination | [`general/api-contracts.md`](general/api-contracts.md) ch. 4 |
| You publish something somebody else consumes | semantic versioning, and reconsider Conventional Commits | [`general/git-workflow.md`](general/git-workflow.md) ch. 2 and 6 |
| Infrastructure gets crowded | split off `.Persistence`, and `.Migrations` if they run on their own | [`dotnet/solution-layout.md`](dotnet/solution-layout.md) ch. 3 |
| A release needs stabilizing while `main` carries on, or production needs a fix while `main` holds something that must not go live | branch from the `production` tag, and ask first why a switch did not cover it | [`general/git-workflow.md`](general/git-workflow.md) ch. 1 |
| The Promote workflow has run for the first time | check that the `production` tag moved, and that a rollback moves it back. Neither has ever been run | [`ops/ci-cd.md`](ops/ci-cd.md) ch. 6 |
| Production carries something you would be called about at night | move it off the shared host | [`ops/environments.md`](ops/environments.md) ch. 5 |
| Somebody outside the team signs off on releases | add the acceptance environment | [`ops/environments.md`](ops/environments.md) ch. 1 |
| `EFCore.NamingConventions` catches up with the EF Core major | drop the manual snake_case fallback | [`ops/database.md`](ops/database.md) ch. 1 |

The first row is the one to put a reminder on. The moment a team forms is exactly the
moment nobody has time to notice that the rules were written for one person.
