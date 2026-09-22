# Architecture: CQRS in .NET 10 (pragmatic, without MediatR)

> Commands change state. Queries read state. Simple, explicit, testable.

This document describes the standard architecture for our .NET APIs. Use it as a reference when setting up a new project and when adding new features.

---

## 1. Principles

- Separate **commands** (writing) from **queries** (reading).
- Keep handlers small and focused: one handler, one responsibility.
- Four layers: **Domain**, **Application**, **Infrastructure**, **Api**.
- No MediatR needed; handlers are plain C# classes injected through DI.
- No extra ceremony: no generic pipelines, base classes or marker interfaces unless there is a real need for them.

### Why CQRS?

- Every handler has one responsibility.
- Read and write models can evolve independently.
- Easy to test and to understand.
- Clear intent in the code and in the API.

### When not to use it?

- The service is small and has few endpoints.
- It is a prototype or an internal tool.
- The domain is simple CRUD without business rules.
- The team does not know the pattern and the deadline is tight.

In those cases: start with a simple service class and split off handlers as soon as the complexity grows.

---

## 2. Layers and dependencies

| Layer | Responsibility | May reference |
|---|---|---|
| **Domain** | Entities and business rules | nothing |
| **Application** | Commands, queries, handlers, DTOs, interfaces | Domain |
| **Infrastructure** | EF Core, repositories, external systems | Application, Domain |
| **Api** | Minimal APIs, DI, configuration | Application, Infrastructure |

Rule: dependencies always point inward. Domain knows nobody; Application knows no EF Core.

Every layer is its own project named `<Product>.<Layer>`, where assembly name, root namespace and folder name are identical. In this document the product is `TodoApp`. What projects are called, what test projects are called and which optional projects exist (`.Contracts`, `.Persistence`, `.Migrations`) is in [`solution-layout.md`](solution-layout.md).

### Folder structure

```
TodoApp.sln
├── src/
│   ├── TodoApp.Domain/
│   │   ├── Auditing/
│   │   │   └── IAuditableEntity.cs
│   │   └── Todos/
│   │       └── Todo.cs
│   ├── TodoApp.Application/
│   │   ├── Auditing/
│   │   │   └── IAuditableDto.cs
│   │   ├── Identity/
│   │   │   └── ICurrentUser.cs
│   │   └── Todos/
│   │       ├── ITodoRepository.cs
│   │       ├── TodoDto.cs
│   │       ├── CreateTodoCommand.cs
│   │       ├── CreateTodoCommandHandler.cs
│   │       ├── GetTodoByIdQuery.cs
│   │       └── GetTodoByIdQueryHandler.cs
│   ├── TodoApp.Infrastructure/
│   │   ├── Auditing/
│   │   │   └── AuditingInterceptor.cs
│   │   ├── Data/
│   │   │   └── TodoDbContext.cs
│   │   ├── Identity/
│   │   │   └── CurrentUser.cs
│   │   └── Todos/
│   │       └── TodoRepository.cs
│   └── TodoApp.Api/
│       ├── Program.cs
│       └── appsettings.json
└── tests/
    ├── TodoApp.Application.UnitTests/
    └── TodoApp.ArchitectureTests/
```

Within a layer, organize per feature (`Todos/`, `Orders/`, ...), not per technical type (`Commands/`, `Handlers/`, ...). Technology is a folder inside Infrastructure (`Data/` for EF Core), never its own project. Cross-cutting concerns such as `Auditing/` and `Identity/` get a folder named after what they hold, never one called `Common` or `Shared`.

---

## 3. Setting up the project

```bash
dotnet new sln -n TodoApp

dotnet new classlib -n TodoApp.Domain         -o src/TodoApp.Domain         -f net10.0
dotnet new classlib -n TodoApp.Application    -o src/TodoApp.Application    -f net10.0
dotnet new classlib -n TodoApp.Infrastructure -o src/TodoApp.Infrastructure -f net10.0
dotnet new web      -n TodoApp.Api            -o src/TodoApp.Api            -f net10.0

dotnet sln add src/TodoApp.Domain src/TodoApp.Application src/TodoApp.Infrastructure src/TodoApp.Api

# Project references
dotnet add src/TodoApp.Application    reference src/TodoApp.Domain
dotnet add src/TodoApp.Infrastructure reference src/TodoApp.Application src/TodoApp.Domain
dotnet add src/TodoApp.Api            reference src/TodoApp.Application src/TodoApp.Infrastructure
```

