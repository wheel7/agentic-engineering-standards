# Containers

How we package .NET applications in a container and run them locally. The database that
goes with it is covered in [`database.md`](database.md); building and publishing images
in [`ci-cd.md`](ci-cd.md).

> Goal: a new colleague can clone the repo, run `aspire run` and has a working
> application with a database, a way to look in it, and the logs of everything in one place.
> Without an installation guide.

---

## 1. Dockerfile

One Dockerfile per entry point, next to the project it builds:
`src/TodoApp.Api/Dockerfile`. The build context is the repository root, because the build
needs all projects.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Only the project files first: this layer is reused as long as
# no package changes.
COPY ["src/TodoApp.Domain/TodoApp.Domain.csproj",                 "src/TodoApp.Domain/"]
COPY ["src/TodoApp.Application/TodoApp.Application.csproj",       "src/TodoApp.Application/"]
COPY ["src/TodoApp.Infrastructure/TodoApp.Infrastructure.csproj", "src/TodoApp.Infrastructure/"]
COPY ["src/TodoApp.ServiceDefaults/TodoApp.ServiceDefaults.csproj", "src/TodoApp.ServiceDefaults/"]
COPY ["src/TodoApp.Api/TodoApp.Api.csproj",                       "src/TodoApp.Api/"]
RUN dotnet restore "src/TodoApp.Api/TodoApp.Api.csproj"

COPY . .
RUN dotnet publish "src/TodoApp.Api/TodoApp.Api.csproj" \
    -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .

USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "TodoApp.Api.dll"]
```

Why it looks like this:

- **Two stages.** The SDK image is well over a gigabyte and does not belong on
  production. Only the publish output goes into the runtime image.
- **Csproj files first.** If you copy everything at once, `restore` runs again on every
  code change. This way only when a package changes.
- **`--no-restore` on publish.** Otherwise publish repeats the restore work.
- **Chiseled runtime image.** No shell, no package manager, much less to patch. Does not
  work if you want to debug inside the container; use `10.0-noble` without chiseled in
  that case.
- **`USER $APP_UID`.** Since .NET 8 the images have a non-root user built in. There is no
  reason at all to run as root.
- **Port 8080.** The default since .NET 8, precisely because a non-root process may not
  open a port below 1024. Do not override this.

Pin the image to the major version (`10.0`), not to `latest`. That way you do get patches
and no unexpected .NET upgrade.

---

## 2. .dockerignore

Required, in the repository root. Without this file, `bin`, `obj` and `node_modules` go
into the build context. That is not only slow: an `obj` folder from your own machine can
make the restore in the container fail with errors that make no sense at all.

```gitignore
**/bin/
**/obj/
**/node_modules/
**/.vs/
**/.vscode/
**/.idea/
**/TestResults/
**/*.user
.git/
.github/
**/appsettings.Development.json
**/.env
.certs/
Dockerfile*
docker-compose*.yml
```

---

## 3. Running locally

Two ways, for two purposes:

| | Is for | Runs |
|---|---|---|
| **Aspire**, `aspire run` | a developer working on it | Postgres with pgAdmin, the migrations, the API and the frontend dev server, from source, with the dashboard |
| **Compose**, `docker compose up` | the journeys, locally and in CI | Postgres and the API **image**, the way a deployed host runs it |

Both put the API on the same port, so run one or the other.

### With Aspire

The entry point is `src/<Product>.AppHost`, and its `AppHost.cs` says what runs and in what
order:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Not a secret, like the password in the compose file below. Fixed rather than generated,
// because the volume keeps the password it was created with.
var userName = builder.AddParameter("postgres-user", "todoapp");
var password = builder.AddParameter("postgres-password", "localdev", secret: true);

var postgres = builder.AddPostgres("postgres", userName, password)
    .WithImageTag("18")
    .WithVolume("todoapp-apphost-pgdata", "/var/lib/postgresql")
    .WithLifetime(ContainerLifetime.Persistent)
    .WithPgAdmin(pgAdmin => pgAdmin.WithLifetime(ContainerLifetime.Persistent));

var database = postgres.AddDatabase("todoapp");

var migrator = builder.AddProject<Projects.TodoApp_DbMigrator>("migrator")
    .WithReference(database, connectionName: "DefaultConnection")
    .WaitFor(database);

var api = builder.AddProject<Projects.TodoApp_Api>("api")
    .WithReference(database, connectionName: "DefaultConnection")
    .WaitForCompletion(migrator)
    .WithHttpHealthCheck("/health/ready");

builder.AddViteApp("web", "../TodoApp.Web")
    .WithNpm(installCommand: "ci")
    .WithEndpoint("http", endpoint =>
    {
        endpoint.Port = 5000;          // the project's frontend port, see "Local ports"
        endpoint.UriScheme = "https";
        endpoint.IsProxied = false;
    })
    .WaitFor(api);

builder.Build().Run();
```

