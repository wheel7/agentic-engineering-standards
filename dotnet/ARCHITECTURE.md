# Architectuur: CQRS in .NET 10 (pragmatisch, zonder MediatR)

> Commands wijzigen state. Queries lezen state. Simpel, expliciet, testbaar.

Dit document beschrijft de standaardarchitectuur voor onze .NET API's. Gebruik het als referentie bij het opzetten van een nieuw project en bij het toevoegen van nieuwe features.

---

## 1. Principes

- Scheid **commands** (schrijven) van **queries** (lezen).
- Houd handlers klein en gefocust: één handler, één verantwoordelijkheid.
- Vier lagen: **Domain**, **Application**, **Infrastructure**, **Api**.
- Geen MediatR nodig; handlers zijn gewone C#-classes die via DI worden geïnjecteerd.
- Geen extra ceremonie: geen generieke pipelines, base classes of marker-interfaces tenzij er echt behoefte aan is.

### Waarom CQRS?

- Elke handler heeft één verantwoordelijkheid.
- Lees- en schrijfmodellen kunnen onafhankelijk evolueren.
- Makkelijk te testen en te begrijpen.
- Duidelijke intentie in de code én in de API.

### Wanneer níet gebruiken?

- De service is klein en heeft weinig endpoints.
- Het is een prototype of interne tool.
- Het domein is simpele CRUD zonder businessregels.
- Het team kent het patroon niet en de deadline is krap.

In die gevallen: begin met een simpele service-class en splits handlers af zodra de complexiteit toeneemt.

---

## 2. Lagen en afhankelijkheden

| Laag | Verantwoordelijkheid | Mag verwijzen naar |
|---|---|---|
| **Domain** | Entities en businessregels | niets |
| **Application** | Commands, queries, handlers, DTO's, interfaces | Domain |
| **Infrastructure** | EF Core, repositories, externe systemen | Application, Domain |
| **Api** | Minimal API's, DI, configuratie | Application, Infrastructure |

Regel: afhankelijkheden wijzen altijd naar binnen. Domain kent niemand; Application kent geen EF Core.

Elke laag is een eigen project met de naam `<Product>.<Laag>`, waarbij assembly-naam, root-namespace en mapnaam gelijk zijn. In dit document is het product `TodoApp`. Hoe projecten heten, hoe testprojecten heten en welke optionele projecten er zijn (`.Contracts`, `.Persistence`, `.Migrations`) staat in [`solution-layout.md`](solution-layout.md).

### Mappenstructuur

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

Organiseer binnen een laag per feature (`Todos/`, `Orders/`, ...), niet per technisch type (`Commands/`, `Handlers/`, ...). Technologie is een map binnen Infrastructure (`Data/` voor EF Core), nooit een eigen project.

---

## 3. Project opzetten

```bash
dotnet new sln -n TodoApp

dotnet new classlib -n TodoApp.Domain         -o src/TodoApp.Domain         -f net10.0
dotnet new classlib -n TodoApp.Application    -o src/TodoApp.Application    -f net10.0
dotnet new classlib -n TodoApp.Infrastructure -o src/TodoApp.Infrastructure -f net10.0
dotnet new web      -n TodoApp.Api            -o src/TodoApp.Api            -f net10.0

dotnet sln add src/TodoApp.Domain src/TodoApp.Application src/TodoApp.Infrastructure src/TodoApp.Api

# Projectreferenties
dotnet add src/TodoApp.Application    reference src/TodoApp.Domain
dotnet add src/TodoApp.Infrastructure reference src/TodoApp.Application src/TodoApp.Domain
dotnet add src/TodoApp.Api            reference src/TodoApp.Application src/TodoApp.Infrastructure
```

`-n` en `-o` krijgen dezelfde naam, zodat assembly, root-namespace en map overeenkomen. Vervang `TodoApp` door de naam van je eigen product.

### NuGet-packages

Let op: EF Core zit **niet** standaard in de .NET 10 SDK; deze packages moet je zelf toevoegen.

```bash
# Infrastructure: EF Core + PostgreSQL provider + snake_case naamgeving
dotnet add src/TodoApp.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL --version 10.0.*
dotnet add src/TodoApp.Infrastructure package EFCore.NamingConventions --version 10.0.*

# Api: nodig voor migrations via de CLI
dotnet add src/TodoApp.Api package Microsoft.EntityFrameworkCore.Design --version 10.0.*

# Eenmalig: de dotnet-ef tool
dotnet tool install --global dotnet-ef --version 10.*
```

Gebruik je de Package Manager Console in Visual Studio, voeg dan ook `Microsoft.EntityFrameworkCore.Tools` toe aan het Api-project.

