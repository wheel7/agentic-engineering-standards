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

Tables and columns are **snake_case**: `created_date`, not `CreatedDate`.

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

### Date in a name says nothing about the type

We name date and time columns with a `Date` suffix: `created_date`, `invoice_date`,
`subscription_end_date`. That reads the way the business talks, which is the rule in
[`../general/language.md`](../general/language.md), and one suffix everywhere beats two
competing ones.

The price is that the name no longer tells you which of the first two rows of the table
above you are looking at. So choose the type deliberately, on what the business actually
means:

- **The business thinks in days.** A birth date, an invoice date, a contract start date.
  Column type `date`, property `DateOnly`. There is no time of day to get wrong.
- **The business means a moment.** The four audit columns, a payment timestamp, when a
  message was delivered. Column type `timestamptz`, property `DateTime` in UTC.

Getting this backwards is the classic version of this bug. Store a field the business
reads as a day in a `timestamptz` and the value becomes midnight UTC. A subscription that
should run to the end of the seventeenth then expires at two in the morning Amsterdam
time, on a day the customer thinks they still have. Nothing in the column name hints at
it, and the report that finds it will be a support ticket.

When in doubt, ask whether anyone would ever care about the time of day in that field. If
not, it is a `date`.

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

Every table carries the same four audit columns: `created_by`, `created_date`,
`modified_by` and `modified_date`. Section 5 describes them, because what goes in them
depends on the user model in section 4.

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

## 4. Users and external identity

Authentication happens at an external provider. Kinde and Entra ID are both fine; which
one a project uses is recorded in its own `CLAUDE.md`. We never store passwords, and we
never treat the provider as the owner of our user records.

Two tables carry this:

- **`users`** holds our own user record, keyed by our own UUID. That id is the only user
  identifier that appears anywhere else in the database.
- **`user_identities`** holds one row per external login: the provider, the subject that
  provider issued, and a foreign key to `users`.

```csharp
public sealed class User : IAuditableEntity
{
    public Guid Id { get; private set; }
    public string Email { get; private set; } = string.Empty;
    public string DisplayName { get; private set; } = string.Empty;

    // Plus the four audit properties from IAuditableEntity, see section 5.
}

public sealed class UserIdentity
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public IdentityProvider Provider { get; private set; }
    public string Subject { get; private set; } = string.Empty;
}
```

Put a unique constraint on `(provider, subject)`. Two rows claiming the same external
login is a bug you want the database to catch, not one you hear about through a support
ticket.

### Why the split

One person can end up with more than one external identity. They sign in with Entra at
work and with Kinde on a customer portal, or the company moves from one provider to the
other and both have to keep working during the migration. A single subject column on
`users` forces a rewrite the first time that happens.

The subject is also not yours. The provider issues it, the provider can change it, and you
can be told to switch providers altogether. Everything else in the database points at
`users.id`, which you control and which never changes.

### Details that bite

- **The table is `users`, not `user`.** Table names are plural anyway, see section 2, and
  `user` is a reserved word in PostgreSQL. A table called `user` has to be quoted in every
  hand-written query for the rest of its life.
- **The subject is an opaque string.** Store it as `text`. Do not parse it, do not assume
  it is an email address, do not assume it is a UUID. Entra hands out an object id, Kinde
  hands out something else, and neither is your business.
- **Store the provider as a string**, following the enum rule in section 3. A number in
  that column tells the next reader nothing.
- **Never use the subject as a foreign key.** If a column points at a user, it holds
  `users.id`.

### The system user

Not every row is created by someone who logged in. Migrations seed data, background jobs
run at night, imports run from a pipeline. Those writes still need a `created_by`.

Seed one fixed row in `users` for that, with a UUID written into the migration rather than
generated. The created columns then never have to be nullable, and no reader ever has to
handle "nobody" as a separate case.

---

## 5. Auditing

Every table carries the same four columns:

| Column | Type | Holds |
|---|---|---|
| `created_by` | `uuid` | The `users.id` of whoever inserted the row |
| `created_date` | `timestamptz` | When it was inserted, in UTC |
| `modified_by` | `uuid`, nullable | Who last changed it |
| `modified_date` | `timestamptz`, nullable | When it was last changed, in UTC |

No exceptions, lookup tables included. "Who put this value here" is exactly the question
you end up asking about a lookup table that has quietly acquired a wrong row.

The modified columns stay null until the row is actually changed. Copying the created
values into them on insert looks tidier and throws away the one thing they tell you, which
is whether anything has touched the row since. Use
`coalesce(modified_date, created_date)` when you need to sort on a single column.