What you get: the migrations run on every start before the API does, so a new migration
needs nothing extra; pgAdmin to look in the database, linked from the dashboard; and the
logs, traces and SQL of every process in one place. The projects behind it, `.AppHost`,
`.DbMigrator` and `.ServiceDefaults`, are in
[`../dotnet/solution-layout.md`](../dotnet/solution-layout.md) chapter 2.

Points to watch, every one of them found the hard way:

- **Not `WithDataVolume()` on Postgres 18.** It mounts `/var/lib/postgresql/data`, and from
  18 the image refuses to start with a mount there, see the compose file below. Mount the
  volume on `/var/lib/postgresql` with `WithVolume`, and pin the major with `WithImageTag`,
  because Aspire's default image is not the version production runs.
- **A fixed user and password.** Aspire generates a password when you do not give one and
  keeps it in the AppHost's user secrets. The volume keeps the password it was created with,
  so the day those secrets are gone, the database no longer lets anybody in.
- **`connectionName: "DefaultConnection"`**, so the API and the migrator read the same
  `ConnectionStrings:DefaultConnection` under Aspire as everywhere else.
- **The frontend endpoint is fixed, HTTPS and not proxied.** By default Aspire gives Vite a
  random port behind its own proxy, and the provider only redirects to the registered port.
  Unproxied, Aspire passes the port to Vite as `--port`, and Vite serves HTTPS itself with
  the certificate from "Local HTTPS" below.
- **`npm ci`, not the default install.** Aspire runs `npm install` on every start, which
  rewrites `package-lock.json` whenever the local npm differs from the one that wrote it.
- **Persistent containers keep running** after `aspire run` stops, so the next start is
  quick. They get a new host port on every start, so reach pgAdmin through the dashboard.
- **The Aspire CLI**, `dotnet tool install --global aspire.cli`, with
  `<AspireUseCliBundle>true</AspireUseCliBundle>` in the AppHost project and an
  `aspire.config.json` in the root that points at it. Without the bundle the build warns
  that features are missing.

The API calls `builder.AddServiceDefaults()` for the logs and traces, after its own logging
setup, because `ClearProviders()` would remove them again. It exports only when the AppHost
sets `OTEL_EXPORTER_OTLP_ENDPOINT`, so the image and the tests are unaffected.

### The compose stack

`docker-compose.yml` in the repository root:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_DB: todoapp
      POSTGRES_USER: todoapp
      POSTGRES_PASSWORD: localdev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todoapp -d todoapp"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    build:
      context: .
      dockerfile: src/TodoApp.Api/Dockerfile
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__DefaultConnection: "Host=db;Port=5432;Database=todoapp;Username=todoapp;Password=localdev"
      # HTTPS locally, see "Local HTTPS" below. Configuration, not the image.
      ASPNETCORE_URLS: "https://+:8443"
      ASPNETCORE_Kestrel__Certificates__Default__Path: /https/localhost.pem
      ASPNETCORE_Kestrel__Certificates__Default__KeyPath: /https/localhost.key
    volumes:
      - ./.certs:/https:ro
    ports:
      # The project's own API port on the host. See "Local ports" below.
      - "5001:8443"
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Points to watch:

