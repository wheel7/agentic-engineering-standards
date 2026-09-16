# CLAUDE.md - <PROJECTNAAM>

<!--
Kopieer dit bestand naar de root van je React-project als CLAUDE.md.

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
@.standards/react/ARCHITECTURE.md
@.standards/react/testing.md

> Let op: de React-standaarden zijn op dit moment **concept**. Wijkt dit project ervan
> af, noteer dat dan hieronder - dat is waardevolle input voor de review.

Bijwerken naar de laatste versie:

```bash
git submodule update --remote .standards
```

Doe dat in een eigen PR, zodat de wijziging in de standaarden zichtbaar is in de diff.

---

## Specifiek voor deze repo

### Domein

<!-- Waar gaat deze applicatie over? Welke schermen en begrippen moet je kennen? -->

- **Kernbegrippen**: <...>
- **Belangrijkste schermen / flows**: <...>

### Techniekkeuzes

<!-- De React-standaard laat deze keuzes bewust open. Vul hier in wat DIT project doet. -->

- **Framework / bundler**: <Vite / Next.js / ...>
- **Routing**: <...>
- **Server-state / datatoegang**: <TanStack Query / fetch in hooks / ...>
- **Client-state**: <...>
- **Styling**: <Tailwind / CSS Modules / ...>
- **Formulieren en validatie**: <...>
- **Componentbibliotheek**: <...>

### Backend / API

- **API-url lokaal**: <http://localhost:xxxx>
- **Waar staat de backend-repo**: <...>
- **Authenticatie**: <hoe logt de frontend in, waar komt het token vandaan>
- **Wordt de client gegenereerd uit OpenAPI?**: <ja/nee, en met welk commando>

### Lokaal draaien

```bash
# <Vul de commando's in die hier echt werken>
npm install
npm run dev
```

- **URL lokaal**: <http://localhost:5173>
- **Benodigde omgevingsvariabelen**: <welke, en waar zet je ze - bijv. .env.local>
- **Tests draaien**: `npm test`
- **Lint en typecheck**: `npm run lint` / `npx tsc --noEmit`

### Afwijkingen van de standaard

<!-- Wijkt dit project bewust af van .standards? Noteer dat hier MET de reden. -->

- <Geen bekende afwijkingen.>

### Overig

<!-- Valkuilen, historisch gegroeide rariteiten, dingen waar iedereen over struikelt. -->

- <...>
