# Testing

Applies to every project, whatever the stack.

This document says which kinds of tests exist, which ones a change has to bring with it,
and what counts as proof that it was actually tested. How you write them is per stack:
[`../dotnet/testing.md`](../dotnet/testing.md) and
[`../react/testing.md`](../react/testing.md).

> A claim that the tests pass is not evidence. The pipeline run is evidence.

That distinction is the whole point of this document. It holds for people and it holds
harder for agents, because an agent produces a confident summary whether or not it ran
anything, and a summary written from intent reads exactly like one written from output.
You cannot tell them apart by reading, so do not try. Make the pipeline the arbiter and
ask the author for the things the pipeline cannot check.

---

## 1. The kinds

| Kind | Proves | Cost | Runs |
|---|---|---|---|
| Unit | one class does what it should, dependencies mocked | milliseconds | every pull request |
| Architecture | the layer rules and conventions still hold | milliseconds | every pull request |
| Integration | the real chain works, with a real database | seconds | every pull request |
| Component | a React component renders and behaves | milliseconds | every pull request |
| Contract | frontend and backend agree on the API shape | seconds | every pull request |
| End-to-end | a user journey works in a browser | minutes | every pull request |
| Smoke | the deployed version answers | seconds | after every deploy |

**Unit tests** are the bulk. A handler with constructor injection needs no host and no
database, so there is no excuse for not having them.

**Architecture tests** are the ones that keep the other documents true. A rule written in
prose gets ignored; a rule that fails the build does not.

**Integration tests** use Testcontainers with the same Postgres image as production. Not
the in-memory provider, which has no constraints and no real SQL, so it passes exactly
where production breaks.

**Contract tests** are not set up yet. The open questions are in
[`api-contracts.md`](api-contracts.md). Until they are answered, a change to a response
shape is caught by a person or not at all.

**End-to-end tests** are worth having and worth rationing. They are slow, they fail for
reasons unrelated to the change, and a suite of them that nobody trusts is worse than
none. Cover the journeys that cost money when they break.

**Smoke tests** run after the deploy and check that the thing that went live answers on
`/health/ready`. They catch the failures no earlier test can see: a missing secret, a
migration that ran but left the application unable to start.

---

## 2. What a change has to bring with it

| The change | Required |
|---|---|
| A business rule in Domain | a unit test on the rule, no mocks needed |
| A new or changed handler | a unit test on the result **and** on the interactions |
| A new or changed endpoint | an integration test that calls it |
| A schema change | an integration test running against the migrated schema |
| A new dependency between layers | the architecture tests stay green, or the change is wrong |
| A React component with state or logic | a component test |
| A bug fix | a test that fails without the fix |
| Formatting, comments, documentation | nothing |

### The bug-fix row is the one that matters

A fix without a regression test is not a fix, it is a change that happens to work today.
Write the test first, watch it fail, then fix it. If you wrote the fix first, revert it
locally and confirm the test goes red before you put it back. A test that passes both
with and without the fix is testing something else.

### Weakening a test is a change to the specification

Deleting a test, marking it skipped, or loosening an assertion so a change can land is
allowed, and it is never allowed silently. Say in the pull request which test changed and
why the old expectation was wrong. If you cannot explain why the old expectation was
wrong, the test was right and the change is not ready.

---

## 3. Evidence

Three things are asked of whoever produced the change, agent or person.

**Tests ship in the same change.** Not in a follow-up ticket, not "next sprint". A
promise to add a test is not a test, and the moment the change lands is the last moment
anybody remembers what it was supposed to do.

**Report what the command actually printed.** Not a paraphrase, not a conclusion. If the
tests could not be run at all, say that plainly instead of writing something that reads
like they were. An honest "I could not run the integration tests here" is useful. A
confident summary of a run that never happened costs a reviewer their trust in every
future summary.

**Name what is not covered.** Every change touches something no test exercises. Saying
which part is not an admission of failure, it is the single most useful line in the pull
request, because it tells the reviewer where to actually look.

### The block that goes in the pull request

Projects put this in `.github/pull_request_template.md`:

```markdown
## Test evidence

- Ran: `<the command>`
- Result: `<the summary line, pasted, not paraphrased>`
- Tests added or changed: `<path>` - `<what it asserts>`
- Not covered: `<what this change touches that no test exercises>`
- Could not verify: `<anything that could not be run here, and why>`
```

The last two lines are not optional and are not allowed to say "none". A change with
nothing uncovered and nothing unverified is a change whose author has not looked.

### The rule with teeth

A pull request that changes behavior and adds no test to the diff does not get approved.
That is checkable by eye, and a bot that flags a PR touching `src/` without touching
`tests/` catches the rest.

---

## 4. Are the tests worth anything

Two numbers, and only one of them is useful.

**Coverage over the whole project** is not a target. Set a global threshold and it gets
met by testing whatever is cheapest to cover, which is rarely what matters. It also says
only that a line ran, never that anything would have failed had the line been wrong.

**Coverage of the lines this change touched** is the useful one. It answers the only
question a reviewer has, which is whether this particular change arrived tested. Coverlet
produces the data and the check belongs in the pull request, not in a monthly report.

**Mutation testing** is the real answer to whether a suite means anything. It changes the
code on purpose and checks whether a test notices. A test that survives every mutation is
executing the code without asserting anything about it.

Use Stryker.NET over Domain and Application, where the rules live. It is slow, so run it
on a schedule rather than on every pull request, and treat a drop in the score the way
you would treat a failing test.

---

## 5. What the reviewer checks

1. The tests the description claims exist are in the diff.
2. For a bug fix, the author says they watched it fail first.
3. CI is green, and green on this commit rather than an earlier one.
4. The "not covered" line is honest enough to be useful.
5. No test was quietly deleted, skipped or loosened.

Points 3 and 5 are the ones an agent cannot help you with, because both are questions
about what is missing rather than about what is there.
