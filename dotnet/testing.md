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

- Eén testproject per feature-laag die je test, bijvoorbeeld `tests/Application.Tests`.
- Naamgeving: `{Methode}_{Verwacht}_{Wanneer}`, zoals `CreateTodo_SavesAndReturnsDto`.
- Test het gedrag van de handler, niet de mock. Verifieer alleen interacties die er
  inhoudelijk toe doen (opslaan, niet-opslaan), niet elke aanroep.
- Domain-entities test je direct, zonder mocks. `Todo.MarkCompleted()` heeft geen
  repository nodig.
- Geef altijd een `CancellationToken` mee, ook in tests. `CancellationToken.None` is prima.

### Packages

```bash
dotnet new xunit -n Application.Tests -o tests/Application.Tests -f net8.0
dotnet add tests/Application.Tests reference src/Application src/Domain
dotnet add tests/Application.Tests package Moq
```

---

## 2. Architectuurtests (NetArchTest)

Deze tests dwingen de regels uit `ARCHITECTURE.md` af. Ze draaien mee met de gewone
testsuite en falen zodra iemand - bewust of per ongeluk - een laag laat verwijzen naar
een laag waar hij niets te zoeken heeft.

### Packages

```bash
dotnet new xunit -n Architecture.Tests -o tests/Architecture.Tests -f net8.0
dotnet add tests/Architecture.Tests reference src/Domain src/Application src/Infrastructure src/Api
dotnet add tests/Architecture.Tests package NetArchTest.Rules
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
    private const string DomainNamespace = "Domain";
    private const string ApplicationNamespace = "Application";
    private const string InfrastructureNamespace = "Infrastructure";
    private const string ApiNamespace = "Api";

    [Fact]
    public void Domain_ShouldNotDependOnAnyOtherLayer()
    {
        var result = Types.InAssembly(typeof(Domain.Todos.Todo).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(ApplicationNamespace, InfrastructureNamespace, ApiNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    [Fact]
    public void Application_ShouldNotDependOnInfrastructureOrApi()
    {
        var result = Types.InAssembly(typeof(Application.Todos.ITodoRepository).Assembly)
            .ShouldNot()
            .HaveDependencyOnAny(InfrastructureNamespace, ApiNamespace)
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    [Fact]
    public void Application_ShouldNotDependOnEntityFrameworkCore()
    {
        var result = Types.InAssembly(typeof(Application.Todos.ITodoRepository).Assembly)
            .ShouldNot()
            .HaveDependencyOn("Microsoft.EntityFrameworkCore")
            .GetResult();

        Assert.True(result.IsSuccessful, Format(result));
    }

    [Fact]
    public void Handlers_ShouldBeSealed()
    {
        var result = Types.InAssembly(typeof(Application.Todos.ITodoRepository).Assembly)
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

---

## 3. Integratietests

**Nog uit te werken.**

Hier hoort te staan hoe we de echte keten testen: API-endpoint erin, database eruit.
Open punten voor het team:

- `WebApplicationFactory<Program>` als host?
- Welke database: Testcontainers met SQL Server, LocalDB, of de EF Core in-memory
  provider? (In-memory gedraagt zich op punten anders dan SQL Server, dus dat is
  niet zonder meer een veilige keuze.)
- Hoe zetten we testdata klaar en ruimen we die weer op?
- Draaien ze mee in elke PR, of alleen nachtelijk?

Tot die tijd: unit tests op de handlers plus de architectuurtests hierboven.
