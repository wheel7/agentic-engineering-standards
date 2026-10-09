---
name: aspire-run
description: Starts a project locally with `aspire run` in the background and reports when it actually works - API ready, frontend up, dashboard link - or why it stopped. Use this whenever the developer asks to start, restart or stop the application locally ("start aspire", "run the app", "restart it with my changes", "stop aspire") in a project laid out by these standards, with an AppHost and aspire.config.json. Do not use it for the compose stack the journeys run against.
---

# Running the project with Aspire

Follows `@.standards/ops/containers.md` chapter 3. That chapter says what the AppHost runs;
this skill is how an agent starts it for the developer and proves that it came up, because
`aspire run` started from an agent fails in ways that look like success.

> `aspire run` exiting with code 0 is not a start. Neither is an AppHost that printed its
> dashboard link. The application runs when `/health/ready` answers 200 and the frontend
> answers on its port.

With the argument `stop`, go straight to "Stopping". With `restart`, stop first, then start.

## 1. Read the project

From the project `CLAUDE.md`, "Running locally": the dashboard, frontend and API ports. Never
assume the defaults from the standards' examples; every project has its own ports.

Then say, in one line each:

- **Which branch it runs from**: `git branch --show-current`, and whether it is behind
  `origin/main` (`git fetch` first). Running a feature branch is fine, but the developer
  must know, because a merged change is not in it and an unmerged one is.
- **Whether something already answers on the API port.** If so, it is an earlier run, of
  this project or of another one on the same port. Ask before stopping it, unless the
  developer asked for a restart.

## 2. Check what makes it fail silently

Each of these has cost an afternoon at least once.

1. **Docker runs**: `docker info`. Without it the database never starts and the API waits
   for it forever.
2. **The certificate exists**: `.certs/localhost.pem` and `.certs/localhost.key`. Without them
   the frontend refuses to start. The commands are in containers.md, "Local HTTPS".
3. **The CLI matches the AppHost.** Compare `aspire --version` with the SDK version in the
   AppHost project (`<Project Sdk="Aspire.AppHost.Sdk/x.y.z">`) and with the `Aspire.Hosting.*`
   package versions next to it. All three have to be the same version.

   Dependabot updates the hosting packages and not the SDK in the `Project` element, so after
   a NuGet group update they drift apart. The symptom is an AppHost that exits at once with
   code 0, and a CLI log that says "Unable to connect to backchannel". The fix:
   - the CLI, on the developer's machine: `dotnet tool update -g Aspire.Cli --version <x.y.z>`;
   - the SDK in the `Project` element, through a pull request like any other change.

## 3. Start it in the background

`aspire run` does not return, so it runs as a background task, from the repository root:

```bash
aspire run --non-interactive
```

On Windows the shell an agent uses often does not have the tool shim on its `PATH`. Then:

```bash
cmd //c "%USERPROFILE%\.dotnet\tools\aspire.cmd run --non-interactive"
```

Do not start it with `dotnet run` on the AppHost to get around a problem. That skips the CLI,
and the version mismatch above is then hidden until the next person runs `aspire run`.

## 4. Wait until it works, not until it printed something

Poll every two seconds, for at most three minutes. The first start pulls images and runs
`npm ci`, so it takes longer than the next ones:

```bash
curl -s -o /dev/null -w "%{http_code}" https://localhost:<API port>/health/ready
curl -s -o /dev/null -w "%{http_code}" https://localhost:<frontend port>/
```

Both 200 is up. Without `-k`: the development certificate is trusted, and a certificate error
is something to report, not to skip.

If the background task ends before that, it failed, whatever its exit code. Read the newest
log in `~/.aspire/logs/` and report the error lines (`[ERRO]`, `[CRIT]`, `exited with code`,
`backchannel`). Step 2.3 explains the most common one.

## 5. Report

- The branch it runs from.
- The dashboard link exactly as the output printed it. It carries a login token, so the
  bare port is not enough.
- The frontend and API URLs. A 401 on the API's `/` is the API working, see the project
  setup skill.
- That migrations ran: the migrator resource finishes before the API starts, so a ready API
  means they did. A new migration on the branch is applied by this start.

That a code change needs a restart: the AppHost runs the projects as they were built when it
started.

## Stopping

Stop the background task that runs `aspire run`, then check that the API port no longer
answers. The Postgres and pgAdmin containers keep running on purpose: they are persistent,
so the next start is quick and the data stays. Removing them, and the volume with the local
database, is a separate decision for the developer, never part of stopping.
