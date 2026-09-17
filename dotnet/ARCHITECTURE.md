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
│   │   └── Todos/
│   │       └── Todo.cs
│   ├── TodoApp.Application/
│   │   └── Todos/
│   │       ├── ITodoRepository.cs
│   │       ├── TodoDto.cs
│   │       ├── CreateTodoCommand.cs
│   │       ├── CreateTodoCommandHandler.cs
│   │       ├── GetTodoByIdQuery.cs
│   │       └── GetTodoByIdQueryHandler.cs
│   ├── TodoApp.Infrastructure/
│   │   ├── Data/
│   │   │   └── TodoDbContext.cs
│   │   └── Todos/
│   │       └── TodoRepository.cs
│   └── TodoApp.Api/
│       ├── Program.cs
│       └── appsettings.json
└── tests/
    ├── TodoApp.Application.UnitTests/
    └── TodoApp.ArchitectureTests/
```

Within a layer, organize per feature (`Todos/`, `Orders/`, ...), not per technical type (`Commands/`, `Handlers/`, ...). Technology is a folder inside Infrastructure (`Data/` for EF Core), never its own project.

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
namespace TodoApp.Domain.Todos;

public sealed class Todo
{
    public Guid Id { get; private set; }
    public string Title { get; private set; } = string.Empty;
    public bool IsCompleted { get; private set; }
    public DateTime CreatedAtUtc { get; private set; }

    // For EF Core
    private Todo() { }

    public Todo(string title)
    {
        Id = Guid.CreateVersion7();
        Title = title;
        IsCompleted = false;
        CreatedAtUtc = DateTime.UtcNow;
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
namespace TodoApp.Application.Todos;

public sealed class TodoDto
{
    public Guid Id { get; init; }
    public string Title { get; init; } = string.Empty;
    public bool IsCompleted { get; init; }
    public DateTime CreatedAtUtc { get; init; }
}
```

Never return domain entities from the API; always a DTO.

### 4.4 Application: command + handler

`src/TodoApp.Application/Todos/CreateTodoCommand.cs`

```csharp
namespace TodoApp.Application.Todos;

public sealed class CreateTodoCommand
{
    public string Title { get; init; } = string.Empty;
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
            CreatedAtUtc = todo.CreatedAtUtc
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
            CreatedAtUtc = todo.CreatedAtUtc
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
using TodoApp.Infrastructure.Data;
using TodoApp.Infrastructure.Todos;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<TodoDbContext>(options =>
    options
        .UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
        .UseSnakeCaseNamingConvention());

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
- Commands, queries and DTOs use `init` properties.
- Every handler has one public method: `HandleAsync(request, CancellationToken)`.
- Always pass a `CancellationToken`, all the way down to EF Core.
- Commands may change state and call `SaveChangesAsync`; queries never do.
- Queries use `AsNoTracking()`.
- Endpoints contain no logic: request in, call the handler, HTTP result back.

---

## 6. Checklist: adding a new feature

1. Add an entity or behavior in **Domain** (if needed).
2. Add a repository method to the interface in **Application**.
3. Create a command or query + handler in **Application**.
4. Create or reuse a DTO.
5. Implement the repository method in **Infrastructure**.
6. Update the EF configuration and add a migration (if needed).
7. Register the handler in `Program.cs` (`AddScoped`).
8. Add the endpoint in `Program.cs`.
9. Write a unit test for the handler.

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
