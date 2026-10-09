# Domain-driven design on top of the architecture

An option, chosen when a project starts. [`ARCHITECTURE.md`](ARCHITECTURE.md) is the
default: entities with private setters and a few intent methods, and the rules mostly in the
handlers. That is the right size for a domain that is CRUD with some checks. This document
is for a domain where the rules are the point: invariants that span fields, state that moves
in steps, history that must stay consistent. The project setup asks which one a project
takes, see [`../skills/project-setup/SKILL.md`](../skills/project-setup/SKILL.md), and the
project's `CLAUDE.md` records it.

Everything in ARCHITECTURE.md still holds: the four layers, CQRS without MediatR, sealed
handlers with one `HandleAsync`, Minimal API endpoints per feature, ProblemDetails, the
database conventions. This document only says what changes. The first project built this way
is Havenmeester; when something here is unclear, its code is the example.

---

## 1. When to choose it

Choose DDD when one of these is true:

- A business rule has to hold whatever screen or job changes the data: a berth never has
  two holders, an invoice that went to the bookkeeping system never changes.
- Something moves through states, and the moves have rules: draft, reviewed, sent.
- Several records belong together and are only consistent together: an invoice and its
  lines, a berth and its history.
- The project is multi-tenant, and one tenant must never see another's data.

Stay with ARCHITECTURE.md when the domain is lists and forms with field validation. DDD costs
more code per feature, and that cost buys nothing when there are no rules to protect.

---

## 2. What changes

| ARCHITECTURE.md | With DDD |
|---|---|
| Entities with private setters and a few intent methods | **Aggregates** that guard their own rules. No public setters at all (an architecture test checks it), created through a constructor or a static factory method (`Member.Register(...)`), changed only through methods named after what the business does (`Location.Assign`, `Invoice.MarkSent`). |
| One repository per entity | **One repository per aggregate root.** A child entity (an invoice line, an assignment) is reached and changed through its root, never on its own. |
| Rules checked in the handler | **Rules in the domain.** The handler loads, calls a method, saves. Validation at the edge (data annotations on the command) stays for the shape of the input: required, length, range. |
| `ConflictException` from the handler | A rule that says no is a **`DomainRuleException(type, title, detail)`** thrown by the domain, which the API turns into a 409 with `type = https://<domain>/errors/<type>`. `ConflictException` stays for conflicts only the handler can see, such as a name already taken. |
| Ownership filtered in every repository method | **One query filter per tenant**, see chapter 4, so no handler or repository can forget it. |
| History, if any, written by hand | **An audit log written by the interceptor** for every change to a domain entity, see chapter 5. |

---

## 3. The domain

### Aggregates

```csharp
public sealed class Location : IAuditableEntity, IClubOwned
{
    private readonly List<LocationAssignment> _assignments = [];

    public Guid Id { get; private set; }
    public Guid ClubId { get; private set; }
    public string Name { get; private set; } = string.Empty;

    public IReadOnlyCollection<LocationAssignment> Assignments => _assignments.AsReadOnly();

    // Audit fields: public setters, set only by the interceptor.
    public Guid CreatedBy { get; set; }
    public DateTime CreatedDate { get; set; }
    public Guid? ModifiedBy { get; set; }
    public DateTime? ModifiedDate { get; set; }

    private Location() { } // for EF Core

    public Location(Guid clubId, string name, Guid locationTypeId) { ... }

    public LocationAssignment Assign(Guid memberId, DateOnly startDate, DateOnly today)
    {
        if (_assignments.Any(a => a.EndDate is null || a.EndDate >= startDate))
            throw new DomainRuleException("location-occupied", "Ligplaats bezet",
                $"Ligplaats {Name} is op {startDate:yyyy-MM-dd} nog in gebruik.");

        var assignment = new LocationAssignment(Id, memberId, startDate);
        _assignments.Add(assignment);
        return assignment;
    }
}
```

- **The collection is a private field** with a read-only view. EF Core maps it with
  `Navigation(...).HasField("_assignments").UsePropertyAccessMode(PropertyAccessMode.Field)`.
- **A child entity has an `internal` constructor and `internal` mutators**, so only its root
  creates or changes it.
- **The detail of an exception is the sentence a user reads**, in the users' language. The
  `type` is the stable code the frontend switches on.
- **An `ArgumentException`** is for input the edge should have stopped (an empty name, a
  negative amount). The API answers it with a 400 on the field, so it is never a 500.
- **"Today" is passed in**, never read inside the domain, so a rule about dates can be
  tested. The application gets it from `TimeProvider`, in the business's time zone, not UTC:
  a berth that ends today is free tomorrow where the harbour is.
- **A value that is computed is never stored**: a status derived from dates, a total from the
  lines. Store it only when computing it is too expensive, and then say why.

### Parameter records

When a method takes many fields, as "change the member's details" does, it takes one
record (`MemberDetails`) and checks it as a whole. The command builds it with a `ToDetails()`
method, so the handler stays one line.

### Domain services

A rule that needs data from more than one aggregate, such as the yearly invoice of a member
(contributions, the berth, earlier invoices), is a static class in the domain that takes
plain inputs and returns the result: `YearlyInvoiceCalculator.LinesFor(input)`. The handler
gathers the inputs. That keeps the rule in the domain and testable without a database.

### Unit tests

Every rule gets a test in `<Product>.Domain.UnitTests`, the project the default layout does
not have and this choice adds. The names say the rule:
`Assign_Throws_WhenTheBerthHasAHolder`.

---

## 4. Tenants: one filter, not a `Where` in every method

When the project is multi-tenant, every entity the tenant owns implements one interface in
the domain:

```csharp
public interface IClubOwned
{
    Guid ClubId { get; }
}
```