PostgreSQL is onze standaarddatabase. De afspraken over kolomtypes, sleutels en migrations staan in [`../ops/database.md`](../ops/database.md); lokaal draaien via `docker compose up` staat in [`../ops/containers.md`](../ops/containers.md).

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

    // Voor EF Core
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

Regels: private setters, state alleen wijzigen via methodes met betekenisvolle namen, geen publieke parameterloze constructor.

`Guid.CreateVersion7()` in plaats van `Guid.NewGuid()`: die sleutels lopen op in de tijd en houden de index compact. Zie [`../ops/database.md`](../ops/database.md).

### 4.2 Application: repository-interface

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

Geef nooit domain-entities terug vanuit de API; altijd een DTO.

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

### 4.7 Infrastructure: repository-implementatie

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

Let op: `GetByIdAsync` gebruikt `AsNoTracking()` en is dus bedoeld voor queries. Moet een command een bestaande entity wijzigen (bijv. `MarkCompleted()`), voeg dan een aparte methode toe **zonder** `AsNoTracking()`, anders worden de wijzigingen niet opgeslagen.

### 4.8 Api: Minimal API-endpoints + DI

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

`database update` draai je lokaal. Op test en productie gaan migrations via de pipeline en nooit bij het opstarten van de applicatie; zie [`../ops/database.md`](../ops/database.md) en [`../ops/ci-cd.md`](../ops/ci-cd.md).

---

## 5. Conventies

| Onderdeel | Naamgeving | Voorbeeld |
|---|---|---|
| Command | `{Werkwoord}{Entity}Command` | `CreateTodoCommand` |
| Command handler | `{Command}Handler` | `CreateTodoCommandHandler` |
| Query | `Get{Entity}By{Criterium}Query` | `GetTodoByIdQuery` |
| Query handler | `{Query}Handler` | `GetTodoByIdQueryHandler` |
| DTO | `{Entity}Dto` | `TodoDto` |
| Repository | `I{Entity}Repository` / `{Entity}Repository` | `ITodoRepository` |

Verdere afspraken:

- Alle classes zijn `sealed`, tenzij overerving echt nodig is.
- Commands, queries en DTO's gebruiken `init`-properties.
- Elke handler heeft één publieke methode: `HandleAsync(request, CancellationToken)`.
- Geef altijd een `CancellationToken` door, tot en met EF Core.
- Commands mogen state wijzigen en roepen `SaveChangesAsync` aan; queries nooit.
- Queries gebruiken `AsNoTracking()`.
- Endpoints bevatten geen logica: request binnen, handler aanroepen, HTTP-resultaat terug.

---

## 6. Checklist: nieuwe feature toevoegen

1. Entity of gedrag toevoegen in **Domain** (indien nodig).
2. Repository-methode toevoegen aan de interface in **Application**.
3. Command of query + handler maken in **Application**.
4. DTO maken of hergebruiken.
5. Repository-methode implementeren in **Infrastructure**.
6. EF-configuratie bijwerken en migration toevoegen (indien nodig).
7. Handler registreren in `Program.cs` (`AddScoped`).
8. Endpoint toevoegen in `Program.cs`.
9. Unit test voor de handler schrijven.

---

## 7. Testen

Unit tests staan in `tests/TodoApp.Application.UnitTests`; zie [`testing.md`](testing.md) voor de opzet en de architectuurtests. Mock `ITodoRepository` (bijvoorbeeld met Moq), roep `HandleAsync(...)` rechtstreeks aan, controleer het resultaat en verifieer de interacties.

```csharp
[Fact]
public async Task CreateTodo_SavesAndReturnsDto()
{
    var repository = new Mock<ITodoRepository>();
    var handler = new CreateTodoCommandHandler(repository.Object);

    var result = await handler.HandleAsync(
        new CreateTodoCommand { Title = "Boodschappen" },
        CancellationToken.None);

    Assert.Equal("Boodschappen", result.Title);
    Assert.False(result.IsCompleted);
    repository.Verify(r => r.AddAsync(It.IsAny<Todo>(), It.IsAny<CancellationToken>()), Times.Once);
    repository.Verify(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
}
```

---

## 8. Wat dit oplevert

- Kleine, gefocuste handlers voor commands en queries.
- Duidelijke scheiding tussen lees- en schrijflogica.
- Makkelijk testen met mockbare afhankelijkheden.
- Een dunne API-laag met Minimal API's.
- Klaar om te groeien: validatie, logging, transacties of MediatR kunnen later worden toegevoegd.

## 9. Volgende stappen (wanneer nodig)

1. Validatie toevoegen (handmatig of met FluentValidation).
2. Logging toevoegen met `ILogger<T>`.
3. Transacties en outbox-pattern wanneer nodig.
4. Unit- en integratietests.

Groei bewust: voeg toe wat je nodig hebt, wanneer je het nodig hebt.
