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
| `.github/workflows/ci.yml` | every pull request, and pushes to `develop` and `main` | build, test, check, drive the journeys |
| `.github/workflows/cd.yml` | a successful CI run | build the image, deploy to test |
| `.github/workflows/promote.yml` | a person, deliberately | deploy an existing image to acceptance or production |

Read the second row carefully. CD does **not** trigger on push. If it did, the two
workflows would run side by side and a failing test would not stop the deploy. Chapter 5
explains the wiring.

The branches those triggers refer to come from
[`../general/git-workflow.md`](../general/git-workflow.md): `develop` is what goes live
next and is the only branch that builds, `main` records what is live. Environments come
from [`environments.md`](environments.md).

Three files is the ceiling. Five workflows that call each other is something nobody reads
any more.

---

## 2. CI

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [develop, main]

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

  web:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: src/TodoApp.Web
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive

      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: npm
          cache-dependency-path: src/TodoApp.Web/package-lock.json

      - run: npm ci
      - run: npm run lint
      - run: npx tsc --noEmit
      - run: npm test
      - run: npm run build
      - run: npm audit --audit-level=high
```

Two jobs, because every project has two halves that build with different tools, see
[`../dotnet/solution-layout.md`](../dotnet/solution-layout.md). They share nothing, so they
run side by side and the slower one sets the time. Both are required checks.

The `web` job follows the React standard, which is still a draft. The shape of the job is
settled, a job per half with `working-directory` on the frontend folder. The exact scripts
are whatever the project's `package.json` defines.

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

## 4. End-to-end tests

A separate job, so the fast checks still report in under a minute and the browser run
comes in behind them. It drives the compose stack from
[`containers.md`](containers.md), which means no deployed environment, no shared state
and no secrets.

The journeys live in `tests/<Product>.E2ETests/`, with their own `package.json`, because a
journey belongs to the product and not to the frontend. See
[`../dotnet/solution-layout.md`](../dotnet/solution-layout.md).

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - uses: actions/setup-node@v4
        with:
          node-version: '24'

      # The database first, so migrations run before anything connects.
      - name: Start the database
        run: docker compose up -d --wait db

      - name: Apply migrations
        run: |
          dotnet tool install --global dotnet-ef --version 10.*
          dotnet ef database update \
            --project src/TodoApp.Infrastructure \
            --startup-project src/TodoApp.Api
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Port=5432;Database=todoapp;Username=todoapp;Password=localdev"

      - name: Start the application
        run: docker compose up -d --wait

      - name: Install Playwright
        working-directory: tests/TodoApp.E2ETests
        run: |
          npm ci
          npx playwright install --with-deps chromium

      - name: Run the journeys
        working-directory: tests/TodoApp.E2ETests
        env:
          BASE_URL: http://localhost:5001
        run: npx playwright test

      - name: Application logs on failure
        if: failure()
        run: docker compose logs --no-color

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: tests/TodoApp.E2ETests/playwright-report/
          retention-days: 7

      - name: Tear down
        if: always()
        run: docker compose down -v
```

Why it is shaped like this:

- **`--wait` honors the health checks** from the compose file, so nothing starts talking
  to Postgres before it accepts connections.
- **Database, then migrations, then the application.** The same order as a real deploy,
  see [`database.md`](database.md). Bringing the whole stack up at once means the
  application starts against a schema that does not exist yet.
- **Only Chromium.** Three browsers triple the runtime and almost never catch a third
  thing.
- **The compose logs on failure.** Without them a failed journey is a timeout and nothing
  else, and you end up guessing at whether the application even started.
- **The report as an artifact.** Playwright writes traces and screenshots; they are the
  difference between fixing the test today and reproducing it for an afternoon.
- **`down -v`** so the volume goes too and the next run starts from nothing.

Match the Node version to what the project actually uses.

### Still to be decided: authentication in the stack

The compose stack has no identity provider in it, so the journeys have nothing to log in
against. Three ways out, and none of them is free:

1. A test-only authentication scheme in the application, switched on by configuration.
   Cheapest, and it puts a bypass in production code. If you take this route, it has to
   be impossible to enable outside the test configuration, and an architecture test
   should prove that.