A note on the names: `created_date` holds a full timestamp, not a date. The UTC rule from
section 3 still applies to it. Because the interceptor below is the only thing that ever
writes these columns, there is exactly one place where that can go wrong.

### The entity interface

`IAuditableEntity` lives in the domain, and every entity implements it.

```csharp
namespace TodoApp.Domain.Auditing;

public interface IAuditableEntity
{
    Guid CreatedBy { get; set; }
    DateTime CreatedDate { get; set; }
    Guid? ModifiedBy { get; set; }
    DateTime? ModifiedDate { get; set; }
}
```

These four are the one documented exception to the private-setter rule in
[`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md). Audit values are not domain
behavior and no domain method ever sets them; infrastructure writes them, so it needs a
way in.

### Who is writing

The application layer declares what it needs, the same way it does for repositories:

```csharp
namespace TodoApp.Application.Identity;

public interface ICurrentUser
{
    Guid UserId { get; }
}
```

The implementation in Infrastructure reads the subject from the claims on the incoming
token and resolves it to a `users.id` through `user_identities`. Cache that lookup, or
every save costs an extra query. With no authenticated principal it returns the system
user.

### The interceptor

Handlers never set audit values by hand. An EF Core interceptor does it, so the rule
cannot be forgotten in the next handler somebody writes.

```csharp
public sealed class AuditingInterceptor : SaveChangesInterceptor
{
    private readonly ICurrentUser _currentUser;
    private readonly TimeProvider _timeProvider;

    public AuditingInterceptor(ICurrentUser currentUser, TimeProvider timeProvider)
    {
        _currentUser = currentUser;
        _timeProvider = timeProvider;
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is null)
            return base.SavingChangesAsync(eventData, result, cancellationToken);

        var userId = _currentUser.UserId;
        var now = _timeProvider.GetUtcNow().UtcDateTime;

        foreach (var entry in eventData.Context.ChangeTracker.Entries<IAuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedBy = userId;
                entry.Entity.CreatedDate = now;
            }
            else if (entry.State == EntityState.Modified)
            {
                entry.Entity.ModifiedBy = userId;
                entry.Entity.ModifiedDate = now;
            }
        }

        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}
```

`TimeProvider` rather than `DateTime.UtcNow`, so a test can pin the clock and assert on
the exact value instead of asserting that something happened recently.

Register it alongside the context:

```csharp
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<AuditingInterceptor>();

builder.Services.AddDbContext<TodoDbContext>((sp, options) =>
    options
        .UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
        .UseSnakeCaseNamingConvention()
        .AddInterceptors(sp.GetRequiredService<AuditingInterceptor>()));
```

### The DTO interface

A DTO that represents an auditable entity implements `IAuditableDto`, carrying the same
four values with the `init` properties DTOs use everywhere else.

```csharp
namespace TodoApp.Application.Auditing;

public interface IAuditableDto
{
    Guid CreatedBy { get; init; }
    DateTime CreatedDate { get; init; }
    Guid? ModifiedBy { get; init; }
    DateTime? ModifiedDate { get; init; }
}
```

Returning `created_by` as a raw id tells the client which user ids exist. Inside an
authenticated application that is normally fine. If the client needs a name to show, add a
field for it rather than replacing the id, so the interface stays the same everywhere.

---

## 6. Migrations

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

## 7. Access and permissions

- The application runs as a user that may only read and write data, not change the
  schema. Migrations run under a separate user.
- Connection strings are never in the repo. Locally in user-secrets or in the compose
  file, outside of that in the secret store of the environment. See
  [`../general/security.md`](../general/security.md).
- Developers do not work on the production database. If you need to take a look, do it on
  a restore.

---

## 8. Backups and environments

Every environment gets its own Postgres container, and whether the data survives differs
per environment. That is in [`environments.md`](environments.md), together with the rule
that a restore into acceptance is anonymized before anyone can look at it.

A backup that has never been restored is an assumption, not a backup. Record per project:

- How often, and how long it is kept.
- Where the copy lives, and that it lives somewhere other than the database itself.
- When the restore was last tested, and by whom.

---

## 9. Checklist: new entity

1. Snake_case follows automatically; do not set column names by hand.
2. Key via `Guid.CreateVersion7()`.
3. The entity implements `IAuditableEntity`, so the four audit columns come with it.
4. A day the business reads as a day is `date`; a moment is `timestamptz` in UTC.
5. Amounts as `numeric(19,4)`.
6. `HasMaxLength` on text fields that have a limit.
7. Concurrency token only where concurrent changes really occur.
8. The DTO implements `IAuditableDto` if it exposes the entity.
9. One migration, with a descriptive name.
10. Does the change break the running version? Split it into an expand step and a contract
    step.
