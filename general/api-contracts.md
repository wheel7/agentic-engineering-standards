# API-contracten

> **Nog in te vullen door het team.** Onderstaande koppen geven de structuur; de
> TODO's zijn de beslissingen die we nog moeten nemen.

Gedeelde afspraken tussen frontend en backend. Dit bestand is bewust
stack-onafhankelijk: beide kanten moeten zich eraan houden, anders is het geen contract.

Gaat het om hóe je een endpoint bouwt, dan hoort dat in
[`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md). Hier staat alleen wat er
over de lijn gaat.

---

## 1. Naamgeving van endpoints

- TODO: meervoud of enkelvoud in paden? (`/todos/{id}` of `/todo/{id}`)
- TODO: casing in paden: kebab-case, camelCase?
- TODO: hoe diep nesten we resources? (`/orders/{id}/lines` of een platte structuur)
- TODO: hoe modelleren we acties die geen CRUD zijn? (`POST /todos/{id}/complete`)
- TODO: versioneren we, en zo ja: in het pad, een header, of niet?

## 2. Foutformaat

- TODO: gebruiken we RFC 7807 (`application/problem+json`)? Dat sluit aan bij
  `Results.Problem()` in .NET, dus dat ligt voor de hand.
- TODO: vaste velden afspreken (bijvoorbeeld `type`, `title`, `status`, `detail`,
  `traceId`).
- TODO: hoe geven we validatiefouten per veld terug?
- TODO: welke statuscodes gebruiken we waarvoor? (400 vs 422, 401 vs 403, 404 vs 204)
- TODO: geven we een foutcode terug die de frontend kan vertalen, of alleen tekst?

## 3. Datums en tijden

- TODO: vastleggen dat alles UTC is en in ISO 8601 gaat (`2026-09-16T14:30:00Z`).
  De .NET-kant gebruikt al `CreatedAtUtc`, dus dat sluit aan.
- TODO: hoe geven we een datum zonder tijd door?
- TODO: waar gebeurt de omzetting naar lokale tijd - alleen in de frontend?

## 4. Paginering

- TODO: welke stijl? Page/size, offset/limit, of cursor-based?
- TODO: vaste naamgeving van de parameters.
- TODO: hoe ziet het antwoord eruit - alleen items, of ook totaal en volgende pagina?
- TODO: standaard- en maximumwaarde voor de paginagrootte.
- TODO: afspraken over sorteren en filteren.

## 5. Overig

- TODO: casing van JSON-properties (camelCase ligt voor de hand voor een
  JavaScript-frontend).
- TODO: laten we `null` weg of sturen we die expliciet mee?
- TODO: publiceren we een OpenAPI-specificatie, en genereert de frontend daar zijn
  client uit?

## 6. Open punten

- [ ] Wie bewaakt het contract als frontend en backend in verschillende repo's zitten?
- [ ] Hoe voeren we een breaking change door zonder de frontend te breken?
- [ ] Komt er een .NET-client (Blazor, console, andere service)? Dan gelden de regels
      voor `.Contracts` uit [`../dotnet/solution-layout.md`](../dotnet/solution-layout.md).
      Een React-frontend leest het contract uit OpenAPI en heeft dat project niet nodig.
