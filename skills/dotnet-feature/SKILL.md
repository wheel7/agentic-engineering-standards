---
name: dotnet-feature
description: Voegt een nieuwe feature (command of query) toe aan een .NET 8 API volgens onze CQRS-architectuur zonder MediatR. Gebruik deze skill wanneer er een nieuw endpoint, command, query of handler bij moet komen in een project met de lagen Domain, Application, Infrastructure en Api - bijvoorbeeld "voeg een endpoint toe om een todo af te ronden" of "maak een query om orders per klant op te halen". Niet gebruiken voor het opzetten van een nieuw project of voor wijzigingen die alleen bestaande code aanpassen.
---

# Nieuwe .NET-feature toevoegen

Volgt de architectuur uit `@.standards/dotnet/ARCHITECTURE.md`. Lees dat document als
je twijfelt over een keuze; deze skill is de uitvoering van de checklist uit
hoofdstuk 6, niet een vervanging ervan.

`<Product>` in de paden hieronder is de productnaam van de solution, bijvoorbeeld
`TodoApp` in `src/TodoApp.Domain/`. Zie `@.standards/dotnet/solution-layout.md`.

## Vooraf

1. Lees `.standards/dotnet/ARCHITECTURE.md` (met name hoofdstuk 5, conventies).
2. Lees het `CLAUDE.md` van het project voor repo-specifieke afwijkingen.
3. Bepaal: is dit een **command** (wijzigt state) of een **query** (leest state)?
   Beide in één handler is geen optie - splits dan.
4. Kijk naar een bestaande feature in dezelfde repo en volg die stijl. De conventies
   hieronder gelden, maar het bestaande project wint bij twijfel.

## Stappen

Doorloop ze in deze volgorde. Sla stappen die niet van toepassing zijn over, maar
benoem wel dát je ze overslaat.

### 1. Domain

Voeg de entity of het gedrag toe in `src/<Product>.Domain/<Feature>/`.

- Private setters; state wijzigt alleen via methodes met een betekenisvolle naam.
- Geen publieke parameterloze constructor (wel een private, voor EF Core).
- Geen verwijzingen naar andere lagen. Domain hangt nergens van af.

### 2. Repository-interface

Voeg de benodigde methode toe aan `I<Entity>Repository` in `src/<Product>.Application/<Feature>/`.

- Elke methode krijgt een `CancellationToken`.
- Moet een command een bestaande entity wijzigen, dan is een aparte methode nodig
  **zonder** `AsNoTracking()` - anders worden wijzigingen niet opgeslagen.

### 3. Command of query + handler

In `src/<Product>.Application/<Feature>/`:

- Naamgeving: `{Werkwoord}{Entity}Command` / `Get{Entity}By{Criterium}Query`, handler
  is `{Naam}Handler`.
- Alles `sealed`, properties met `init`.
- Eén publieke methode: `HandleAsync(request, CancellationToken)`.
- Constructor-injectie van de repository, geen service locator.
- Command: roept `SaveChangesAsync` aan. Query: nooit, en gebruikt `AsNoTracking()`.

### 4. DTO

Hergebruik `{Entity}Dto` als die er is, anders maak je hem in dezelfde map.
Geef nooit een domain-entity terug vanuit de API.

### 5. Infrastructure

Implementeer de repository-methode in `src/<Product>.Infrastructure/<Feature>/`.

### 6. EF-configuratie en migration

Alleen als het datamodel wijzigt:

```bash
dotnet ef migrations add <Naam> --project src/<Product>.Infrastructure --startup-project src/<Product>.Api
```

Laat het uitvoeren van `database update` aan de gebruiker.

### 7. Handler registreren

In `src/<Product>.Api/Program.cs`: `builder.Services.AddScoped<{Naam}Handler>();`

### 8. Endpoint toevoegen

Ook in `Program.cs`. Het endpoint bevat **geen logica**: request binnen, handler
aanroepen, HTTP-resultaat terug.

- Command: `Results.Created(...)` of `Results.NoContent()`.
- Query: `Results.Ok(...)`, of `Results.NotFound()` als het resultaat `null` is.
- Geef de `CancellationToken` door.

### 9. Unit test

Zie `@.standards/dotnet/testing.md`. Mock de repository met Moq, roep `HandleAsync`
direct aan, controleer het resultaat **en** verifieer de interacties (bij een command:
is er opgeslagen; bij een query: is er níet opgeslagen).

## Afronden

- Draai `dotnet build` en `dotnet test`.
- Loop de conventietabel uit hoofdstuk 5 van ARCHITECTURE.md na.
- Zijn er architectuurtests, dan moeten die groen zijn - die bewaken de laagregels.
- Meld welke bestanden je hebt toegevoegd of gewijzigd, en welke stappen je hebt
  overgeslagen en waarom.