2. A fake OIDC provider as a fourth container. Clean, and it is a container to keep
   working.
3. Point the stack at a real tenant at the provider. Realistic, and it makes CI depend on
   an external service and on secrets.

Decide this per project and write down which one and why. Until it is decided, the
journeys can only cover what an anonymous visitor sees.

### Still to be decided: the frontend in the stack

The job above points `BASE_URL` at the API, on the host port from the compose file. Nothing in the compose
stack serves the React build, so as written there is no page for the browser to open.
[`containers.md`](containers.md) chapter 7 says the frontend does not get a production
container, which is right for production and leaves this job without a frontend.

The options are to build the frontend in the job and serve the output with a static file
server, to let Playwright start the dev server through its `webServer` setting, or to give
compose a frontend service that exists only for this purpose. The first is closest to what
production serves. Not decided yet.

---

## 5. CD

The deploy hangs off the CI run, not off the push. A green run on `develop` goes to test
on its own. A green run on `main` is a hotfix and goes to production, because that is the
only reason anything lands on `main` from outside this pipeline.

> **Still to be decided: deploying the frontend.** Everything below builds and ships the
> API image. The frontend's build output is static files, and there is no job yet that
> publishes them, no target they go to and no way to roll them back. The principle from
> the top of this document still holds: build once, in CI, and promote that same output
> rather than rebuilding per environment. That has a consequence worth knowing before you
> start: a Vite build bakes its environment variables in at build time, so the API URL
> cannot be one of them if the same output has to serve every environment.

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
  BRANCH: ${{ github.event.workflow_run.head_branch }}

jobs:
  image:
    if: >-
      github.event.workflow_run.conclusion == 'success' &&
      (github.event.workflow_run.head_branch == 'develop' ||
       github.event.workflow_run.head_branch == 'main')
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

  test:
    needs: image
    if: github.event.workflow_run.head_branch == 'develop'
    uses: ./.github/workflows/deploy.yml
    with:
      sha: ${{ github.event.workflow_run.head_sha }}
      environment: test
    secrets: inherit

  production:
    needs: image
    if: github.event.workflow_run.head_branch == 'main'
    uses: ./.github/workflows/deploy.yml
    with:
      sha: ${{ github.event.workflow_run.head_sha }}
      environment: production
    secrets: inherit
```

The deploying itself lives in a reusable `deploy.yml`, called here and from the promotion
workflow, so that migrations, the rollout and the smoke test are written once:

```yaml
name: Deploy

on:
  workflow_call:
    inputs:
      sha:          { required: true, type: string }
      environment:  { required: true, type: string }

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ inputs.sha }}
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

      # Roll out ghcr.io/<repo>:${{ inputs.sha }} here.
      # How depends on where it runs.

      - name: The deployed version answers
        run: |
          for attempt in $(seq 1 10); do
            if curl -fsS "${{ vars.PUBLIC_URL }}/health/ready"; then
              exit 0
            fi
            sleep 10
          done
          exit 1
```

The order is not optional: **migrations first, then the new version**. During a deploy old
and new run side by side for a moment, so the schema has to cope with both. Hence the
two-step approach for breaking changes from [`database.md`](database.md).

### Why the deploy does not trigger on push

Two workflows that both listen to `push` run independently. GitHub does not order them
and CD has no idea CI exists, so a red test does not stop anything. The broken version
goes live while the failure email is still being written. Triggering CD from the CI run
itself is what makes the tests a gate instead of a report.

Branch protection is the other half of this and does not replace it. Protection stops a
failing branch from being merged; the `workflow_run` gate stops a deploy when CI fails on
a protected branch anyway, which happens through a flaky test, a difference in the runner,
or somebody pushing past their own protection.

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
work. And the branch guard keeps CI runs from pull requests out of it, because those
carry the source branch as their head, not `develop` or `main`.

A workflow only triggers `workflow_run` once its file exists on the default branch, so
this wiring does nothing until it is merged. Test it by merging it, not by watching a PR.

### The smoke test

The gate stops a build that fails its tests. It does not notice a deploy that succeeded
into a broken environment: a missing secret, a migration that ran but left the
application unable to start, a health check that never goes ready.

One `curl` against `/health/ready` from [`containers.md`](containers.md), retried for a
minute or two, catches that class of failure while you are still looking at the run.

---

## 6. Promotion

Acceptance and production are deliberate acts. The same image that CI built for a commit
on `develop` is rolled out again, never rebuilt, so what serves customers is byte for byte
what passed the tests.

```yaml
name: Promote

