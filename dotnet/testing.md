# Testing: .NET

Goes with [`ARCHITECTURE.md`](ARCHITECTURE.md). That describes how we lay out code; this
document describes how we check that it is right and stays that way.

Two kinds of tests are described here:

- **Unit tests** - does a handler do what it should?
- **Architecture tests** - are we sticking to the layer rules and conventions?

The architecture prescribes a structure, but a document enforces nothing. The
architecture tests do: they fail in CI as soon as someone breaks a layer rule.

---

## 1. Unit tests

The setup from `ARCHITECTURE.md` makes handlers directly testable: they are plain classes
with constructor injection, without a framework around them. So you need no host, no
`WebApplicationFactory` and no database.

**Approach:**

1. Mock `ITodoRepository` with Moq.
2. Create the handler with the mock.
3. Call `HandleAsync(...)` directly.
4. Check the **result** (the returned DTO).
5. Verify the **interactions** with the repository.

Those last two are both needed. The result says whether the handler returns the right
thing; the verification says whether it actually saved. Otherwise a handler that returns
a nice DTO but forgets `SaveChangesAsync` passes all the same.

### Example: command handler

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

### Example: query handler

For queries the opposite applies: a query may **never** save. Verify that explicitly.

```csharp
[Fact]
public async Task GetTodoById_ReturnsNull_WhenNotFound()
{
    var repository = new Mock<ITodoRepository>();
    repository
        .Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync((Todo?)null);

    var handler = new GetTodoByIdQueryHandler(repository.Object);

    var result = await handler.HandleAsync(
        new GetTodoByIdQuery { Id = Guid.NewGuid() },
        CancellationToken.None);

    Assert.Null(result);
    repository.Verify(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Never);
}
```

### Conventions

- One test project per layer you test, with suffix `.UnitTests`: `tests/TodoApp.Application.UnitTests`.
  Naming of test projects is in [`solution-layout.md`](solution-layout.md).
- Naming: `{Method}_{Expected}_{When}`, such as `CreateTodo_SavesAndReturnsDto`.
- Test the behavior of the handler, not the mock. Only verify interactions that really
  matter (saving, not saving), not every call.
- Domain entities you test directly, without mocks. `Todo.MarkCompleted()` needs no
  repository.
- Always pass a `CancellationToken`, in tests too. `CancellationToken.None` is fine.

### Packages

```bash
dotnet new xunit -n TodoApp.Application.UnitTests -o tests/TodoApp.Application.UnitTests -f net10.0
dotnet add tests/TodoApp.Application.UnitTests reference src/TodoApp.Application src/TodoApp.Domain
dotnet add tests/TodoApp.Application.UnitTests package Moq
```

---

## 2. Architecture tests (NetArchTest)

These tests enforce the rules from `ARCHITECTURE.md`. They run along with the normal test
suite and fail as soon as someone - deliberately or by accident - makes one layer
reference a layer it has no business with.

### Packages

```bash
dotnet new xunit -n TodoApp.ArchitectureTests -o tests/TodoApp.ArchitectureTests -f net10.0
dotnet add tests/TodoApp.ArchitectureTests reference src/TodoApp.Domain src/TodoApp.Application src/TodoApp.Infrastructure src/TodoApp.Api
dotnet add tests/TodoApp.ArchitectureTests package NetArchTest.Rules
```

### Rules we enforce

| Rule | Why |
|---|---|
| Domain depends on nothing | Business rules have to be understandable and testable apart from technology |
| Application does not reference Infrastructure or Api | Dependencies point inward, not outward |
| Application knows no EF Core | Otherwise the persistence choice leaks into the application layer |
| Handlers are `sealed` | Handlers are not an extension point; inheritance makes them unpredictable |

### Example tests

```csharp
using NetArchTest.Rules;
using Xunit;

public class ArchitectureTests
{
    private const string DomainNamespace = "TodoApp.Domain";
    private const string ApplicationNamespace = "TodoApp.Application";
    private const string InfrastructureNamespace = "TodoApp.Infrastructure";
    private const string ApiNamespace = "TodoApp.Api";

    [Fact]
    public void Domain_ShouldNotDependOnAnyOtherLayer()
    {
        var result = Types.InAssembly(typeof(TodoApp.Domain.Todos.Todo).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(ApplicationNamespace, InfrastructureNamespace, ApiNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    [Fact]
    public void Application_ShouldNotDependOnInfrastructureOrApi()
    {
        var result = Types.InAssembly(typeof(TodoApp.Application.Todos.ITodoRepository).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(InfrastructureNamespace, ApiNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    [Fact]
    public void Application_ShouldNotDependOnEntityFrameworkCore()
    {
        var result = Types.InAssembly(typeof(TodoApp.Application.Todos.ITodoRepository).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    [Fact]
    public void Handlers_ShouldBeSealed()
    {
        var result = Types.InAssembly(typeof(TodoApp.Application.Todos.ITodoRepository).Assembly)
            .That()
            .HaveNameEndingWith("Handler")
            .Should()
            .BeSealed()
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    private static string Format(TestResult result) =>
        result.IsSuccessful
            ? string.Empty
            : "Violations: " + string.Join(", ", result.FailingTypeNames);
}
```

The `Format` helper is not a detail: without that message a failed architecture test says
only "false is not true", and then you still do not know which class breaks the rule.

### Extending

Add an architecture test as soon as a convention from `ARCHITECTURE.md` keeps coming back
in review. Candidates that are not enforced yet:

- Handlers have exactly one public method `HandleAsync`
- Commands, queries and DTOs are `sealed` and use `init` properties
- Domain entities are not returned from the Api layer
- Once there is a `.Contracts` project: it has no dependency on another layer

---

## 3. Integration tests

Here we test the real chain: API endpoint in, database out. The project is called
`tests/TodoApp.Api.IntegrationTests`.

**Host**: `WebApplicationFactory<Program>`, so that the same DI registrations run as in
production.

**Database**: Testcontainers with the same Postgres image as in
[`../ops/containers.md`](../ops/containers.md). Not the EF Core in-memory provider: it
knows no constraints, no foreign keys and no real SQL, so a test passes there while
production fails. A real database in a container costs a few seconds of startup time and
is well worth it.

The same approach works locally and in CI, because the GitHub runners have Docker. See
[`../ops/ci-cd.md`](../ops/ci-cd.md).

```bash
dotnet new xunit -n TodoApp.Api.IntegrationTests -o tests/TodoApp.Api.IntegrationTests -f net10.0
dotnet add tests/TodoApp.Api.IntegrationTests reference src/TodoApp.Api
dotnet add tests/TodoApp.Api.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing
dotnet add tests/TodoApp.Api.IntegrationTests package Testcontainers.PostgreSql
```

**They run in every pull request.** A suite that only runs at night tells you in the
morning that someone else broke something, and by then the context is gone.

Still to be decided by the team:

- How we set up and clean up test data: one container per test class, or one container
  with a transaction that rolls back after each test.
- Whether we run the migrations or create the schema in one go when starting the
  container.
