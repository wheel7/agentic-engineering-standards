# Git workflow

Applies to every project, whatever the stack.

---

## 1. Branching

GitFlow without the ceremony. Four kinds of branch, and two of them are long-lived:

| Branch | Is | Branches from |
|---|---|---|
| `main` | what is live in production | nothing, it is the root |
| `develop` | what goes live next | - |
| `feature/*` | one change | `develop` |
| `hotfix/*` | a production fix that cannot wait for `develop` | `main` |

**No `release/*` branch** until you actually need one. It earns its keep only when a
release has to be stabilized while `develop` carries on with the next thing. If every
change flows straight through, the release branch is a second copy of `develop` that
somebody has to keep synchronized.

**Naming**: `feature/1234-short-description`, `hotfix/1234-short-description`. Ticket
number first, so the branch sorts and searches by it.

**A branch lives days, not weeks.** A feature branch that has been open long enough to
need a rebase against a moved `develop` was too big when it started. Split it: the part
that can land behind a switch goes first.

**Nobody pushes straight to `main` or `develop`.** Including you. Especially you, see
chapter 4.

### What the branches mean for the pipeline

Only `develop` builds. `main` is a record, not a trigger. That keeps the artifact the
thing that gets promoted, rather than the code, which is the trap when GitFlow meets
continuous delivery. The full wiring is in [`../ops/ci-cd.md`](../ops/ci-cd.md).

After a production deploy, `main` is fast-forwarded to the commit that was deployed. It
therefore always says what is live, without anybody having to remember. When that
fast-forward fails, there is a hotfix on `main` that was never merged back into
`develop`, and you were about to deploy over it.

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
| `feature/*` | `develop` | squash | one change, one commit, a history you can read |
| `hotfix/*` | `main` | squash | same |
| `develop` | `main` | fast-forward | keeps the SHA that was tested and deployed |

That last row is the one that matters. A merge commit there would give `main` a SHA that
no image was ever built for, and the tag on the deployed image would point somewhere else.

**Size**: if you cannot describe the change in two sentences, split it. A reviewer who
opens eight hundred lines does not review them, they skim them and approve.

**The template** lives in `.github/pull_request_template.md` and carries the test evidence
block from [`testing.md`](testing.md).

**Required checks** on `main` and `develop`, through branch protection:

- build succeeds
- all tests pass, in every category that applies
- no vulnerable packages

Branch protection is what turns those from a report into a gate. See
[`../ops/ci-cd.md`](../ops/ci-cd.md).

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

**Do not use the admin bypass.** GitHub will happily let you push past your own branch
protection, and the entire value of that protection is that it checks the things no
colleague is going to check. A rule you can wave through is a preference.

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

We do not give the application a version number. The image tag is the git SHA, and `main`
records what is live. That is enough to answer every question anybody actually asks, which
is what is running and what changed since.

Semantic versioning starts mattering the moment somebody else consumes what you build, so
a library or a published API gets one then, and not before.

---

## 7. Open questions

- [ ] What happens when a customer has to stay on an older version? That is the situation
      a `release/*` branch exists for, and the answer changes the branching model rather
      than extending it.
