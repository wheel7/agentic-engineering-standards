# Containers

How we package .NET applications in a container and run them locally. The database that
goes with it is covered in [`database.md`](database.md); building and publishing images
in [`ci-cd.md`](ci-cd.md).

> Goal: a new colleague can clone the repo, run `docker compose up` and has a working
> application with a database. Without an installation guide.

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
Dockerfile*
docker-compose*.yml
```

---

## 3. Running locally with compose

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
      - pgdata:/var/lib/postgresql/data
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
    ports:
      - "8080:8080"
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
- **The password above is not a secret.** It only applies to a throwaway database on your
  own machine and is in the repo on purpose. No other password belongs there;
  see [`../general/security.md`](../general/security.md).

This compose file does not run migrations. Do that yourself locally, see
[`database.md`](database.md).

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

If the frontend does run along in compose for local development, then with the dev server
and a bind mount, not with a production build.

---

## 8. Checklist

1. `Dockerfile` next to the entry point, build context is the root.
2. `.dockerignore` present and up to date.
3. Runtime image is chiseled and runs as non-root on port 8080.
4. `docker compose up` gives a working application with a database.
5. Configuration comes from environment variables, not from baked-in files.
6. Logs go to stdout, health endpoints exist.
7. Image is tagged on the git sha.
