# Agentic Engineering Standards

Shared engineering standards for full-stack projects, written so that people and AI
coding agents work from the same source of truth.

Every project these standards describe has the same shape: a .NET API and a React
frontend, in one repository. That is a premise and not an option, and it is why there is
one template, one layout and one pipeline rather than a variant per stack.

The content is ordinary engineering standards: architecture, naming, database
conventions, containers, CI/CD. What makes it agentic is the delivery. Every project
pulls this repository in as a git submodule and imports the parts it needs from its own
`CLAUDE.md`, so an agent working in that repository is held to the same conventions as
the person reviewing its pull request. The [`skills/`](skills/) folder goes a step
further with step-by-step instructions an agent can follow for recurring tasks.

If you came looking for an agent framework, this is not that. It is a set of rules, plus
the plumbing that makes an agent actually follow them.

The idea: generic knowledge lives here **once**, instead of being scattered across dozens
of project repositories. A project repository holds only a thin `CLAUDE.md` with what is
genuinely project-specific, such as the domain, the database and how to run it locally,
and imports the rest from here.

---

## Getting started

### Once per machine

Install the skills. They are a Claude Code plugin, so this is not per project. From a
shell:

```bash
claude plugin marketplace add wheel7/agentic-engineering-standards
claude plugin install wheel7@agentic-standards
```

Inside a running Claude Code terminal session the same two commands exist as
`/plugin marketplace add ...` and `/plugin install ...`. They do not work in the VS Code
extension, which answers that `/plugin` is not available in this environment, and they are
not shell commands either. If you only have the extension and `claude` is not on your
`PATH`, the extension ships the binary in its own folder under
`resources/native-binary/`, and the two shell commands above work with that.

Adding a marketplace registers this repository as a plugin source on your machine. Nothing
is published anywhere by doing it.

You now have `/wheel7:project-setup`, `/wheel7:dotnet-feature` and
`/wheel7:react-component` in every project on this machine.

### Per new project

Order matters, because the imports are read when a session starts.

```bash
# 1. In the new, empty repository
git submodule add https://github.com/wheel7/agentic-engineering-standards.git .standards

# 2. Copy the template and rename it
cp .standards/templates/CLAUDE.project.md CLAUDE.md

# 3. Only now start Claude Code
claude
```

Then run `/wheel7:project-setup`. It asks the eight things that cannot be derived from an
empty repository, product name, domain language and the local ports among them, and
scaffolds afterwards.
Finish by filling in the placeholders it could not answer for you in `CLAUDE.md`.

### Working on these standards themselves

Point Claude Code at a local checkout instead of the installed copy, and pick up edits
with `/reload-plugins`:

```bash
claude --plugin-dir /path/to/agentic-engineering-standards
```

---

## Structure

```
agentic-engineering-standards/
├── README.md
├── ROADMAP.md          What is still open, and what to revisit when
├── CODEOWNERS
├── .claude-plugin/     Makes this repo installable as a Claude Code plugin
│   ├── plugin.json
│   └── marketplace.json
├── general/            Stack-independent conventions
│   ├── language.md
│   ├── testing.md
│   ├── git-workflow.md
│   ├── security.md
│   └── api-contracts.md
├── dotnet/             .NET-specific
│   ├── ARCHITECTURE.md
│   ├── solution-layout.md
│   └── testing.md
├── react/              React-specific
│   ├── ARCHITECTURE.md
│   └── testing.md
├── ops/                Running and shipping
│   ├── database.md
│   ├── containers.md
│   ├── environments.md
│   └── ci-cd.md
├── skills/             Agent skills, step-by-step working instructions
│   ├── project-setup/SKILL.md
│   ├── dotnet-feature/SKILL.md
│   └── react-component/SKILL.md
└── templates/          Example CLAUDE.md to copy into a project
    └── CLAUDE.project.md
```

One repository for both halves of the stack. `general/` holds what applies everywhere,
with `dotnet/` and `react/` beside it. A project is always full stack, so it pulls in a
single submodule and imports all of it.

`ops/` sits on a different axis from the stack folders. Those describe how you write
code; `ops/` describes how the thing runs. PostgreSQL, containers and GitHub Actions are
the defaults. A project that deviates records that, with the reason, in its own
`CLAUDE.md`.

Two files are marked as a draft rather than as settled policy: the React architecture and
testing documents. Both say so at the top.

---

## Using this in a project

### 1. Add it as a submodule

The standards land in the `.standards` folder:

```bash
git submodule add https://github.com/wheel7/agentic-engineering-standards.git .standards
git commit -m "Add engineering standards as a submodule"
```

### 2. Cloning a project that already uses the submodule

```bash
git clone --recurse-submodules <project-url>
```

Already cloned without `--recurse-submodules`? Then:

```bash
git submodule update --init --recursive
```

This one bites in CI too. `actions/checkout` does not initialize submodules by default,
so `.standards` ends up empty and nothing complains. See
[`ops/ci-cd.md`](ops/ci-cd.md).

### 3. Updating to the latest standards

Updating is a **deliberate** act, through a pull request in the project, never
automatic. That way the team sees in the diff which conventions changed:

```bash
git submodule update --remote .standards
git add .standards
git commit -m "Update standards"
```

A submodule is pinned to a specific commit. A project therefore never falls behind or
runs ahead unexpectedly. It changes when you change it.

---

## Importing into your project `CLAUDE.md`

The `CLAUDE.md` in the root of your project pulls the standards in with `@` imports:

```markdown
@.standards/general/language.md
@.standards/general/git-workflow.md
@.standards/dotnet/ARCHITECTURE.md
@.standards/react/ARCHITECTURE.md
@.standards/ops/database.md
```

Copy [`templates/CLAUDE.project.md`](templates/CLAUDE.project.md) into the root of your
project as `CLAUDE.md` to get started. There is one template because there is one kind of
project: a .NET API and a React frontend in one repository. It imports every document, and
a project that genuinely has no use for one removes that line and records why under its
deviations.

Then fill in the "Specific to this repo" section. Anything that turns out to be generic
does not belong there, but here.

---

## Skills

[`skills/`](skills/) holds working instructions for recurring tasks: setting up a new
project, adding a feature, building a component. They are written for Claude Code and
read perfectly well as a checklist for a person.

[`project-setup`](skills/project-setup/SKILL.md) is the one to start with. It asks the
handful of decisions that cannot be derived from an empty repository, such as the product
name and the language of the domain, before it scaffolds anything.

Installing them is in [Getting started](#getting-started) above. There is one way in,
the plugin, and the commands are `/wheel7:project-setup`, `/wheel7:dotnet-feature` and
`/wheel7:react-component`.

### Where `.standards` comes from

It is not a folder in this repository and never will be. It is the path the submodule
lands on inside **your** project, chosen by the last argument of `git submodule add`:

```
my-project/
├── CLAUDE.md
├── src/
└── .standards/          <- the whole of this repository appears here
    ├── general/         <- imported by CLAUDE.md
    ├── dotnet/          <- imported by CLAUDE.md
    ├── react/           <- imported by CLAUDE.md
    ├── ops/             <- imported by CLAUDE.md
    ├── templates/       <- you copied CLAUDE.md out of here once
    └── skills/          <- on disk, but not where Claude Code loads skills from
```

So `.standards/dotnet/ARCHITECTURE.md` in a project is the same file as
[`dotnet/ARCHITECTURE.md`](dotnet/ARCHITECTURE.md) here.

The name is baked into around sixty references across the templates and the skills, so
mount it at `.standards` and nowhere else. Documents inside this repository never use
that prefix; they link to each other with ordinary relative paths, because inside the
submodule there is no `.standards` above them.

### Ignore `.standards/skills/`

A submodule is a full clone, so `skills/` lands in the project too. It is not where the
skills come from. Claude Code loads skills from `~/.claude/skills/`, from a project's own
`.claude/skills/`, or from a plugin, and a `SKILL.md` inside a submodule is none of those.

Treat it as a side effect of cloning, the way you treat this repository's `LICENSE` ending
up in your project. The skills come from the plugin.

### Why both the plugin and the submodule

They carry different things and neither replaces the other.

| | Carries | Loaded |
|---|---|---|
| Plugin | the skills | on demand, when a task calls for one |
| Submodule | the standards documents | into every session, through the `@` imports in your `CLAUDE.md` |

A skill is a procedure you want available and out of the way until it is needed. A
standard is a rule that has to be in front of Claude while it writes any code at all, and
pinned per project so that updating it shows up as a diff.

The skills refer to the documents by their `.standards/` paths, so a project that installs
the plugin without adding the submodule ends up with skills pointing at files that are not
there. Do both.

---

## Guidelines are the rules, tooling is the enforcement

A document that says "handlers are `sealed`" gets ignored sooner or later. A test that
fails does not.

So: these are **guidelines** that explain what we do and why. The actual enforcement
comes from tooling:

- **architecture tests** with NetArchTest for layer rules and conventions
- **analyzers** and `.editorconfig` for code style
- **CI** that runs both on every pull request

See [`dotnet/testing.md`](dotnet/testing.md) and [`ops/ci-cd.md`](ops/ci-cd.md) for what
is in place today.

---

## Contributing

Ownership per folder is recorded in [`CODEOWNERS`](CODEOWNERS). Changes go through a pull
request, reviewed by the owner of that folder.

Two rules of thumb:

- What is still open, and the rules that have to be revisited when something changes, are
  collected in [`ROADMAP.md`](ROADMAP.md). Close an item by moving the answer into the
  document that owns it and deleting the line there.
- Something that applies in more than one project belongs here. Something that applies in
  one project belongs in that project's `CLAUDE.md`.
- Changing a skill reaches installed plugins only when `version` in
  [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json) goes up. Changing a standards
  document reaches a project when that project moves its submodule. Both are deliberate on
  purpose.
- Explain *why*, not only *what*. A rule without a reason does not get followed.

These standards were written for one team and are shared in case they are useful to
others. Nothing in here is universal advice. Where we made a choice, we wrote down the
reasoning, so you can decide whether it applies to you.

---

## License

[MIT](LICENSE). Take what is useful, adapt it to your own team, ship it. The only
condition is that the copyright notice travels with substantial copies.
