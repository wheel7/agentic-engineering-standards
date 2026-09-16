# Engineering Standards

Gedeelde ontwikkelstandaarden en AI-skills van ons team, voor zowel .NET- als
React-projecten.

Het idee: generieke kennis staat **één keer** hier, en niet verspreid over tientallen
projectrepo's. Een projectrepo bevat alleen een dun `CLAUDE.md` met wat écht
repo-specifiek is (domein, database, lokaal draaien) en importeert de rest uit deze repo.

---

## Structuur

```
engineering-standards/
├── README.md
├── CODEOWNERS
├── general/            Stack-onafhankelijke afspraken
│   ├── git-workflow.md
│   ├── security.md
│   └── api-contracts.md
├── dotnet/             .NET-specifiek
│   ├── ARCHITECTURE.md
│   ├── solution-layout.md
│   └── testing.md
├── react/              React-specifiek
│   ├── ARCHITECTURE.md
│   └── testing.md
├── skills/             AI-skills (stap-voor-stap werkinstructies)
│   ├── dotnet-feature/SKILL.md
│   └── react-component/SKILL.md
└── templates/          Voorbeeld-CLAUDE.md om naar een project te kopiëren
    ├── CLAUDE.dotnet.md
    └── CLAUDE.react.md
```

Uitgangspunt: **één repo voor alle stacks**. `general/` voor wat overal geldt, daarnaast
een map per stack. Zo hoeft een project maar één submodule binnen te halen, ook als het
een full-stack repo is.

---

## Gebruik in een project

### 1. Toevoegen als submodule

De standaarden komen in de map `.standards`:

```bash
git submodule add https://github.com/wheel7/engineering-standards.git .standards
git commit -m "Engineering standards toevoegen als submodule"
```

### 2. Clonen van een project dat de submodule al gebruikt

```bash
git clone --recurse-submodules <url-van-het-project>
```

Al gecloned zonder `--recurse-submodules`? Dan alsnog:

```bash
git submodule update --init --recursive
```

### 3. Bijwerken naar de laatste standaarden

Updaten gebeurt **bewust**, via een PR in het project — niet automatisch. Zo ziet het
team in de diff welke afspraken er veranderd zijn:

```bash
git submodule update --remote .standards
git add .standards
git commit -m "Standaarden bijwerken"
```

Een submodule staat vastgepind op een specifieke commit. Een project loopt dus nooit
onverwacht achter of voor op de standaarden: het verandert pas als je het zelf doet.

---

## Importeren in je project-`CLAUDE.md`

Het `CLAUDE.md` in de root van je project importeert alleen de bestanden die het nodig
heeft, met `@`-imports:

```markdown
@.standards/general/git-workflow.md
@.standards/dotnet/ARCHITECTURE.md
@.standards/dotnet/testing.md
```

Importeer bewust en niet alles: een React-only project heeft `dotnet/` niet nodig.

Kopieer als startpunt het passende bestand uit [`templates/`](templates/) naar de root
van je project als `CLAUDE.md`:

- [`templates/CLAUDE.dotnet.md`](templates/CLAUDE.dotnet.md)
- [`templates/CLAUDE.react.md`](templates/CLAUDE.react.md)

Vul daarna de sectie "Specifiek voor deze repo" in. Alles wat generiek blijkt te zijn
hoort niet daar, maar hier in deze repo.

---

## Skills

De map [`skills/`](skills/) bevat werkinstructies voor terugkerende taken, zoals het
toevoegen van een feature volgens onze architectuur. Ze zijn bedoeld voor Claude Code,
maar lezen prima als checklist voor mensen.

Gebruik ze door er in je project-`CLAUDE.md` naar te verwijzen, of door de map te
koppelen aan je lokale skills-configuratie.

---

## Instructies zijn richtlijnen, tooling is de handhaving

Een document dat zegt "handlers zijn `sealed`" wordt vroeg of laat genegeerd. Een test
die faalt niet.

Daarom: dit zijn **richtlijnen** die uitleggen wat we doen en waarom. De echte
handhaving komt van tooling:

- **architectuurtests** (NetArchTest) voor laagregels en conventies
- **analyzers** en `.editorconfig` voor codestijl
- **gedeelde CI** die beide draait op elke PR

Dit is nog niet uitgewerkt. Zolang dat zo is, is een review het vangnet — zie
[`dotnet/testing.md`](dotnet/testing.md) voor wat er al wél staat.

---

## Bijdragen

Eigenaarschap per map staat in [`CODEOWNERS`](CODEOWNERS). Wijzigingen gaan via een PR,
met review van de eigenaar van die map.

Twee vuistregels:

- Iets dat in meerdere projecten geldt hoort hier. Iets dat maar in één project geldt
  hoort in het `CLAUDE.md` van dat project.
- Leg uit *waarom*, niet alleen *wat*. Een regel zonder reden wordt niet gevolgd.
