# Language

Applies to every project, whatever the stack.

Two different questions hide behind "which language do we write in". One is about the
words we use for technical machinery. The other is about the words we use for the
business. They have different answers, and running them together is how a codebase ends
up with `GetPolisByIdQuery`.

---

## 1. Technical vocabulary is English

Layer names, patterns, type suffixes, test names, variables, branch names. English, in
every project, whoever the customer is.

`CreateOrderCommand`, never `MaakBestellingCommand`.

This is not a preference. The framework, the libraries, the error messages and every
answer you will ever search for are English. A codebase that translates half of that
vocabulary makes every reader translate it back.

## 2. Domain concepts follow the business

Use the words the business actually uses. If the people paying for the software say
"polis" and "schademelding", then the class is `Polis` and the event is
`SchademeldingIngediend`.

Translating those to `Policy` and `ClaimSubmitted` looks tidier, and it puts a dictionary
between the code and the conversation the code came from. An analyst says
"schademelding", the developer thinks "that is a Claim", and that translation happens in
someone's head every single time. Eventually someone gets it wrong, usually in the one
place where the two concepts are not quite the same thing.

Keeping the business word is the entire point of a ubiquitous language.

The reverse holds as well. If the business speaks English, the domain is English, even in
a team that does not.

## 3. Never mix inside one concept

The two rules above meet in the middle, and that is where it gets ugly. A `Polis` entity
with a `PolicyDto` beside it is worse than either choice taken consistently.

Pick the domain word once and use it everywhere that concept shows up: entity, DTO,
repository, endpoint, database table, folder name.

The technical suffix stays English, so you get `PolisRepository` and `CreatePolisCommand`.
That reads oddly for about a day. After that it reads like the business talking.

## 4. Documentation

This repository is English, because it is public and the people who might get something
out of it are not all Dutch.

A project's own documentation is the team's call. Pick one language and stay there.
Half-translated documentation is worse than either option, because nobody can tell what
the norm is any more.

## 5. Commit messages and pull requests

English. They sit next to the code, they travel with the repository, and everyone who
clones it reads them.

## 6. Write the choice down

The domain language cannot be derived from the code, and a new team member will guess
wrong roughly half the time. Record it once, in the project's own `CLAUDE.md`. The
template has a place for it.

New projects get asked this during setup. See
[`../skills/project-setup/SKILL.md`](../skills/project-setup/SKILL.md).