The `ClubId` is set in the constructor and never changes. The `DbContext` gets the current
user and puts one query filter on every entity that implements it:

```csharp
public HavenmeesterDbContext(DbContextOptions<HavenmeesterDbContext> options, ICurrentUser? currentUser = null)

// Without a current user (the migrator, an import) the context sees every tenant.
public bool SeesEveryClub => _currentUser is null;
public Guid? CurrentClubId => _currentUser?.ClubId;

private void ApplyClubFilters(ModelBuilder modelBuilder)
{
    var context = Expression.Constant(this);
    foreach (var entity in modelBuilder.Model.GetEntityTypes()
                 .Where(e => typeof(IClubOwned).IsAssignableFrom(e.ClrType)))
    {
        var row = Expression.Parameter(entity.ClrType, "row");
        var body = Expression.OrElse(
            Expression.Property(context, nameof(SeesEveryClub)),
            Expression.Equal(
                Expression.Convert(Expression.Property(row, nameof(IClubOwned.ClubId)), typeof(Guid?)),
                Expression.Property(context, nameof(CurrentClubId))));
        modelBuilder.Entity(entity.ClrType).HasQueryFilter(Expression.Lambda(body, row));
    }
}
```

- **Another tenant's record is not found**, so the API answers 404, as the standard says.
- **Writes are guarded as well.** The saving interceptor refuses an added or changed entity
  whose `ClubId` is not the request's, so a bug cannot write into another tenant.
- **A create takes the tenant from the current user**, never from the body:
  `_currentUser.RequireClubId()`, which throws a 403 when the request has none.
- **A reference in the body to something not in this tenant** (a member type of another
  club) is an `InvalidReferenceException` on that field, a 400.
- **Every feature has an integration test** that another tenant's record is a 404 and never
  shows up in a list or a search.

How the current tenant is known (claims, a header for an admin who switches) belongs to the
project's authentication and is recorded in its `CLAUDE.md`.

---

## 5. The audit log

`IAuditableEntity` says who created and last changed a row. With DDD the same interceptor
also writes what changed: one `AuditLogEntry` per changed field (entity type, entity id,
field, old value, new value), one for a creation, one for a deletion. History screens read
it; no handler writes history by hand.

Two things never go through it. A write that is not a change to the entity, such as a "last
signed in" timestamp, goes straight to the database with `ExecuteUpdateAsync`, past the
change tracker, so it does not overwrite who last modified the row. And the log itself.

---

## 6. Application: files per feature

The command, query, DTO, repository interface and handlers of a feature sit together in
`<Product>.Application/<Feature>/`, as in ARCHITECTURE.md. With DDD a feature has more of
them, so they are grouped:

- A small feature (a lookup list, a tariff): one file, `<Feature>Feature.cs`.
- A large one (members, invoices): `<Feature>Dtos.cs`, `<Feature>Commands.cs` (commands
  and queries), `I<Feature>Repository.cs`, `<Feature>Handlers.cs`.

Read models are not aggregates. A query may project straight from the tables into its DTO,
joins included, with `AsNoTracking()`; it does not load aggregates to show a list.

---

## 7. Sending to an outside system

When an aggregate is sent somewhere that cannot be undone (a bookkeeping system, a payment
provider), two requests must never both send it, and a lost answer must never lead to sending
it blindly again:

1. **Claim it first.** A `StartSending()` method moves it to a Sending state, and the save
   uses a row version (`Property<uint>("Version").IsRowVersion()` on PostgreSQL maps to
   `xmin`), so the second of two concurrent claims fails with a 409.
2. **From the claim on, ignore the request's cancellation token**: something the other side
   booked must be marked as booked, whether or not the browser is still there.
3. **A lost answer (timeout, connection gone) leaves it in Sending**, with the reason, and
   it is never sent again by itself. A person checks the other system and releases it.
4. **A clear refusal** marks it failed: nothing was booked, it can be changed and sent again.

---

## 8. Architecture tests on top

Besides the tests of [`testing.md`](testing.md):

- every class in the domain with an `Id` implements `IAuditableEntity`;
- **no public setters** on those classes, except the audit fields;
- the domain references no other layer.

```csharp
[Fact]
public void Entities_ShouldOnlyChangeThroughTheirOwnMethods()
{
    var auditFields = typeof(IAuditableEntity).GetProperties().Select(p => p.Name).ToHashSet();

    var failing = DomainAssembly.GetTypes()
        .Where(t => t.IsClass && !t.IsAbstract && t.GetProperty("Id") is not null)
        .SelectMany(t => t.GetProperties()
            .Where(p => p.SetMethod is { IsPublic: true } && !auditFields.Contains(p.Name))
            .Select(p => $"{t.Name}.{p.Name}"))
        .ToList();

    Assert.True(failing.Count == 0, "Public setters: " + string.Join(", ", failing));
}
```

---

## 9. Checklist: a feature with DDD

1. The aggregate and its rules in **Domain**, each rule with a unit test. A rule that says no
   throws a `DomainRuleException` with a detail in the users' language.
2. New entity: implements `IAuditableEntity`, and `IClubOwned` when a tenant owns it. EF
   configuration next to its repository, and a migration.
3. Commands, queries, DTOs, the repository interface and handlers in **Application**. A
   create takes the tenant from `RequireClubId()`; references are checked to be in this
   tenant.
4. The repository in **Infrastructure**; queries project into DTOs.
5. `<Feature>Services.cs` and `<Feature>Endpoints.cs` in **Api**, one line each in
   `Program.cs`, see ARCHITECTURE.md chapter 4.8.
6. Integration tests per endpoint, including another tenant's record giving a 404.
7. Build, regenerate the frontend client, commit both.
