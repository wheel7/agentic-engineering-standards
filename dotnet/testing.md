# Testen: .NET

Hoort bij [`ARCHITECTURE.md`](ARCHITECTURE.md). Die beschrijft hoe we code indelen; dit
document beschrijft hoe we controleren dat het klopt en zo blijft.

Twee soorten tests staan hier beschreven:

- **Unit tests** - doet een handler wat hij moet doen?
- **Architectuurtests** - houden we ons aan de laagregels en conventies?

De architectuur schrijft een structuur voor, maar een document dwingt niets af. De
architectuurtests doen dat wel: die falen in CI zodra iemand een laagregel breekt.

---

## 1. Unit tests

De opzet uit `ARCHITECTURE.md` maakt handlers direct testbaar: het zijn gewone classes
met constructor-injectie, zonder framework eromheen. Je hebt dus geen host, geen
`WebApplicationFactory` en geen database nodig.

**Aanpak:**

1. Mock `ITodoRepository` met Moq.
2. Maak de handler aan met de mock.
3. Roep `HandleAsync(...)` rechtstreeks aan.
4. Controleer het **resultaat** (de teruggegeven DTO).
5. Verifieer de **interacties** met de repository.

Die laatste twee zijn allebei nodig. Het resultaat zegt of de handler het juiste
teruggeeft; de verificatie zegt of hij ook echt heeft opgeslagen. Een handler die een
mooie DTO teruggeeft maar `SaveChangesAsync` vergeet, slaagt anders gewoon.

### Voorbeeld: command handler

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

### Voorbeeld: query handler

Bij queries hoort het omgekeerde: een query mag **nooit** opslaan. Verifieer dat expliciet.

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

### Afspraken

- Eén testproject per laag die je test, met suffix `.UnitTests`: `tests/TodoApp.Application.UnitTests`.
  Naamgeving van testprojecten staat in [`solution-layout.md`](solution-layout.md).
- Naamgeving: `{Methode}_{Verwacht}_{Wanneer}`, zoals `CreateTodo_SavesAndReturnsDto`.
- Test het gedrag van de handler, niet de mock. Verifieer alleen interacties die er
  inhoudelijk toe doen (opslaan, niet-opslaan), niet elke aanroep.
- Domain-entities test je direct, zonder mocks. `Todo.MarkCompleted()` heeft geen
  repository nodig.
- Geef altijd een `CancellationToken` mee, ook in tests. `CancellationToken.None` is prima.

### Packages

```bash
dotnet new xunit -n TodoApp.Application.UnitTests -o tests/TodoApp.Application.UnitTests -f net10.0
dotnet add tests/TodoApp.Application.UnitTests reference src/TodoApp.Application src/TodoApp.Domain
dotnet add tests/TodoApp.Application.UnitTests package Moq
```

---

## 2. Architectuurtests (NetArchTest)

Deze tests dwingen de regels uit `ARCHITECTURE.md` af. Ze draaien mee met de gewone
testsuite en falen zodra iemand - bewust of per ongeluk - een laag laat verwijzen naar
een laag waar hij niets te zoeken heeft.

### Packages

```bash
dotnet new xunit -n TodoApp.ArchitectureTests -o tests/TodoApp.ArchitectureTests -f net10.0
dotnet add tests/TodoApp.ArchitectureTests reference src/TodoApp.Domain src/TodoApp.Application src/TodoApp.Infrastructure src/TodoApp.Api
dotnet add tests/TodoApp.ArchitectureTests package NetArchTest.Rules
```

### Regels die we afdwingen

| Regel | Waarom |
|---|---|
| Domain hangt nergens van af | Businessregels moeten los van techniek te begrijpen en te testen zijn |
| Application verwijst niet naar Infrastructure of Api | Afhankelijkheden wijzen naar binnen, niet naar buiten |
| Application kent geen EF Core | Anders lekt de persistentiekeuze de applicatielaag in |
| Handlers zijn `sealed` | Handlers zijn geen uitbreidingspunt; overerving maakt ze onvoorspelbaar |

### Voorbeeldtests

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
            : "Overtredingen: " + string.Join(", ", result.FailingTypeNames);
}
```

De `Format`-helper is geen detail: zonder die melding zegt een gefaalde architectuurtest
alleen "false is not true", en dan weet je nog niet welke class de regel breekt.

### Uitbreiden

Voeg een architectuurtest toe zodra een conventie uit `ARCHITECTURE.md` in review
herhaaldelijk terugkomt. Kandidaten die nu nog niet zijn afgedwongen:

- Handlers hebben precies één publieke methode `HandleAsync`
- Commands, queries en DTO's zijn `sealed` en gebruiken `init`-properties
- Domain-entities worden niet teruggegeven vanuit de Api-laag
- Zodra er een `.Contracts`-project is: dat heeft geen dependency op een andere laag

---

## 3. Integratietests

Hier testen we de echte keten: API-endpoint erin, database eruit. Het project heet
`tests/TodoApp.Api.IntegrationTests`.

**Host**: `WebApplicationFactory<Program>`, zodat dezelfde DI-registraties draaien als in
productie.

**Database**: Testcontainers met hetzelfde Postgres-image als in
[`../ops/containers.md`](../ops/containers.md). Niet de EF Core in-memory provider: die
kent geen constraints, geen foreign keys en geen echte SQL, dus een test slaagt daar
terwijl productie faalt. Een echte database in een container kost een paar seconden
opstarttijd en is dat ruimschoots waard.

Dezelfde aanpak werkt lokaal en in CI, want de GitHub-runners hebben Docker. Zie
[`../ops/ci-cd.md`](../ops/ci-cd.md).

```bash
dotnet new xunit -n TodoApp.Api.IntegrationTests -o tests/TodoApp.Api.IntegrationTests -f net10.0
dotnet add tests/TodoApp.Api.IntegrationTests reference src/TodoApp.Api
dotnet add tests/TodoApp.Api.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing
dotnet add tests/TodoApp.Api.IntegrationTests package Testcontainers.PostgreSql
```

**Draaien mee in elke pull request.** Een suite die alleen 's nachts draait, vertelt je
's ochtends dat iemand anders iets heeft gebroken, en dan is de context weg.

Nog te beslissen door het team:

- Hoe we testdata klaarzetten en opruimen: één container per testklasse, of één container
  met een transactie die na elke test terugdraait.
- Of we de migrations draaien of het schema in één keer aanmaken bij het starten van de
  container.