`-n` and `-o` get the same name, so that assembly, root namespace and folder match. Replace `TodoApp` with the name of your own product.

### NuGet packages

Note: EF Core is **not** part of the .NET 10 SDK by default; you have to add these packages yourself.

```bash
# Infrastructure: EF Core + PostgreSQL provider + snake_case naming
dotnet add src/TodoApp.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL --version 10.0.*
dotnet add src/TodoApp.Infrastructure package EFCore.NamingConventions --version 10.0.*

# Api: needed for migrations through the CLI
dotnet add src/TodoApp.Api package Microsoft.EntityFrameworkCore.Design --version 10.0.*

# One time: the dotnet-ef tool
dotnet tool install --global dotnet-ef --version 10.*
```

If you use the Package Manager Console in Visual Studio, also add `Microsoft.EntityFrameworkCore.Tools` to the Api project.

PostgreSQL is our standard database. The conventions for column types, keys and migrations are in [`../ops/database.md`](../ops/database.md); running locally with `docker compose up` is in [`../ops/containers.md`](../ops/containers.md).

---

## 4. Code

### 4.1 Domain: entity

`src/TodoApp.Domain/Todos/Todo.cs`

```csharp
using TodoApp.Domain.Auditing;

namespace TodoApp.Domain.Todos;

public sealed class Todo : IAuditableEntity
{
    public Guid Id { get; private set; }
    public string Title { get; private set; } = string.Empty;
    public bool IsCompleted { get; private set; }

    public Guid CreatedBy { get; set; }
    public DateTime CreatedDate { get; set; }
    public Guid? ModifiedBy { get; set; }
    public DateTime? ModifiedDate { get; set; }

    // For EF Core
    private Todo() { }

    public Todo(string title)
    {
        Id = Guid.CreateVersion7();
        Title = title;
        IsCompleted = false;
    }

    public void MarkCompleted()
    {
        if (!IsCompleted)
            IsCompleted = true;
    }
}
```

Rules: private setters, change state only through methods with meaningful names, no public parameterless constructor.

`Guid.CreateVersion7()` instead of `Guid.NewGuid()`: those keys increase over time and keep the index compact. See [`../ops/database.md`](../ops/database.md).

Every entity implements `IAuditableEntity`, which is why the four audit properties are there and why they are the one place with public setters. The constructor does not fill them: a `SaveChanges` interceptor sets who and when, so no handler can forget it. See [`../ops/database.md`](../ops/database.md) for the interface, the interceptor and the user model behind `CreatedBy`.

### 4.2 Application: repository interface

`src/TodoApp.Application/Todos/ITodoRepository.cs`

```csharp
using TodoApp.Domain.Todos;

namespace TodoApp.Application.Todos;

public interface ITodoRepository
{
    Task AddAsync(Todo todo, CancellationToken cancellationToken);

    Task<Todo?> GetByIdAsync(Guid id, CancellationToken cancellationToken);

    Task SaveChangesAsync(CancellationToken cancellationToken);
}
```

### 4.3 Application: DTO

`src/TodoApp.Application/Todos/TodoDto.cs`

```csharp
using TodoApp.Application.Auditing;

namespace TodoApp.Application.Todos;

public sealed class TodoDto : IAuditableDto
{
    public required Guid Id { get; init; }
    public required string Title { get; init; }
    public required bool IsCompleted { get; init; }

    public required Guid CreatedBy { get; init; }
    public required DateTime CreatedDate { get; init; }
    public required Guid? ModifiedBy { get; init; }
    public required DateTime? ModifiedDate { get; init; }
}
```

Never return domain entities from the API; always a DTO. A DTO that represents an auditable entity implements `IAuditableDto`, so the audit values travel in the same shape everywhere.

**Every property is `required`**, in a DTO and in a command or query that is bound from a request body. The OpenAPI document reads it:

- On a response, `required` is what makes the document say the field is always there. Without it every field is optional, and a client generated from the document has to check each one for a value that is never missing. [`../general/api-contracts.md`](../general/api-contracts.md) chapter 5 says a response always carries the field.
- On a request, `required` makes a missing field a `400`. Without it a missing `int` arrives as `0` and a missing `bool` as `false`, which are valid values, so nothing notices.

### 4.4 Application: command + handler

`src/TodoApp.Application/Todos/CreateTodoCommand.cs`

```csharp
namespace TodoApp.Application.Todos;

public sealed class CreateTodoCommand
{
    public required string Title { get; init; }
}
```

`src/TodoApp.Application/Todos/CreateTodoCommandHandler.cs`

