# Solution-layout en projectnaamgeving: .NET

Hoort bij [`ARCHITECTURE.md`](ARCHITECTURE.md). Die beschrijft de lagen en hoe de code
erin eruitziet; dit document beschrijft hoe de solution is ingedeeld en hoe projecten,
assemblies en namespaces heten.

> Assembly-naam, root-namespace en mapnaam zijn altijd identiek.

Dat is de belangrijkste regel. Hij maakt navigeren voorspelbaar en voorkomt dat
namespaces wegdrijven van de plek waar de bestanden staan.

---

## 1. Het patroon

```
<Product>.<Laag>
```

- `<Product>` is de naam van de applicatie of service, bijvoorbeeld `TodoApp` of
  `Billing`. Geen bedrijfsprefix (`Wheel7.`): dat voegt niets toe en maakt elke
  namespace langer.
- `<Laag>` is een van de lagen uit `ARCHITECTURE.md`: `Domain`, `Application`,
  `Infrastructure`, `Api`.
- Lagen zijn enkelvoud: `.Domain`, niet `.Domains`. Meervoud alleen voor iets dat
  echt een verzameling is (`.Migrations`).

Waarom niet kale namen als `Domain` of `Application`? Die botsen zodra twee producten
elkaar referencen of packages delen, `Application` schuurt met framework-types, en een
architectuurtest die op namespace `"Application"` filtert raakt meer dan je bedoelt.
Met `TodoApp.Application` is zo'n filter exact.

### Meerdere bounded contexts

Zit er meer dan één bounded context in één solution, dan komt de context vóór de laag:
`Shop.Ordering.Domain`, `Shop.Catalog.Domain`. Doe dit alleen als er echt meerdere
contexts zijn; één product met één context krijgt gewoon `<Product>.<Laag>`.

---

## 2. Standaardindeling

```
TodoApp.sln
├── src/
│   ├── TodoApp.Domain/
│   ├── TodoApp.Application/
│   ├── TodoApp.Infrastructure/
│   └── TodoApp.Api/
└── tests/
    ├── TodoApp.Application.UnitTests/
    ├── TodoApp.Api.IntegrationTests/
    └── TodoApp.ArchitectureTests/
```

Het entry point heet naar wat het is: `.Api`, `.Web`, `.Blazor`, `.Worker`. Een
solution met een API én een worker heeft dus twee entry points naast elkaar.

### Testprojecten

Testprojecten krijgen de naam van het project dat ze testen, plus een suffix dat zegt
welk soort test het is:

| Suffix | Test | Voorbeeld |
|---|---|---|
| `.UnitTests` | één class in isolatie, dependencies gemockt | `TodoApp.Application.UnitTests` |
| `.IntegrationTests` | de echte keten, met host en database | `TodoApp.Api.IntegrationTests` |
| `.ArchitectureTests` | laagregels en conventies over de hele solution | `TodoApp.ArchitectureTests` |

Suffix en geen prefix, zodat een testproject in de Solution Explorer naast het project
staat dat het test. Een kaal `.Tests` is te weinig zodra je unit- én integratietests
hebt, en dat moment komt altijd.

De architectuurtests testen niet één laag maar de solution, dus daar staat geen laag
in de naam.

---

## 3. Optionele projecten

Maak deze alleen aan als de situatie erom vraagt. Een project dat leeg begint "voor
later" wordt een vergaarbak.

### `.Contracts`: gedeelde types met een .NET-client

De standaardindeling is API-only. Komt er een .NET-client bij die dezelfde API
aanroept (Blazor WebAssembly, een console-tool, een andere service), dan hebben beide
kanten dezelfde request- en response-types nodig. Die client mag `.Application` en
`.Domain` niet referencen: dan komen entities en repositories in de browser terecht.

Daar is `.Contracts` voor. De naam zegt wat erin hoort, en daarmee ook wat niet:

- **Wel**: request- en response-types, de enums die ze dragen, constanten die meereizen
  (routes, headernamen).
- **Niet**: services, extension methods, helpers, interfaces van de applicatielaag.

`.Contracts` heeft geen enkele projectreferentie. De client referencet `.Contracts` en
niets anders.

