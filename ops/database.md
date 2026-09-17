# Database: PostgreSQL

Belongs with [`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md). That document
describes how the persistence layer sits in the code; this one describes the conventions
for the database itself and for migrations.

> PostgreSQL is the standard. If a project deviates from it, that is recorded with the
> reason in that project's `CLAUDE.md`.

Why Postgres: it runs the same everywhere (locally in a container, in CI, on Azure and on
your own server), no license is needed, and the managed variants are cheaper than the SQL
Server ones. Running it locally is covered in
[`containers.md`](containers.md).

---

## 1. Packages

```bash
dotnet add src/TodoApp.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL --version 10.0.*
dotnet add src/TodoApp.Infrastructure package EFCore.NamingConventions --version 10.0.*
```

The naming package follows the EF Core major. If it lags behind, set snake_case manually
in `OnModelCreating` for the time being and update the package once it is out.

Registration in `Program.cs`:

```csharp
builder.Services.AddDbContext<TodoDbContext>(options =>
    options
        .UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
        .UseSnakeCaseNamingConvention());
```

---

## 2. Naming in the database

Tables and columns are **snake_case**: `created_at_utc`, not `CreatedAtUtc`.

This is not a matter of taste. Postgres folds unquoted names to lower case, so a
PascalCase column has to be quoted in every query. Without this convention,
`select * from "Todos" where "IsCompleted" = false` becomes the norm and everyone who ever
looks in the database by hand complains. In C# everything stays PascalCase; the package
translates.

- Table names plural (`todos`), column names singular.
- No prefixes like `tbl_` or `fk_` in table names.
- We let EF Core name indexes and constraints, unless there is a good reason not to.

---

## 3. Types and columns

| Topic | Convention | Why |
|---|---|---|
| Date and time | `DateTime` with `Kind.Utc`, column type `timestamptz` | Npgsql throws an exception on a non-UTC `DateTime`, so the mistake surfaces right away instead of months later |
| Date without time | `DateOnly`, column type `date` | No midnight shift caused by time zones |
| Primary key | `Guid` via `Guid.CreateVersion7()` | See below |
| Amounts | `decimal`, column type `numeric(19,4)` | `float` and `double` round money wrong |
| Text | `text`, with `HasMaxLength` for validation | In Postgres, `varchar(n)` is not faster than `text`; the length is a rule, not an optimization |
| Enums | Store as string via `HasConversion<string>()` | Readable in the database, and renumbering the enum breaks nothing |

### Keys

Use `Guid.CreateVersion7()`, not `Guid.NewGuid()`. A version 7 GUID starts with a
timestamp and is therefore ascending. Random version 4 GUIDs write all over the index,
which bloats the index and makes inserts slower as the table grows. Same key, no extra
cost.

```csharp
Id = Guid.CreateVersion7();
```

An ascending `int` is allowed too, but not for anything that ends up in a URL or an API
response: that leaks the number of records and lets you guess other people's records.

### Standard columns

Every entity that records something that has to be explainable later gets at least
`CreatedAtUtc`. Add `ModifiedAtUtc` as soon as something can be changed.

Concurrent changes are caught with the system column `xmin`, the Postgres equivalent of
`rowversion`:

```csharp
todo.Property<uint>("xmin")
    .IsRowVersion()
    .HasColumnName("xmin");
```

Only do this where two users really can change the same row at the same time. Adding it
everywhere only produces conflicts that nobody handles.

### Searching on text

Postgres is case sensitive. Search on name or email with `ILIKE`, or put the column on a
non-deterministic collation. Choose per case and record it in the project; `ToLower()` in
a `Where` makes the index unusable.

---

## 4. Migrations

- **One migration per pull request.** More means the PR is too big.
- **Never edit a migration that has already been applied somewhere.** Not on test either.
  You correct it with a new migration.
- **You roll back by rolling forward.** A `Down` method that nobody has ever run is not a
  safety net. When something goes wrong on production, a new migration goes over it.
- **Naming**: describe the change, not the date. `AddTodoDueDate`, not
  `Update3`.

### Breaking changes in two steps

Renaming or dropping a column breaks the running version during a deploy, because old and
new run side by side for a moment. So split it:

1. **Expand.** Add the new column, leave the old one, and write to both.
   Deploy.
2. **Contract.** Remove the old column and the double writing. Deploy.

Two PRs, two releases. That is the price of deploying without downtime.

### Running migrations

**Not** at application startup. `Database.Migrate()` in `Program.cs` is tempting, but as
soon as two instances are running they start at the same time, and you have no control at
all over the moment. On top of that, the application then permanently has permissions to
change the schema.

Instead: as a separate step in the pipeline, before the new version goes live. See
[`ci-cd.md`](ci-cd.md). Build a migration bundle for it:

```bash
dotnet ef migrations bundle \
  --project src/TodoApp.Infrastructure \
  --startup-project src/TodoApp.Api \
  --self-contained -r linux-x64 \
  -o migrate
```

That produces a single executable that you run with a connection string. The runner needs
no SDK for it and you can see exactly what is being executed.

---

## 5. Access and permissions

- The application runs as a user that may only read and write data, not change the
  schema. Migrations run under a separate user.
- Connection strings are never in the repo. Locally in user-secrets or in the compose
  file, outside of that in the secret store of the environment. See
  [`../general/security.md`](../general/security.md).
- Developers do not work on the production database. If you need to take a look, do it on
  a restore.

---

## 6. Backups

A backup that has never been restored is an assumption, not a backup. Record per project:

- How often, and how long it is kept.
- Where the copy lives, and that it lives somewhere other than the database itself.
- When the restore was last tested, and by whom.

---

## 7. Checklist: new entity

1. Snake_case follows automatically; do not set column names by hand.
2. Key via `Guid.CreateVersion7()`.
3. Timestamps in UTC, as `timestamptz`.
4. Amounts as `numeric(19,4)`.
5. `HasMaxLength` on text fields that have a limit.
6. Concurrency token only where concurrent changes really occur.
7. One migration, with a descriptive name.
8. Does the change break the running version? Split it into an expand step and a contract step.
