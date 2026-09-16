# Database: PostgreSQL

Hoort bij [`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md). Die beschrijft hoe
de persistentielaag in de code zit; dit document beschrijft de afspraken over de
database zelf en over migrations.

> PostgreSQL is de standaard. Wijkt een project daarvan af, dan staat dat met de reden
> in het `CLAUDE.md` van dat project.

Waarom Postgres: het draait overal hetzelfde (lokaal in een container, in CI, op Azure
en op een eigen server), er is geen licentie nodig, en de beheerde varianten zijn
goedkoper dan die van SQL Server. Lokaal draaien staat in
[`containers.md`](containers.md).

---

## 1. Packages

```bash
dotnet add src/TodoApp.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL --version 10.0.*
dotnet add src/TodoApp.Infrastructure package EFCore.NamingConventions --version 10.0.*
```

De naamgevingspackage volgt de EF Core-major. Loopt hij achter, zet snake_case dan
tijdelijk handmatig in `OnModelCreating` en werk de package bij zodra hij uit is.

Registratie in `Program.cs`:

```csharp
builder.Services.AddDbContext<TodoDbContext>(options =>
    options
        .UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection"))
        .UseSnakeCaseNamingConvention());
```

---

## 2. Naamgeving in de database

Tabellen en kolommen zijn **snake_case**: `created_at_utc`, niet `CreatedAtUtc`.

Dit is geen smaak. Postgres vouwt niet-aangehaalde namen om naar kleine letters, dus een
PascalCase-kolom moet je in elke query aanhalen. Zonder deze conventie wordt
`select * from "Todos" where "IsCompleted" = false` de norm en klaagt iedereen die ooit
handmatig in de database kijkt. In C# blijft alles gewoon PascalCase; de package vertaalt.

- Tabelnamen meervoud (`todos`), kolomnamen enkelvoud.
- Geen prefixen als `tbl_` of `fk_` in tabelnamen.
- Indexen en constraints laten we door EF Core benoemen, tenzij er een goede reden is.

---

## 3. Types en kolommen

| Onderwerp | Afspraak | Waarom |
|---|---|---|
| Datum en tijd | `DateTime` met `Kind.Utc`, kolomtype `timestamptz` | Npgsql gooit een exception bij een niet-UTC `DateTime`, dus de fout komt meteen naar boven in plaats van maanden later |
| Datum zonder tijd | `DateOnly`, kolomtype `date` | Geen middernacht-verschuiving door tijdzones |
| Primaire sleutel | `Guid` via `Guid.CreateVersion7()` | Zie hieronder |
| Bedragen | `decimal`, kolomtype `numeric(19,4)` | `float` en `double` ronden geld verkeerd af |
| Tekst | `text`, met `HasMaxLength` voor validatie | In Postgres is `varchar(n)` niet sneller dan `text`; de lengte is een regel, geen optimalisatie |
| Enums | Opslaan als string via `HasConversion<string>()` | Leesbaar in de database, en hernummeren van de enum breekt niets |

### Sleutels

Gebruik `Guid.CreateVersion7()`, niet `Guid.NewGuid()`. Een versie 7-GUID begint met een
tijdstempel en is daardoor oplopend. Willekeurige versie 4-GUID's schrijven overal in de
index, wat de index opblaast en inserts trager maakt naarmate de tabel groeit. Dezelfde
sleutel, geen extra kosten.

```csharp
Id = Guid.CreateVersion7();
```

Een oplopende `int` mag ook, maar niet voor iets dat in een URL of een API-response
terechtkomt: dan lekt het aantal records en kun je andermans records raden.

### Standaardkolommen

Elke entity die iets vastlegt wat later te verklaren moet zijn krijgt minimaal
`CreatedAtUtc`. Voeg `ModifiedAtUtc` toe zodra iets gewijzigd kan worden.

Gelijktijdige wijzigingen vang je af met de systeemkolom `xmin`, het Postgres-equivalent
van `rowversion`:

```csharp
todo.Property<uint>("xmin")
    .IsRowVersion()
    .HasColumnName("xmin");
```

Doe dit alleen waar twee gebruikers echt tegelijk dezelfde rij kunnen wijzigen. Overal
toevoegen levert alleen maar conflicten op die niemand afhandelt.

### Zoeken op tekst

Postgres is hoofdlettergevoelig. Zoeken op naam of e-mail doe je met `ILIKE`, of je zet
de kolom op een non-deterministische collatie. Kies per geval en leg het vast in het
project; `ToLower()` in een `Where` maakt de index onbruikbaar.

---

## 4. Migrations

- **Eén migration per pull request.** Meer betekent dat de PR te groot is.
- **Nooit een migration bewerken die al ergens is toegepast.** Ook niet op test.
  Corrigeren doe je met een nieuwe migration.
- **Terugrollen doe je vooruit.** Een `Down`-methode die niemand ooit heeft gedraaid is
  geen vangnet. Bij een fout op productie gaat er een nieuwe migration overheen.
- **Naamgeving**: beschrijf de wijziging, niet de datum. `AddTodoDueDate`, niet
  `Update3`.

### Breaking changes in twee stappen

Een kolom hernoemen of weghalen breekt de draaiende versie tijdens een deploy, want even
draaien oud en nieuw naast elkaar. Splits het daarom:

1. **Uitbreiden.** Voeg de nieuwe kolom toe, laat de oude staan, en schrijf naar allebei.
   Deploy.
2. **Opruimen.** Verwijder de oude kolom en het dubbele schrijven. Deploy.

Twee PR's, twee releases. Dat is de prijs van deployen zonder downtime.

### Migrations uitvoeren

**Niet** bij het opstarten van de applicatie. `Database.Migrate()` in `Program.cs` is
verleidelijk, maar zodra er twee instanties draaien beginnen die tegelijk, en je hebt
geen enkele controle over het moment. Bovendien heeft de applicatie dan permanent
rechten om het schema te wijzigen.

Wel: als aparte stap in de pipeline, vóór de nieuwe versie live gaat. Zie
[`ci-cd.md`](ci-cd.md). Bouw daarvoor een migratiebundel:

```bash
dotnet ef migrations bundle \
  --project src/TodoApp.Infrastructure \
  --startup-project src/TodoApp.Api \
  --self-contained -r linux-x64 \
  -o migrate
```

Dat levert één uitvoerbaar bestand op dat je met een connectionstring draait. De runner
heeft daarvoor geen SDK nodig en je kunt precies zien wat er wordt uitgevoerd.

---

## 5. Toegang en rechten

- De applicatie draait met een gebruiker die alleen data mag lezen en schrijven, niet
  het schema mag wijzigen. Migrations draaien onder een aparte gebruiker.
- Connectionstrings staan nooit in de repo. Lokaal in user-secrets of in het
  compose-bestand, daarbuiten in de secret store van de omgeving. Zie
  [`../general/security.md`](../general/security.md).
- Ontwikkelaars werken niet op de productiedatabase. Is meekijken nodig, dan op een
  restore.

---

## 6. Back-ups

Een back-up die nooit is teruggezet is een aanname, geen back-up. Per project vastleggen:

- Hoe vaak, en hoe lang bewaard.
- Waar de kopie staat, en dat die ergens anders staat dan de database zelf.
- Wanneer de restore voor het laatst is getest, en door wie.

---

## 7. Checklist: nieuwe entity

1. Snake_case volgt automatisch; geen kolomnamen handmatig instellen.
2. Sleutel via `Guid.CreateVersion7()`.
3. Tijdstempels in UTC, als `timestamptz`.
4. Bedragen als `numeric(19,4)`.
5. `HasMaxLength` op tekstvelden die een grens hebben.
6. Concurrency token alleen waar gelijktijdig wijzigen echt voorkomt.
7. Eén migration, met een beschrijvende naam.
8. Breekt de wijziging de draaiende versie? Splits hem in uitbreiden en opruimen.
