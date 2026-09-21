# Git workflow

Applies to every project, whatever the stack.

---

## 1. Branching

One long-lived branch, and short branches that land on it through a pull request.

| Branch | Is | Branches from |
|---|---|---|
| `main` | what goes live next | nothing, it is the root |
| `feature/*` | one change | `main` |

| Tag | Is | Moved by |
|---|---|---|
| `production` | the commit that is live | the promotion workflow, and nothing else |

**Landing a change and going live are two decisions.** Merging a pull request says this
code is right and may be in `main`. Promoting a commit says this version may be live now.
Neither follows from the other: you can merge five changes and promote the last one, or
promote the one before it.

**Why not GitFlow.** An earlier version of this document had `develop` for what goes live
next and `main` for what is live, with the promotion workflow fast-forwarding `main`
after every production deploy. The first project that used it paid for the second branch
four times in its first week, and got nothing back for it:

- CD and the promotion workflow only exist for GitHub once their files are on the default
  branch. With `main` as the default and `main` only moving through the promotion
  workflow, the pipeline could not start at all.
- `Closes #12` only works on a merge into the default branch, and a new pull request aims
  at it, so both pointed at the branch nobody merges into.
- A rollback is a promotion of an older commit, and an older commit is not a fast-forward,
  so the step that recorded what is live would have failed exactly when it mattered.
- "Nobody pushes to `main`" and "the promotion workflow pushes to `main`" cannot both be
  true once the branch is protected.

A branch is the wrong tool for "which commit is live". That is a pointer to one commit that
has to be able to move backwards, which is what a tag is.

**No `release/*` and no `hotfix/*` branch** until you actually need one. A production fix
is a change like any other: a pull request into `main`, promoted as soon as it is merged.
That only stops working when `main` holds something that must not go live yet, and the
first answer to that is a switch, see below, not a branch. If it happens anyway, branch from
the `production` tag, fix it there, promote that commit, and bring the fix into `main`
with a pull request straight after.

**Naming**: `feature/1234-short-description`. Ticket number first, so the branch sorts and
searches by it.

**A branch lives days, not weeks.** A feature branch that has been open long enough to
need a rebase against a moved `main` was too big when it started. Split it: the part that
can land behind a switch goes first. With one branch this matters more, not less, because
everything in `main` is one promotion away from a customer.

**Nobody pushes straight to `main`.** Including you. Especially you, see chapter 4.

### What this means for the pipeline

Every green run on `main` produces an image, tagged with its commit. What gets promoted is
that image, never the code, so production runs byte for byte what passed the tests. The full
wiring is in [`../ops/ci-cd.md`](../ops/ci-cd.md).

After a production deploy the promotion workflow moves the `production` tag to the commit
that went live. `git log production` therefore answers what is live, and
`git log production..main` answers what is waiting, without anybody having to remember.

---

## 2. Commit messages

**English**, see [`language.md`](language.md).

- The subject line says what changed, in at most 72 characters.
- The body says why. What changed is visible in the diff; why it changed is not, and in
  six months that is the only part anybody needs.
- Reference the ticket in the pull request, not in every commit.

**We do not use Conventional Commits.** The payoff of `feat:` and `fix:` prefixes is
automated versioning and generated changelogs, and neither applies to an application that
is deployed continuously and that nobody consumes as an artifact. Adopt it the day you
publish something versioned, and not before, because a format nobody needs is a format
that gets policed.

Because we squash, the pull request title is what ends up in the history. Write it as the
commit message you want, not as a note to the reviewer.

---

## 3. Pull requests

**Every change goes through one**, including a one-line fix and including when you are
the only person on the project. The pull request is what makes CI run before the code
lands rather than after.

**Reviewers**: one, and never the author. Working alone is the documented exception, see
chapter 4.

**Merge strategy:**

| From | To | How | Why |
|---|---|---|---|
| `feature/*` | `main` | squash | one change, one commit, a history you can read |

One commit per change also means one image per change, and a history in which every line
is something you could promote.