- **Pin the Postgres major** to the same number as production. A minor difference is
  fine, a major one is not.
- **`condition: service_healthy`.** Without a healthcheck the API starts before Postgres
  accepts connections, and then it crashes on the first query.
- **Named volume** for the data. A bind mount to a Windows folder gives permission
  problems.
- **The volume goes on `/var/lib/postgresql`**, without `/data` behind it. From major 18
  the image keeps its data in a directory per major version underneath that path, and it
  refuses to start when it finds a mount on the old `/var/lib/postgresql/data`. Every
  compose example older than 18 has the old path, so this is the line that breaks when you
  copy one. On 17 and earlier it is the other way around.
- **The password above is not a secret.** It only applies to a throwaway database on your
  own machine and is in the repo on purpose. No other password belongs there;
  see [`../general/security.md`](../general/security.md).

This compose file does not run migrations. The journeys job runs them before it starts the
API, see [`ci-cd.md`](ci-cd.md); locally, `dotnet ef database update` against this database
does the same.

### Local ports

Every project picks two ports on `localhost`, one for the frontend and one for the API,
and keeps them. They are asked for during setup and recorded in the project `CLAUDE.md`.

The reason is the identity provider. Kinde and Entra ID only redirect to a callback URL
that was registered with them, and the port is part of that URL. So
`https://localhost:<frontend port>` is typed into the provider once, and from then on the
frontend has to be there. A dev server that quietly moves to the next free port gives you
an error page at the provider that does not mention ports at all.

The tool defaults are the wrong pick for a second reason. Every Vite project wants 5173 and
every container example maps 8080, so the second project you start on the same machine
either fails or, worse, gets a different port without saying so.

One number has to show up in several places, and they have to agree:

| Where | What |
|---|---|
| The frontend dev server | the frontend port, **strict**, so a taken port is an error and not a silent move |
| The `web` endpoint in `AppHost.cs` | the frontend port, unproxied |
| `launchSettings.json` of the API | the API port, for Aspire and `dotnet run` |
| `ports:` of the API in compose | the API port on the host side |
| The API URL the frontend is configured with | the API port |
| The CORS allowlist for development | the frontend origin, see [`../general/security.md`](../general/security.md) |
| The application registration at the provider | the frontend origin, as callback and as logout URL |

The API answers on the same port under Aspire, under `dotnet run` and in a container, so
the frontend never has to be pointed somewhere else depending on how you started the rest.
Only the host side of the mapping is the project's own.

The Aspire dashboard gets a fixed port too, in the AppHost's `launchSettings.json`, so it can
be bookmarked. Take the one below the frontend port: 5047, 5048 and 5049 for dashboard,
frontend and API. The dashboard is not registered anywhere, so nothing breaks when it moves.

This is about development only. A deployed environment has hostnames, see
[`environments.md`](environments.md).

### Local HTTPS

The frontend and the API both run on `https://localhost` in development, and there is no
plain HTTP listener next to it to fall back to.

The reason is that production is HTTPS, and a good part of what goes wrong around signing
in depends on the scheme: the callback URL registered at the provider, the origins on the
CORS allowlist, cookies marked `Secure`, and mixed content. With HTTP locally, every one of
those has a development variant that differs from production, and the difference shows up
on the day of the first deploy. An origin is scheme, host and port, so the scheme belongs in
the same table as the ports above.

**One certificate for both halves: the ASP.NET development certificate.** No mkcert, no
second certificate authority to trust, nothing to install that the .NET SDK did not bring.
Once per machine, from the repository root:

```bash
dotnet dev-certs https --trust
dotnet dev-certs https -ep .certs/localhost.pem --format Pem -np
```

The first line creates the certificate and has the machine trust it. The second exports it
as two PEM files, certificate and key, without a password.

