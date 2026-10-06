# Language

Applies to every project, whatever the stack.

Two different questions hide behind "which language do we write in". One is about the
words we use for technical machinery. The other is about the words we use for the
business. They have different answers, and running them together is how a codebase ends
up with `GetPolisByIdQuery` next to a `PolicyDto`.

---

## 1. Technical vocabulary is English

Layer names, patterns, type suffixes, test names, variables, branch names. English, in
every project, whoever the customer is.

`CreateOrderCommand`, never `MaakBestellingCommand`.

This is not a preference. The framework, the libraries, the error messages and every
answer you will ever search for are English. A codebase that translates half of that
vocabulary makes every reader translate it back.

## 2. Domain concepts: a choice per project, made at the start

The words for the business itself, the things the users talk about, are a choice. There
are two valid answers, and every project picks one of them when it starts. The setup
asks it explicitly; see [`../skills/project-setup/SKILL.md`](../skills/project-setup/SKILL.md).

### Option A: the language of the users

Use the words the business actually uses. If the people paying for the software say
"polis" and "schademelding", then the class is `Polis` and the event is
`SchademeldingIngediend`.

Translating those to `Policy` and `ClaimSubmitted` puts a dictionary between the code and
the conversation the code came from. An analyst says "schademelding", the developer
thinks "that is a Claim", and that translation happens in someone's head every single
time. Eventually someone gets it wrong, usually in the one place where the two concepts
are not quite the same thing. Keeping the business word is the point of a ubiquitous
language.

Choose this when the domain is rich in words that have no clean English equivalent, or
when the people who describe the requirements will read the code, the tests or the
database.

### Option B: English

The domain concepts are English as well: `Member`, `Location`, `Invoice`, and the event is
`MembershipCancelled`.

Choose this when the project continues an existing application or data model that is
already English, so the old and the new system use the same words. Also when the team or
the code may cross a language border, or when the business words translate one-to-one
without losing meaning.

The cost is the dictionary from option A. Pay it once: the project's `CLAUDE.md` holds a
glossary from the users' word to the code word (`lid` -> `Member`, `ligplaats` ->
`Location`), so nobody has to guess the translation and nobody picks a second one.

### Either way

What users read stays in their language: screen texts, validation and error messages,
e-mails. That is text, not domain vocabulary, and it does not depend on this choice.

## 3. Never mix inside one concept

The rules above meet in the middle, and that is where it gets ugly. A `Polis` entity with
a `PolicyDto` beside it is worse than either choice taken consistently.

Pick the domain word once and use it everywhere that concept shows up: entity, DTO,
repository, API endpoint, frontend route, database table, folder name.

The technical suffix stays English, so with option A you get `PolisRepository` and
`CreatePolisCommand`. That reads oddly for about a day. After that it reads like the
business talking.

## 4. URLs and routes

A route is code, not text. It follows the same two rules as everything else:

- **Resource segments use the domain word**, in the language chosen in chapter 2,
  kebab-case and plural for a collection: `/members`, `/invoice-runs`, or with option A
  `/polissen`.
- **Action segments are technical vocabulary, so always English**: `new`, `edit`, and an
  English verb for a domain action (`assign`, `cancel`, `invite`, `send`). Never `nieuw`,
  `bewerken` or `toevoegen`, whatever the domain language is.
- **Parameters are named after the concept**: `:memberId`, `:polisId`.
- **Navigation grouping stays out of the URL.** A sidebar section called "Settings" is a
  UI concern, so it is `/member-types`, not `/settings/member-types`.

So with option B a screen is `/app/members/:memberId/edit`; with option A it is
`/app/polissen/:polisId/edit`. API endpoints follow the same rules.

The one exception is public content pages meant to be found by a search engine, such
as a marketing page. Their slug is content, written for the reader, so it is in the
reader's language. That never applies to anything behind a login.

## 5. Documentation

This repository is English, because it is public and the people who might get something
out of it are not all Dutch.

A project's own documentation is the team's call. Pick one language and stay there.
Half-translated documentation is worse than either option, because nobody can tell what
the norm is any more.

## 6. Commit messages and pull requests

English. They sit next to the code, they travel with the repository, and everyone who
clones it reads them.

## 7. Write the choice down

The domain language cannot be derived from the code, and a new team member will guess
wrong roughly half the time. Record it once, in the project's own `CLAUDE.md`, together
with the agreed domain terms and, for option B, the glossary. The template has a place
for it.

New projects get asked this during setup. See
[`../skills/project-setup/SKILL.md`](../skills/project-setup/SKILL.md).