**Wanneer wel, wanneer niet.** Een React-frontend is geen reden voor `.Contracts`: die
praat JSON en leest het contract uit de OpenAPI-specificatie, zie
[`../general/api-contracts.md`](../general/api-contracts.md). Pas bij een tweede
.NET-project dat de API aanroept voer je `.Contracts` in, niet eerder.

**Gevolg voor het CQRS-patroon.** `ARCHITECTURE.md` bindt het command uit
`.Application` rechtstreeks als request body in het endpoint. Zodra `.Contracts`
bestaat kan dat niet meer, want de client mag `.Application` niet kennen. Dan:

1. Request-types komen in `.Contracts` (`CreateTodoRequest`).
2. Response-types verhuizen van `.Application` naar `.Contracts` (`TodoDto` of
   `TodoResponse`).
3. Het endpoint mapt request naar command, in één regel. Dat is nog steeds "geen
   logica": het is de enige plek waar de twee elkaar raken.
4. `.Application` referencet `.Contracts` (mag, want `.Contracts` heeft geen
   dependencies). `.Domain` referencet `.Contracts` niet.

Voeg dan ook een architectuurtest toe: `.Contracts` heeft geen dependency op een
andere laag.

### `.Persistence` en `.Migrations`

Migrations horen bij persistentie. Zolang alleen de applicatie ze uitvoert staan ze in
`.Infrastructure`, zoals `ARCHITECTURE.md` beschrijft.

Groeit `.Infrastructure` zo dat EF Core, HTTP-clients en messaging elkaar in de weg
zitten, splits dan `.Persistence` af voor alles wat met de database te maken heeft.

Wil je migrations ook los van de applicatie kunnen draaien (in een pipeline, door een
beheerder), geef ze dan een eigen uitvoerbaar project `.Migrations`. `.Persistence`
blijft dan een library.

Noem het project naar de taak, niet naar het tool: `.Migrations`, niet `.DbUp`,
`.FluentMigrator` of `.EfMigrations`. Zie ook de regel over technologienamen hieronder.

---

## 4. Namen die we niet gebruiken

| Naam | Waarom niet | Wat dan wel |
|---|---|---|
| `.Core` | Betekent in elke codebase iets anders: domein, applicatielaag of plumbing. Wij hebben al een naam per laag. | `.Domain` of `.Application`, afhankelijk van wat je bedoelt |
| `.Common`, `.Shared`, `.Utils`, `.Helpers` | Betekenen niets, dus alles past erin en het project vult zich. | Noem het naar de inhoud: `.Abstractions`, `.Logging`, `.Validation` |
| `.Extensions` | Extension methods zijn een taalfeature, geen categorie. Dit is hoe `.Utils` opnieuw wordt uitgevonden. | Zet ze bij wat ze uitbreiden: mapping bij de mapping, registratie in het project dat registreert, display-extensies in de UI |
| `.BLL`, `.DAL` | Dateren de codebase en zeggen niets tegen wie ze niet in 2010 heeft geleerd. | `.Application`, `.Infrastructure` |
| `.EntityFramework`, `.SqlServer`, `.RabbitMq` | Technologie in een projectnaam betekent projecten hernoemen als je het tool vervangt. | Technologie is een map binnen `.Infrastructure`: `Infrastructure/Data/`, `Infrastructure/Messaging/` |

Een extension method op een frameworktype die nergens bij past is meestal een teken dat
de logica ergens concreters hoort.

---

## 5. Checklist: nieuwe solution

1. Kies de productnaam. Kort, zonder bedrijfsprefix, en gelijk aan de naam van de
   repository als dat kan.
2. Maak de vier lagen aan als `<Product>.<Laag>` in `src/`. Zie hoofdstuk 3 van
   `ARCHITECTURE.md` voor de commando's.
3. Controleer dat assembly-naam, root-namespace en mapnaam gelijk zijn. `dotnet new`
   met `-n <Product>.<Laag> -o src/<Product>.<Laag>` regelt dat.
4. Maak testprojecten aan met het juiste suffix in `tests/`.
5. Voeg geen optioneel project toe totdat de situatie uit hoofdstuk 3 zich voordoet.