| Who | Gets the certificate from |
|---|---|
| The API under Aspire or `dotnet run` | the certificate store, by itself. `launchSettings.json` only says `https://localhost:<port>` |
| The API in compose | `.certs`, mounted read-only, through the Kestrel settings in the compose file above |
| The frontend dev server and preview | the same two files in `.certs`, read in the bundler configuration |

What to get right:

- **`.certs/` holds a private key.** It is in `.gitignore` and in `.dockerignore`, it is per
  machine, and it is never copied anywhere. It is a key for `localhost` that only your own
  machine trusts, so it is not the kind of secret
  [`../general/security.md`](../general/security.md) is about, and it still does not belong
  in a repository.
- **HTTPS in compose is configuration, not the image.** `ASPNETCORE_URLS` and the
  certificate paths are set in the compose file. The image still listens on plain 8080, as
  chapter 1 says, because in a deployed environment the reverse proxy terminates TLS. The
  same image goes everywhere; only what is around it differs.
- **The frontend must build and test without the certificate.** Read the files only when a
  server starts, and when they are missing, stop with the two commands above in the
  message. The `web` job in CI has no certificate and should not need one.
- **On Linux and macOS, `chmod 644` both files.** `dev-certs` writes them readable by their
  owner only, the certificate as well as the key, and the container runs as a different
  user. The API then fails to start with an access denied on the `.pem`. On Windows the bind
  mount hides this, so it shows up for the first time in CI.
- **A CI runner makes its own certificate**, with the second command, and nothing there
  trusts it. The journeys run with `ignoreHTTPSErrors`; they are not about TLS. See
  [`ci-cd.md`](ci-cd.md).

---

## 4. Configuration

Everything that differs per environment comes in as an environment variable. Nested keys
are written with a double underscore, because a dot is not allowed everywhere:

```
ConnectionStrings__DefaultConnection
Logging__LogLevel__Default
```

`appsettings.json` contains only default values that apply everywhere. Do not bake
environment-specific files into the image: the same image goes to test and to production,
only the variables differ.

---

## 5. What a container expects from the application

- **Logs to stdout**, structured as JSON. Not to files: those are gone as soon as the
  container is gone, and nobody can reach them.
- **Health endpoints.** `/health/live` says whether the process is still alive,
  `/health/ready` whether it can handle traffic (including the database connection). An
  orchestrator restarts on the first and routes traffic based on the second. One combined
  endpoint leads to restarts because the database is briefly slow.
- **Shut down cleanly on SIGTERM.** ASP.NET Core does this itself, but running background
  tasks have to respect the `CancellationToken`. Otherwise a deploy cuts off requests.
- **No state in the container.** No uploads on disk, no sessions in memory without a
  shared store. A container can be replaced at any moment.

---

## 6. Images and tags

- Tag on the git sha: `ghcr.io/wheel7/todoapp:9f3c1ab`. That way you always know exactly
  which code is running.
- Do not use `latest` to deploy. As an extra tag next to the sha it is fine.
- One image per entry point. A `.Worker` next to an `.Api` is its own image, not a switch
  in the same one.
- The same image goes to every environment. Rebuilding per environment means you run
  something on production that you have not tested anywhere.

---

## 7. Frontend

A React application usually does not belong in a container here. The build output is a
folder with static files; you put those on a CDN or on static hosting. A container with
nginx in front of it is another part to patch without it solving anything.

Locally the frontend is the dev server, started by the AppHost, see chapter 3. The journeys
serve a production build, see [`ci-cd.md`](ci-cd.md).

---

## 8. Checklist

1. `Dockerfile` next to the entry point, build context is the root.
2. `.dockerignore` present and up to date.
3. Runtime image is chiseled and runs as non-root on port 8080.
4. `aspire run` gives a working application with a database, pgAdmin and the dashboard, on
   HTTPS, on the project's own ports. `docker compose up` gives the database and the API
   image, for the journeys.
5. Configuration comes from environment variables, not from baked-in files.
6. Logs go to stdout, health endpoints exist.
7. Image is tagged on the git sha.