```csharp
using TodoApp.Domain.Todos;

namespace TodoApp.Application.Todos;

public sealed class CreateTodoCommandHandler
{
    private readonly ITodoRepository _repository;

    public CreateTodoCommandHandler(ITodoRepository repository)
        => _repository = repository;

    public async Task<TodoDto> HandleAsync(
        CreateTodoCommand command,
        CancellationToken cancellationToken)
    {
        var todo = new Todo(command.Title);

        await _repository.AddAsync(todo, cancellationToken);
        await _repository.SaveChangesAsync(cancellationToken);

        return new TodoDto
        {
            Id = todo.Id,
            Title = todo.Title,
            IsCompleted = todo.IsCompleted,
            CreatedBy = todo.CreatedBy,
            CreatedDate = todo.CreatedDate,
            ModifiedBy = todo.ModifiedBy,
            ModifiedDate = todo.ModifiedDate
        };
    }
}
```

### 4.5 Application: query + handler

`src/TodoApp.Application/Todos/GetTodoByIdQuery.cs`

```csharp
namespace TodoApp.Application.Todos;

public sealed class GetTodoByIdQuery
{
    public Guid Id { get; init; }
}
```

`src/TodoApp.Application/Todos/GetTodoByIdQueryHandler.cs`

```csharp
using TodoApp.Domain.Todos;

namespace TodoApp.Application.Todos;

public sealed class GetTodoByIdQueryHandler
{
    private readonly ITodoRepository _repository;

    public GetTodoByIdQueryHandler(ITodoRepository repository)
        => _repository = repository;

    public async Task<TodoDto?> HandleAsync(
        GetTodoByIdQuery query,
        CancellationToken cancellationToken)
    {
        var todo = await _repository.GetByIdAsync(query.Id, cancellationToken);
        if (todo is null)
            return null;

        return new TodoDto
        {
            Id = todo.Id,
            Title = todo.Title,
            IsCompleted = todo.IsCompleted,
            CreatedBy = todo.CreatedBy,
            CreatedDate = todo.CreatedDate,
            ModifiedBy = todo.ModifiedBy,
            ModifiedDate = todo.ModifiedDate
        };
    }
}
```

### 4.6 Infrastructure: EF Core DbContext

`src/TodoApp.Infrastructure/Data/TodoDbContext.cs`

```csharp
using TodoApp.Domain.Todos;
using Microsoft.EntityFrameworkCore;

namespace TodoApp.Infrastructure.Data;

public sealed class TodoDbContext : DbContext
{
    public TodoDbContext(DbContextOptions<TodoDbContext> options)
        : base(options) { }

    public DbSet<Todo> Todos => Set<Todo>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        var todo = modelBuilder.Entity<Todo>();

        todo.HasKey(t => t.Id);
        todo.Property(t => t.Title)
            .IsRequired()
            .HasMaxLength(200);
    }
}
```

### 4.7 Infrastructure: repository implementation

`src/TodoApp.Infrastructure/Todos/TodoRepository.cs`

```csharp
using TodoApp.Application.Todos;
using TodoApp.Domain.Todos;
using TodoApp.Infrastructure.Data;
using Microsoft.EntityFrameworkCore;

namespace TodoApp.Infrastructure.Todos;

public sealed class TodoRepository : ITodoRepository
{
    private readonly TodoDbContext _dbContext;

    public TodoRepository(TodoDbContext dbContext) => _dbContext = dbContext;

    public async Task AddAsync(Todo todo, CancellationToken cancellationToken)
        => await _dbContext.Todos.AddAsync(todo, cancellationToken);

    public async Task<Todo?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
        => await _dbContext.Todos
            .AsNoTracking()
            .FirstOrDefaultAsync(t => t.Id == id, cancellationToken);

    public async Task SaveChangesAsync(CancellationToken cancellationToken)
        => await _dbContext.SaveChangesAsync(cancellationToken);
}
```

Note: `GetByIdAsync` uses `AsNoTracking()` and is therefore meant for queries. If a command has to change an existing entity (for example `MarkCompleted()`), add a separate method **without** `AsNoTracking()`, otherwise the changes are not saved.

### 4.8 Api: Minimal API endpoints + DI

`src/TodoApp.Api/Program.cs`

