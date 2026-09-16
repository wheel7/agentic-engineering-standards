---
name: react-component
description: CONCEPT - voegt een nieuw React-component toe volgens onze (nog niet vastgestelde) frontendarchitectuur. Gebruik deze skill wanneer er een nieuw component, scherm of feature bij moet komen in een React-project, bijvoorbeeld "maak een TodoList-component" of "voeg een scherm toe om orders te tonen". De afspraken hierachter zijn nog niet door het team bevestigd; controleer altijd eerst hoe het bestaande project het doet.
---

# Nieuw React-component toevoegen

> **CONCEPT - nog te reviewen door het team.**
>
> De onderliggende afspraken in `@.standards/react/ARCHITECTURE.md` en
> `@.standards/react/testing.md` zijn nog concept. Volg bij een conflict altijd wat
> het bestaande project al doet, en meld de afwijking.

## Vooraf

1. Lees `.standards/react/ARCHITECTURE.md` en `.standards/react/testing.md`.
2. Lees het `CLAUDE.md` van het project - daar staat welke keuzes dit project al
   gemaakt heeft (state, styling, datatoegang).
3. Kijk naar een vergelijkbaar bestaand component en volg die stijl. Dat weegt zwaarder
   dan deze skill, zolang de standaard concept is.
4. Bepaal waar het component hoort: binnen een feature (`src/features/<feature>/`) of
   gedeeld (`src/components/`). Bij twijfel: begin binnen de feature en verplaats het
   pas als een tweede feature het nodig heeft.

## Stappen

### 1. Plaatsing en naamgeving

- Component in PascalCase: `TodoList.tsx`.
- Hook in camelCase met `use`-prefix: `useTodos.ts`.
- Een feature importeert niet rechtstreeks uit een andere feature.

### 2. Props en types

- TypeScript, expliciete props-type, geen `any`.
- Types van de feature in `types.ts`; alleen exporteren wat naar buiten nodig is.

### 3. Presentatie en data scheiden

- Houd het component zelf zo veel mogelijk presentatie: props erin, UI eruit.
- Datatoegang (fetch, caching) in een hook of in `api.ts` van de feature, niet
  verspreid door het component.
- Gebruik het datatoegangsmechanisme dat dit project al hanteert.

### 4. Toegankelijkheid

- Gebruik semantische elementen (`button`, niet een `div` met `onClick`).
- Zorg dat interactieve elementen een toegankelijke naam hebben - dat is ook waar de
  tests op selecteren.

### 5. Test

Zie `@.standards/react/testing.md`. Test wat de gebruiker ziet en doet:

- Selecteer op rol, label of tekst - niet op CSS-classes of implementatiedetails.
- Dek minimaal de belangrijkste interactie af.

### 6. Exporteren

Neem het component op in de `index.ts` van de feature als het daarbuiten gebruikt wordt.

## Afronden

- Draai de linter, de typecheck (`tsc --noEmit`) en de tests van het project.
- Meld welke bestanden je hebt toegevoegd.
- Kwam je een keuze tegen die nog openstaat in `react/ARCHITECTURE.md` (state,
  styling, formulieren)? Meld dan wat je gekozen hebt en waarom, zodat het team dat
  kan meenemen in de review van de standaard.
