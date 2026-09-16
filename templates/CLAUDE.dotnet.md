# CLAUDE.md - <PROJECTNAAM>

<!--
Kopieer dit bestand naar de root van je .NET-project als CLAUDE.md.

Voorwaarde: de standaarden zijn toegevoegd als submodule in .standards
    git submodule add https://github.com/wheel7/engineering-standards.git .standards

Houd dit bestand DUN. Alles wat ook voor andere projecten geldt hoort niet hier,
maar in de engineering-standards repo. Vervang alle <placeholders> hieronder.
-->

## Standaarden

Deze gelden voor dit project. Ze staan in de submodule `.standards`; wijzigingen
daaraan gaan via een PR in die repo, niet hier.

@.standards/general/git-workflow.md
@.standards/general/security.md
@.standards/general/api-contracts.md
@.standards/dotnet/ARCHITECTURE.md
@.standards/dotnet/solution-layout.md
@.standards/dotnet/testing.md
@.standards/ops/database.md
@.standards/ops/containers.md
@.standards/ops/ci-cd.md

Bijwerken naar de laatste versie:

```bash
git submodule update --remote .standards
```

Doe dat in een eigen PR, zodat de wijziging in de standaarden zichtbaar is in de diff.

---

## Specifiek voor deze repo

### Solution

<!-- De productnaam bepaalt de projectnamen: <Product>.Domain, <Product>.Api, enz.
     Zie .standards/dotnet/solution-layout.md. -->

- **Productnaam**: <Product>
- **Entry points**: <Product>.Api <en eventueel .Worker, .Blazor>

### Domein

<!-- Waar gaat deze applicatie over? Welke kernbegrippen moet je kennen?
     Bijvoorbeeld: entiteiten, hun onderlinge relatie, en de belangrijkste
     businessregels die niet uit de code af te lezen zijn. -->

- **Kernbegrippen**: <...>
- **Belangrijkste businessregels**: <...>
- **Externe systemen waar we mee koppelen**: <...>

### Database

<!-- Welke database, waar draait die, hoe kom je erbij? -->

- **Type**: PostgreSQL <major-versie, gelijk aan docker-compose.yml en productie>
- **Connectionstring lokaal**: komt uit `docker-compose.yml`; wijkt dit project daarvan
  af, noteer dan hier hoe.
- **Migrations**:

```bash
dotnet ef migrations add <Naam> --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
dotnet ef database update       --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
```

- **Testdata / seeding**: <...>

### Lokaal draaien

```bash
# Alles in containers, inclusief database:
docker compose up

# Of alleen de database in een container en de API erbuiten:
docker compose up -d db
dotnet run --project src/<Product>.Api
```

- **URL lokaal**: <https://localhost:xxxx>
- **Benodigde secrets**: <welke, en hoe zet je ze klaar - bijv. dotnet user-secrets>
- **Afhankelijkheden die moeten draaien**: <database, message broker, mock-services>
- **Tests draaien**: `dotnet test`

### Hosting

<!-- Waar draait dit, hoe komt het daar, en hoe krijg je het terug als het misgaat?
     De generieke afspraken staan in .standards/ops/; hier alleen wat per project
     verschilt. -->

- **Draait op**: <Azure Container Apps / eigen server / ...>
- **Omgevingen**: <test, productie - en de URL's>
- **Deploy**: <welke workflow, en wat triggert hem>
- **Terugrollen**: <hoe, en wie mag dat>
- **Logs en alerting**: <waar kijk je als het misgaat>
- **Secrets**: <waar staan ze, wie beheert ze>

### Afwijkingen van de standaard

<!-- Wijkt dit project bewust af van .standards? Noteer dat hier MET de reden.
     Zonder reden wordt het later per ongeluk "opgelost". Staat hier niets,
     dan geldt de standaard onverkort. -->

- <Geen bekende afwijkingen.>

### Overig

<!-- Valkuilen, historisch gegroeide rariteiten, dingen waar iedereen over struikelt. -->

- <...>
