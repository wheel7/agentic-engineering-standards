# Git workflow

> **Still to be decided by the team.** The headings below give the structure; the
> TODOs are the decisions we still have to make.

Applies to every project, whatever the stack.

---

## 1. Branching

- TODO: which model? Trunk-based with short feature branches, or GitFlow with
  `develop`/`release` branches?
- TODO: settle branch naming, for example
  `feature/<ticket>-short-description`, `bugfix/...`, `hotfix/...`.
- TODO: how long may a branch stay open before we split it up?
- TODO: who may push straight to `main`, and under what conditions?

## 2. Commit messages

- TODO: do we use Conventional Commits (`feat:`, `fix:`, `chore:`)? If so, which
  types do we allow?
- Settled: commit messages and pull requests are English. See
  [`language.md`](language.md).
- TODO: do we reference the ticket number, and where - in the title or the body?
- TODO: maximum length of the title line.

## 3. Pull requests

- TODO: number of required reviewers.
- TODO: merge strategy: squash, merge commit, or rebase?
- Settled: build, tests and the vulnerable-package check. Set them as required checks
  on `main`. See [`testing.md`](testing.md) and
  [`../ops/ci-cd.md`](../ops/ci-cd.md).
- TODO: maximum size of a PR - when do we ask for a split?
- Settled: yes, and it carries the test evidence block from [`testing.md`](testing.md).
  What else goes in it is still open.
- TODO: set branch protection rules on `main`.

## 4. Submodule `.standards`

This much is already settled:

- Updating `.standards` happens in its own PR, so that the change in the standards is
  visible in the diff. See the [README](../README.md).
- Do not mix a submodule update with functional changes in the same PR.

## 5. Open questions

- [ ] How do we handle long-running releases and hotfixes on production?
- [ ] Tagging and version numbering.
