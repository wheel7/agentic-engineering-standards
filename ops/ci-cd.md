# CI/CD: GitHub Actions

How we build, test and deploy. Connects to [`containers.md`](containers.md) for the image
and [`database.md`](database.md) for the migrations.

> Build once, deploy that same artifact to every environment.

Rebuilding per environment means the version on production is never exactly the version
you tested. So compiling happens once, and what travels after that is the image plus the
migration bundle.

The README names tooling as the place where conventions are really enforced. This document
is that place: a rule that is not in a workflow gets ignored sooner or later.

---

## 1. Layout

Two workflows, no more:

| File | Runs on | Does |
|---|---|---|
| `.github/workflows/ci.yml` | every pull request and push to `main` | build, test, check |
| `.github/workflows/cd.yml` | a successful CI run on `main` | build image, run migrations, deploy, smoke test |

Read that second row carefully. CD does **not** trigger on push. If it did, the two
workflows would run side by side and a failing test would not stop the deploy. Chapter 4
explains the wiring.

Only split it up further when something really gets added. Five workflows that call each
other is something nobody reads any more.

---

## 2. CI

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive

      - name: Standards present
        run: test -f .standards/README.md

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release --logger trx --results-directory TestResults

      - name: Vulnerable packages
        run: |
          dotnet list package --vulnerable --include-transitive > vuln.txt
          cat vuln.txt
          ! grep -q "has the following vulnerable packages" vuln.txt
```

Three things that easily go wrong:

- **`submodules: recursive` is required.** A submodule stores no files in the parent
  repository, only a URL and a commit SHA, and `actions/checkout` does not initialize
  submodules by default. So `.standards` ends up empty and nothing fails. That silence is
  the real problem: `dotnet build` does not read markdown, so the build stays green while
  an agent runs without any of the conventions. It only turns into a hard error the day
  this repo starts carrying something the build consumes, such as a shared
  `Directory.Build.props`. The guard step below turns it back into a loud failure.
  `true` is enough here; `recursive` only matters with nested submodules. If the
  submodule repo is private, the checkout also needs a token that can read both
  repositories, because the default `GITHUB_TOKEN` is scoped to the repository the
  workflow runs in.
- **`dotnet list package --vulnerable` returns exit code 0**, even when there are
  vulnerabilities. Without the `grep` behind it, that step is decoration. This immediately
  fills in one of the open questions in [`../general/security.md`](../general/security.md).
- **`concurrency` with `cancel-in-progress`.** Otherwise three builds of the same branch
  run side by side when you push several times in quick succession.

Pin actions to a major tag as above, or to a commit SHA if you want to be stricter. When
you create a workflow, check which major is current.

---

## 3. Integration tests

Use **Testcontainers** with the same Postgres image as in
[`containers.md`](containers.md). The test starts the database itself, so the workflow
needs no service container and it works exactly the same locally. Docker is present on
the GitHub runners.

This decides the open question in [`../dotnet/testing.md`](../dotnet/testing.md): no
in-memory provider. It has no constraints, no transactions and no SQL, so tests pass
there while production fails.

---

## 4. CD

The deploy hangs off the CI run, not off the push.

```yaml
name: CD

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]

permissions:
  contents: read
  packages: write
  id-token: write

env:
  # Never github.sha in this workflow. See the warning below.
  SHA: ${{ github.event.workflow_run.head_sha }}

jobs:
  image:
    if: >-
      github.event.workflow_run.conclusion == 'success' &&
      github.event.workflow_run.head_branch == 'main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ env.SHA }}
          submodules: recursive

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          file: src/TodoApp.Api/Dockerfile
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ env.SHA }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: image
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ env.SHA }}
          submodules: recursive

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Build migration bundle
        run: |
          dotnet ef migrations bundle \
            --project src/TodoApp.Infrastructure \
            --startup-project src/TodoApp.Api \
            --self-contained -r linux-x64 -o migrate

      - name: Run migrations
        env:
          CONNECTION: ${{ secrets.DB_CONNECTION_MIGRATIONS }}
        run: ./migrate --connection "$CONNECTION"

      # Deploy the new image after this. How depends on where it runs.

  smoke:
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: The deployed version answers
        run: |
          for attempt in $(seq 1 10); do
            if curl -fsS "${{ vars.PRODUCTION_URL }}/health/ready"; then
              exit 0
            fi
            sleep 10
          done
          exit 1
