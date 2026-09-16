# CI/CD: GitHub Actions

Hoe we bouwen, testen en uitrollen. Sluit aan op [`containers.md`](containers.md) voor
de image en [`database.md`](database.md) voor de migrations.

> Bouw één keer, deploy datzelfde artefact naar elke omgeving.

Opnieuw bouwen per omgeving betekent dat de versie op productie nooit precies de versie
is die je hebt getest. Compileren gebeurt dus één keer, en wat daarna reist is de image
plus de migratiebundel.

De README noemt tooling als de plek waar afspraken echt worden afgedwongen. Dit document
is die plek: een regel die niet in een workflow staat, wordt vroeg of laat genegeerd.

---

## 1. Indeling

Twee workflows, meer niet:

| Bestand | Draait op | Doet |
|---|---|---|
| `.github/workflows/ci.yml` | elke pull request en push naar `main` | bouwen, testen, controleren |
| `.github/workflows/cd.yml` | push naar `main` | image bouwen, migrations draaien, uitrollen |

Splits pas verder op als er echt iets bij komt. Vijf workflows die elkaar aanroepen
leest niemand meer.

---

## 2. CI

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - run: dotnet restore
      - run: dotnet build --no-restore -c Release
      - run: dotnet test --no-build -c Release --logger trx --results-directory TestResults

      - name: Kwetsbare packages
        run: |
          dotnet list package --vulnerable --include-transitive > vuln.txt
          cat vuln.txt
          ! grep -q "has the following vulnerable packages" vuln.txt
```

Drie dingen die makkelijk misgaan:

- **`submodules: recursive` is verplicht.** Onze projecten halen deze standaarden binnen
  als submodule in `.standards`. Zonder deze regel is die map leeg in CI. Is de
  submodule-repo privé, dan heeft de checkout ook een token met leesrechten nodig; de
  standaard `GITHUB_TOKEN` komt niet buiten de eigen repository.
- **`dotnet list package --vulnerable` geeft exitcode 0**, ook als er kwetsbaarheden
  zijn. Zonder de `grep` erachter is die stap decoratie. Dit vult meteen een van de
  open punten in [`../general/security.md`](../general/security.md).
- **`concurrency` met `cancel-in-progress`.** Anders draaien er drie builds van dezelfde
  branch naast elkaar als je snel achter elkaar pusht.

Pin actions op een major-tag zoals hierboven, of op een commit-SHA als je strenger wilt
zijn. Controleer bij het aanmaken van een workflow welke major actueel is.

---

## 3. Integratietests

Gebruik **Testcontainers** met hetzelfde Postgres-image als in
[`containers.md`](containers.md). De test start de database zelf, dus de workflow heeft
geen service-container nodig en het werkt lokaal precies hetzelfde. Op de
GitHub-runners is Docker aanwezig.

Dit beslist de openstaande vraag in [`../dotnet/testing.md`](../dotnet/testing.md): geen
in-memory provider. Die kent geen constraints, geen transacties en geen SQL, dus tests
slagen daar terwijl productie faalt.

---

## 4. CD

```yaml
name: CD

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          file: src/TodoApp.Api/Dockerfile
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: image
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v5
        with:
          submodules: recursive

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Migratiebundel bouwen
        run: |
          dotnet ef migrations bundle \
            --project src/TodoApp.Infrastructure \
            --startup-project src/TodoApp.Api \
            --self-contained -r linux-x64 -o migrate

      - name: Migrations uitvoeren
        env:
          CONNECTION: ${{ secrets.DB_CONNECTION_MIGRATIONS }}
        run: ./migrate --connection "$CONNECTION"

      # Hierna de nieuwe image uitrollen. Hoe, hangt af van waar het draait.
```

De volgorde is niet vrijblijvend: **migrations eerst, dan de nieuwe versie**. Tijdens een
deploy draaien oud en nieuw even naast elkaar, dus het schema moet met allebei overweg
kunnen. Daarom de tweestapsaanpak voor breaking changes uit
[`database.md`](database.md).

`environment: production` geeft je in GitHub verplichte goedkeurders en aparte secrets
per omgeving. Zet dat aan voordat er iets echt live staat.

---

## 5. Secrets

- Geen langlevende cloudsleutels in GitHub-secrets. Gebruik OIDC met een federated
  credential, zodat een run een kortlevend token krijgt. Daarvoor staat
  `id-token: write` in de permissions.
- Secrets per omgeving via GitHub Environments, niet allemaal op repo-niveau.
- De migratiegebruiker is een andere dan de applicatiegebruiker. Alleen de deploy-job
  kent de eerste.
- `permissions` staat expliciet en zo krap mogelijk in elke workflow. De standaard is
  ruimer dan je wilt.

---

## 6. Wat groen moet zijn voordat je mergt

Zet deze als verplichte checks in de branch protection van `main`:

- build slaagt
- alle tests slagen
- geen kwetsbare packages

Dit hoort bij de open punten in [`../general/git-workflow.md`](../general/git-workflow.md)
over branch protection en verplichte checks. Zolang die niet ingevuld zijn, is de
workflow wel aanwezig maar blokkeert hij niets.

---

## 7. Checklist: nieuwe repository

1. `ci.yml` en `cd.yml` overnemen en de projectnamen aanpassen.
2. `submodules: recursive` in elke checkout.
3. Branch protection op `main` met de checks uit hoofdstuk 6.
4. Environment `production` met goedkeurders en eigen secrets.
5. OIDC ingericht voor de deploy, geen statische sleutels.
6. Eerste deploy handmatig meekijken, inclusief de migratiestap.
