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
| `.github/workflows/cd.yml` | push to `main` | build image, run migrations, deploy |

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

```yaml
name: CD

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
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
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: image
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v5
        with:
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
```

The order is not optional: **migrations first, then the new version**. During a deploy old
and new run side by side for a moment, so the schema has to cope with both. Hence the
two-step approach for breaking changes from
[`database.md`](database.md).

`environment: production` gives you required reviewers in GitHub and separate secrets per
environment. Turn that on before anything is really live.

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
- all tests pass
- no vulnerable packages

This belongs with the open questions in [`../general/git-workflow.md`](../general/git-workflow.md)
about branch protection and required checks. As long as those are not filled in, the
workflow is present but blocks nothing.

---

## 7. Checklist: new repository

1. Copy `ci.yml` and `cd.yml` and adjust the project names.
2. `submodules: recursive` in every checkout, plus the guard step that proves it worked.
3. Branch protection on `main` with the checks from chapter 6.
4. Environment `production` with reviewers and its own secrets.
5. OIDC set up for the deploy, no static keys.
6. Watch the first deploy manually, including the migration step.