```csharp
using TodoApp.Application.Todos;
using TodoApp.Infrastructure.Auditing;
using TodoApp.Infrastructure.Data;
using TodoApp.Infrastructure.Todos;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<AuditingInterceptor>();

builder.Services.AddDbContext<TodoDbContext>((sp, options) =>
    options
        .UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
        .UseSnakeCaseNamingConvention()
        .AddInterceptors(sp.GetRequiredService<AuditingInterceptor>()));

// Repositories
builder.Services.AddScoped<ITodoRepository, TodoRepository>();

// Handlers
builder.Services.AddScoped<CreateTodoCommandHandler>();
builder.Services.AddScoped<GetTodoByIdQueryHandler>();

var app = builder.Build();

// CREATE TODO (Command)
app.MapPost("/todos", async (
    CreateTodoCommand command,
    CreateTodoCommandHandler handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(command, ct);
    return Results.Created($"/todos/{result.Id}", result);
});

// GET TODO BY ID (Query)
app.MapGet("/todos/{id:guid}", async (
    Guid id,
    GetTodoByIdQueryHandler handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(new GetTodoByIdQuery { Id = id }, ct);
    return result is null ? Results.NotFound() : Results.Ok(result);
});

app.Run();
```

`src/TodoApp.Api/appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=todoapp;Username=todoapp;Password=localdev"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### 4.9 Changing and removing: the id comes from the route

An update binds the fields from the body and the id from the route. The command is a
`sealed record`, so the endpoint can put the id in with `with`, and `[JsonIgnore]` keeps it
out of the body and out of the OpenAPI document:

```csharp
public sealed record UpdateTodoCommand
{
    [JsonIgnore]
    public Guid Id { get; init; }