on:
  workflow_dispatch:
    inputs:
      sha:
        description: The commit to promote. An image must already exist for it.
        required: true
      environment:
        description: Where to
        required: true
        type: choice
        options: [acceptance, production]

permissions:
  contents: write
  id-token: write

jobs:
  promote:
    uses: ./.github/workflows/deploy.yml
    with:
      sha: ${{ inputs.sha }}
      environment: ${{ inputs.environment }}
    secrets: inherit

  record:
    needs: promote
    if: inputs.environment == 'production'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Point main at what is live
        run: git push origin ${{ inputs.sha }}:main
```

`environment:` on the deploy job is what asks for approval, so a promotion to production
waits for whoever is listed on that GitHub Environment.

### Why main is fast-forwarded and not merged

That last step moves `main` to the commit that just went live, so `git log main` answers
what is running without anybody maintaining it.

It is a fast-forward on purpose. A merge commit would give `main` a SHA that no image was
ever built for, and then the tag on the running container points at a commit that is not
the one `main` says is live.

**When that push fails, do not force it.** A rejected fast-forward means `main` has a
commit that `develop` does not: a hotfix that was never merged back. Deploying over it
would ship the bug you already fixed. Merge `main` into `develop`, let CI run, and promote
again.

### Why this does not loop

Pushing to `main` triggers CI, which triggers CD, which would deploy again. It does not,
because a push made with the automatic `GITHUB_TOKEN` does not start a new workflow run.
That is a deliberate GitHub behavior to stop exactly this recursion.

It also means the opposite is worth knowing: swap that token for a personal access token
or an App token and the loop comes back.

---

## 7. Secrets

- No long-lived cloud keys in GitHub secrets. Use OIDC with a federated credential, so
  that a run gets a short-lived token. That is what `id-token: write` in the permissions
  is for.
- Secrets per environment via GitHub Environments, not all of them at repository level.
- The migration user is a different one from the application user. Only the deploy job
  knows the first.
- `permissions` is set explicitly and as tightly as possible in every workflow. The
  default is wider than you want.

---

## 8. What has to be green before you merge

Set these as required checks in the branch protection of `main`:

- build succeeds, for both the `build` and the `web` job
- all tests pass, in every category that applies to the change
- no vulnerable packages, in NuGet and in npm

Required checks are what makes the workflow a gate rather than a report. A workflow
without them runs, goes red, and changes nothing about what a person is allowed to do
next.

Two separate gates, and you want both:

| Gate | Stops | Set up as |
|---|---|---|
| Branch protection | merging a branch whose CI is red | required checks on `main` |
| `workflow_run` on CD | deploying when CI is red | the trigger in chapter 5 |

What has to be tested before any of this is worth running is in
[`../general/testing.md`](../general/testing.md).

---

## 9. Checklist: new repository

1. Copy `ci.yml`, `cd.yml`, `deploy.yml` and `promote.yml`, and adjust the project names.
2. `submodules: recursive` in every checkout, plus the guard step that proves it worked.
3. CD triggers on `workflow_run`, never on `push`, and every checkout and tag uses
   `head_sha`.
4. Branch protection on `main` and `develop` with the checks from chapter 8.
5. A GitHub Environment per deployed environment, with reviewers on acceptance and
   production and its own secrets. See [`environments.md`](environments.md).
6. OIDC set up for the deploy, no static keys.
7. `PUBLIC_URL` set per GitHub Environment, so the smoke test has somewhere to look.
8. Working alone? Branch protection still applies, and the admin bypass stays unused.
   See [`../general/git-workflow.md`](../general/git-workflow.md).
9. Watch the first deploy manually, including the migration step.