```

### Why the deploy does not trigger on push

Two workflows that both listen to `push` run independently. GitHub does not order them
and CD has no idea CI exists, so a red test does not stop anything. The broken version
goes live while the failure email is still being written. Triggering CD from the CI run
itself is what makes the tests a gate instead of a report.

Branch protection is the other half of this and does not replace it. Protection stops a
failing branch from being merged; the `workflow_run` gate stops a deploy when CI fails on
`main` anyway, which happens through a flaky test, a difference in the runner, or an
admin pushing straight to the branch.

### The trap in workflow_run

In a `workflow_run` event, `github.sha` and the default `actions/checkout` ref both point
at the head of the default branch **at the moment the event fires**, not at the commit CI
actually tested. Push twice in quick succession and you will build the second commit,
deploy it, and tag it with a green tick that belongs to the first one.

So, without exception:

- Every checkout gets `ref: ${{ env.SHA }}`.
- Every image tag uses the same value.
- Anything that reports which version went live uses it too.

Two more things that are easy to miss. `types: [completed]` fires on failure and
cancellation as well, so the `conclusion == 'success'` guard is what actually does the
work. And the `head_branch` guard keeps CI runs from pull requests out of it, because
those carry the PR branch as their head.

A workflow only triggers `workflow_run` once its file exists on the default branch, so
this wiring does nothing until it is merged. Test it by merging it, not by watching a PR.

### The simpler alternative

One workflow with a deploy job that has `needs:` on the test job gives you the same gate
with none of the sha subtleties, because there is only one run and one commit. The price
is that the two concerns live in one file and every pull request shows a skipped deploy
job.

Take that route if the `workflow_run` wiring ever turns out to be more machinery than it
earns. What is not an option is two workflows that both trigger on push.

### The smoke test

The gate stops a build that fails its tests. It does not notice a deploy that succeeded
into a broken environment: a missing secret, a migration that ran but left the
application unable to start, a health check that never goes ready.

One `curl` against `/health/ready` from [`containers.md`](containers.md), retried for a
minute or two, catches that class of failure while you are still looking at the run.

---

## 5. Secrets

- No long-lived cloud keys in GitHub secrets. Use OIDC with a federated credential, so
  that a run gets a short-lived token. That is what `id-token: write` in the permissions
  is for.
- Secrets per environment via GitHub Environments, not all of them at repository level.
- The migration user is a different one from the application user. Only the deploy job
  knows the first.
- `permissions` is set explicitly and as tightly as possible in every workflow. The
  default is wider than you want.

---

## 6. What has to be green before you merge

Set these as required checks in the branch protection of `main`:

- build succeeds
- all tests pass, in every category that applies to the change
- no vulnerable packages

Required checks are what makes the workflow a gate rather than a report. A workflow
without them runs, goes red, and changes nothing about what a person is allowed to do
next.

Two separate gates, and you want both:

| Gate | Stops | Set up as |
|---|---|---|
| Branch protection | merging a branch whose CI is red | required checks on `main` |
| `workflow_run` on CD | deploying when CI on `main` is red | the trigger in chapter 4 |

What has to be tested before any of this is worth running is in
[`../general/testing.md`](../general/testing.md).

---

## 7. Checklist: new repository

1. Copy `ci.yml` and `cd.yml` and adjust the project names.
2. `submodules: recursive` in every checkout, plus the guard step that proves it worked.
3. CD triggers on `workflow_run`, never on `push`, and every checkout and tag uses
   `head_sha`.
4. Branch protection on `main` with the checks from chapter 6.
5. Environment `production` with reviewers and its own secrets.
6. OIDC set up for the deploy, no static keys.
7. `PRODUCTION_URL` set as a repository variable, so the smoke test has somewhere to look.
8. Watch the first deploy manually, including the migration step.