    public required string Title { get; init; }
}
```

```csharp
app.MapPut("/todos/{id:guid}", async (
    Guid id,
    UpdateTodoCommand command,
    UpdateTodoCommandHandler handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(command with { Id = id }, ct);
    return result is null ? Results.NotFound() : Results.Ok(result);
});
```

- The handler loads the entity with a **tracked** repository method, changes it through a
  method on the entity, and saves. It returns `null` when there is no such entity for this
  user, and the endpoint makes that a `404`.
- An update returns `200` with the DTO, so the client has what the server made of it.
- A delete returns `204`, or `404` when there was nothing to delete.
- When create and update take the same fields, put them on an `abstract record` both
  inherit from, with the validation on it once.

### 4.10 Errors: 400, 404 and 409

What goes over the wire is in [`../general/api-contracts.md`](../general/api-contracts.md)
chapter 2. How the code gets there:

- **400, a malformed request**, is validation at the edge: data annotations on the command,
  and `IValidatableObject` for a rule across fields such as a minimum above a maximum. Put
  the error on the member a user corrects. The keys are rewritten to the request body's
  camelCase names in one `CustomizeProblemDetails`, so every endpoint gets that for free.
- **400, a body that cannot be read**, invalid JSON or a missing `required` field, needs a
  handler of its own. In Development ASP.NET throws `BadHttpRequestException` instead of
  answering, and the exception handler turns it into a `500`. So in every project:

  ```csharp
  public sealed class BadRequestExceptionHandler : IExceptionHandler
  {
      private readonly IProblemDetailsService _problemDetails;

      public BadRequestExceptionHandler(IProblemDetailsService problemDetails) => _problemDetails = problemDetails;

      public async ValueTask<bool> TryHandleAsync(
          HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
      {
          if (exception is not BadHttpRequestException badRequest)
              return false;

          httpContext.Response.StatusCode = badRequest.StatusCode;
          return await _problemDetails.TryWriteAsync(new ProblemDetailsContext
          {
              HttpContext = httpContext,
              ProblemDetails =
              {
                  Status = badRequest.StatusCode,
                  Title = "The request could not be read.",
                  Detail = "The body is not valid JSON, or a required field is missing."
              }
          });
      }
  }
  ```

  The fixed detail is on purpose: the parser's own message names your internal types.
- **404** is a handler returning `null` or `false`. A record of another user is a 404 too,
  never a 403, because every repository method filters on the owner.
- **409, it conflicts with what is already there**, such as a name that has to be unique.
  The handler throws a `ConflictException(type, title, detail)` that lives in
  `.Application/Errors/`, and a `ConflictExceptionHandler` in `.Api/Errors/` writes it as
  problem+json with `type` set to `https://<product-domain>/errors/<type>`. The frontend
  switches on that `type`. A unique index in the database backs the check up for a race.

Register both handlers with `AddExceptionHandler<...>()`, after `AddProblemDetails()`.

### 4.11 JSON on the wire

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.NumberHandling = JsonNumberHandling.Strict;
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
```

- **Strict numbers**, because the web default also accepts a number as a string, and the
  OpenAPI document then types every integer as "number or string".
- **Enums by name**, `"Sometimes"` and not `1`. It matches how
  [`../ops/database.md`](../ops/database.md) stores them, a reordered enum breaks nothing,
  and the contract reads. An integration test that reads a response with its own
  `JsonSerializerOptions` needs the same converter, or it cannot read what the API wrote.

### After a change to an endpoint or a DTO

The OpenAPI document is generated on build and committed, and the frontend generates its
client from it, see [`../general/api-contracts.md`](../general/api-contracts.md) chapter 6.
So a change to the contract is three files in the same pull request: the code, the
document, and the regenerated client. CI fails on either of the last two being out of date.

### Migrations

```bash
dotnet ef migrations add InitialCreate --project src/TodoApp.Infrastructure --startup-project src/TodoApp.Api
dotnet ef database update             --project src/TodoApp.Infrastructure --startup-project src/TodoApp.Api
```

You run `database update` locally. On test and production, migrations go through the pipeline and never at application startup; see [`../ops/database.md`](../ops/database.md) and [`../ops/ci-cd.md`](../ops/ci-cd.md).

---

## 5. Conventions

| Part | Naming | Example |
|---|---|---|
| Command | `{Verb}{Entity}Command` | `CreateTodoCommand` |
| Command handler | `{Command}Handler` | `CreateTodoCommandHandler` |
| Query | `Get{Entity}By{Criterion}Query` | `GetTodoByIdQuery` |
| Query handler | `{Query}Handler` | `GetTodoByIdQueryHandler` |
| DTO | `{Entity}Dto` | `TodoDto` |
| Repository | `I{Entity}Repository` / `{Entity}Repository` | `ITodoRepository` |

Further conventions:

- All classes are `sealed`, unless inheritance is really needed.
- Commands, queries and DTOs use `init` properties, and every property bound from a request
  or returned in a response is `required`, see 4.3.
- A command that carries an id from the route is a `sealed record`, see 4.9.
- A conflict is a `ConflictException`, never a status code picked in the endpoint, see 4.10.
- Every handler has one public method: `HandleAsync(request, CancellationToken)`.
- Always pass a `CancellationToken`, all the way down to EF Core.
- Commands may change state and call `SaveChangesAsync`; queries never do.
- Queries use `AsNoTracking()`.
- Endpoints contain no logic: request in, call the handler, HTTP result back.

---

## 6. Checklist: adding a new feature

1. Add an entity or behavior in **Domain** (if needed). An entity implements `IAuditableEntity`.
2. Add a repository method to the interface in **Application**.
3. Create a command or query + handler in **Application**.
4. Create or reuse a DTO. One that exposes an auditable entity implements `IAuditableDto`.
5. Implement the repository method in **Infrastructure**.
6. Update the EF configuration and add a migration (if needed).
7. Register the handler in `Program.cs` (`AddScoped`).
8. Add the endpoint in `Program.cs`.
9. Build, and regenerate the frontend client from the OpenAPI document. Commit both.
10. Write a unit test for the handler, and an integration test for the endpoint, including
    that another user's record is a 404.

---

## 7. Testing

Unit tests live in `tests/TodoApp.Application.UnitTests`; see [`testing.md`](testing.md) for the setup and the architecture tests. Mock `ITodoRepository` (with Moq, for example), call `HandleAsync(...)` directly, check the result and verify the interactions.

```csharp
[Fact]
public async Task CreateTodo_SavesAndReturnsDto()
{
    var repository = new Mock<ITodoRepository>();
    var handler = new CreateTodoCommandHandler(repository.Object);

    var result = await handler.HandleAsync(
        new CreateTodoCommand { Title = "Buy groceries" },
        CancellationToken.None);

    Assert.Equal("Buy groceries", result.Title);
    Assert.False(result.IsCompleted);
    repository.Verify(r => r.AddAsync(It.IsAny<Todo>(), It.IsAny<CancellationToken>()), Times.Once);
    repository.Verify(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
}
```

---

## 8. What you get

- Small, focused handlers for commands and queries.
- A clear separation between read and write logic.
- Easy testing with mockable dependencies.
- A thin API layer with Minimal APIs.
- Ready to grow: validation, logging, transactions or MediatR can be added later.

## 9. Next steps (when needed)

1. Add validation (by hand or with FluentValidation).
2. Add logging with `ILogger<T>`.
3. Transactions and the outbox pattern when needed.
4. Unit and integration tests.

Grow deliberately: add what you need, when you need it.