**Make the repository enforce it.** In the repository settings, allow squash merging only,
with the pull request title as the commit title, and turn on deleting the branch after a
merge. With all three methods on, the wrong one is a single click away and the default
button is a merge commit.

**Size**: if you cannot describe the change in two sentences, split it. A reviewer who
opens eight hundred lines does not review them, they skim them and approve.

**The template** lives in `.github/pull_request_template.md` and carries the test evidence
block from [`testing.md`](testing.md).

**Required checks** on `main`, through branch protection:

- build succeeds
- all tests pass, in every category that applies
- no vulnerable packages

Branch protection is what turns those from a report into a gate. See
[`../ops/ci-cd.md`](../ops/ci-cd.md).

**Protection is a ruleset, not a classic branch protection rule.** Both do the job for a
branch. A ruleset is the one GitHub is building on, anyone who can read the repository can
see it, it can be switched off without being deleted, and it covers tags as well. It also
replaces the "include administrators" switch with a list of who may bypass it, and that
list is empty: nobody, the owner included.

The ruleset for `main` is a file, [`../templates/ruleset-main.json`](../templates/ruleset-main.json),
so a project applies it rather than clicking it together:

```bash
gh api -X POST repos/<owner>/<repo>/rulesets --input .standards/templates/ruleset-main.json
```

It requires a pull request, squash as the only merge method, and the `build`, `web` and
`e2e` checks from CI, and it blocks force pushes and deleting the branch. The number of
required approvals in the file is zero, which is the setting for working alone, chapter 4.
It becomes one the day a second developer arrives. Prove that it
took: a direct push to `main` has to come back with "push declined due to repository rule
violations".

On a private repository this needs a paid plan, GitHub Pro for a personal account or Team
for an organization. On a free plan GitHub refuses, and then the rule is kept by hand and
written down as a deviation in the project `CLAUDE.md`.

---

## 4. Working solo

Most of this document assumes a second person. When there is not one, the rule is not
that the process gets lighter. It is that the process has to become mechanical, because
every check that was going to be caught socially now is not going to be caught at all.

**What changes:**

- Required reviewers drops to zero. There is nobody to ask.
- You read your own diff in the pull request before merging it. Not the summary, the diff.
  That is the review, and it is the only one there is.

**What does not change:**

- Every change still goes through a pull request.
- Branch protection stays on, with the same required checks.
- The test evidence block still gets filled in, including the lines about what is not
  covered.

**Nobody bypasses it, and that includes you.** The bypass list of the ruleset stays empty.
The entire value of that protection is that it checks the things no colleague is going to
check, and a rule you can wave through is a preference. It holds for an agent working
with your credentials just the same, which is a reason to want it and not a side effect.

**If an agent wrote the change, you are the whole of the human oversight.** That is the
one place where solo genuinely raises the bar rather than lowering it. Read what the diff
does, not what the description says it does. See
[`testing.md`](testing.md).

### Revisit when a second developer arrives

| Then | Becomes |
|---|---|
| Required reviewers | one, and not the author |
| Self-approval on a pull request | off |
| Pull request size | something worth arguing about, because someone else pays for it |
| A convention that lived in your head | written down here or in the project `CLAUDE.md` |

Put a reminder on it. The moment a team forms is exactly the moment nobody has time to
notice that the rules were written for one person.

---

## 5. Submodule `.standards`

- Updating `.standards` happens in its own pull request, so that the change in the
  standards is visible in the diff. See the [README](../README.md).
- Do not mix a submodule update with functional changes in the same pull request.

---

## 6. Versions and tags

We do not give the application a version number. The image tag is the git SHA, and the
`production` tag records what is live. That is enough to answer every question anybody actually asks, which
is what is running and what changed since.

Semantic versioning starts mattering the moment somebody else consumes what you build, so
a library or a published API gets one then, and not before.

---

## 7. Open questions

- [ ] What happens when a customer has to stay on an older version? That is the situation
      a `release/*` branch exists for, and the answer changes the branching model rather
      than extending it.
