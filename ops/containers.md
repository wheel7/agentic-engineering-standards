# Containers

Hoe we .NET-applicaties in een container verpakken en lokaal draaien. De database die
erbij hoort staat in [`database.md`](database.md); het bouwen en publiceren van images
in [`ci-cd.md`](ci-cd.md).

> Doel: een nieuwe collega kan de repo clonen, `docker compose up` draaien en heeft een
> werkende applicatie met database. Zonder installatiehandleiding.

---

## 1. Dockerfile

Eén Dockerfile per entry point, naast het project dat hij bouwt:
`src/TodoApp.Api/Dockerfile`. De buildcontext is de repository-root, want de build heeft
alle projecten nodig.

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Eerst alleen de projectbestanden: deze laag wordt hergebruikt zolang
# er geen package verandert.
COPY ["src/TodoApp.Domain/TodoApp.Domain.csproj",                 "src/TodoApp.Domain/"]
COPY ["src/TodoApp.Application/TodoApp.Application.csproj",       "src/TodoApp.Application/"]
COPY ["src/TodoApp.Infrastructure/TodoApp.Infrastructure.csproj", "src/TodoApp.Infrastructure/"]
COPY ["src/TodoApp.Api/TodoApp.Api.csproj",                       "src/TodoApp.Api/"]
RUN dotnet restore "src/TodoApp.Api/TodoApp.Api.csproj"

COPY . .
RUN dotnet publish "src/TodoApp.Api/TodoApp.Api.csproj" \
    -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled AS final
WORKDIR /app
COPY --from=build /app/publish .

USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "TodoApp.Api.dll"]
```

Waarom het er zo uitziet:

- **Twee stages.** De SDK-image is ruim een gigabyte en hoort niet op productie. Alleen
  het publicatieresultaat gaat mee naar de runtime-image.
- **Csproj-bestanden eerst.** Kopieer je meteen alles, dan draait `restore` bij elke
  codewijziging opnieuw. Zo alleen wanneer een package verandert.
- **`--no-restore` bij publish.** Anders herhaalt publish het restore-werk.
- **Chiseled runtime-image.** Geen shell, geen package manager, veel minder te patchen.
  Werkt niet als je in de container wilt debuggen; gebruik dan `10.0-noble` zonder
  chiseled.
- **`USER $APP_UID`.** De images hebben sinds .NET 8 een niet-root gebruiker ingebouwd.
  Draaien als root is nergens voor nodig.
- **Poort 8080.** Sinds .NET 8 de standaard, juist omdat een niet-root proces geen poort
  onder 1024 mag openen. Overschrijf dit niet.

Pin de image op de major-versie (`10.0`), niet op `latest`. Zo krijg je wel patches en
geen onverwachte .NET-upgrade.

---

## 2. .dockerignore

Verplicht, in de repository-root. Zonder dit bestand gaat `bin`, `obj` en
`node_modules` mee de buildcontext in. Dat is niet alleen traag: een `obj`-map van je
eigen machine kan de restore in de container laten mislukken met fouten die nergens op
slaan.

```gitignore
**/bin/
**/obj/
**/node_modules/
**/.vs/
**/.vscode/
**/.idea/
**/TestResults/
**/*.user
.git/
.github/
**/appsettings.Development.json
**/.env
Dockerfile*
docker-compose*.yml
```

---

## 3. Lokaal draaien met compose

`docker-compose.yml` in de repository-root:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_DB: todoapp
      POSTGRES_USER: todoapp
      POSTGRES_PASSWORD: localdev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U todoapp -d todoapp"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    build:
      context: .
      dockerfile: src/TodoApp.Api/Dockerfile
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__DefaultConnection: "Host=db;Port=5432;Database=todoapp;Username=todoapp;Password=localdev"
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

Aandachtspunten:

- **Pin de Postgres-major** op hetzelfde nummer als productie. Een minorverschil is
  prima, een major niet.
- **`condition: service_healthy`.** Zonder healthcheck start de API voordat Postgres
  verbindingen accepteert, en dan crasht hij bij de eerste query.
- **Named volume** voor de data. Een bind mount naar een Windows-map geeft
  rechtenproblemen.
- **Het wachtwoord hierboven is geen secret.** Het geldt alleen voor een wegwerpdatabase
  op je eigen machine en staat bewust in de repo. Elk ander wachtwoord hoort daar niet;
  zie [`../general/security.md`](../general/security.md).

Dit compose-bestand draait geen migrations. Doe dat lokaal zelf, zie
[`database.md`](database.md).

---

## 4. Configuratie

Alles wat per omgeving verschilt komt binnen als omgevingsvariabele. Geneste keys
schrijf je met een dubbele underscore, want een punt mag niet overal:

```
ConnectionStrings__DefaultConnection
Logging__LogLevel__Default
```

`appsettings.json` bevat alleen standaardwaarden die overal gelden. Geen
omgevingsspecifieke bestanden meebakken in de image: dezelfde image gaat naar test en
naar productie, alleen de variabelen verschillen.

---

## 5. Wat een container van de applicatie verwacht

- **Logs naar stdout**, gestructureerd als JSON. Niet naar bestanden: die zijn weg zodra
  de container weg is, en niemand komt erbij.
- **Health endpoints.** `/health/live` zegt of het proces nog leeft, `/health/ready` of
  hij verkeer aankan (inclusief databaseverbinding). Een orchestrator herstart op de
  eerste en stuurt verkeer op basis van de tweede. Eén gecombineerd endpoint leidt tot
  herstarts omdat de database even traag is.
- **Netjes afsluiten op SIGTERM.** ASP.NET Core doet dit zelf, maar lopende
  achtergrondtaken moeten de `CancellationToken` respecteren. Anders breekt een deploy
  verzoeken af.
- **Geen state in de container.** Geen uploads op schijf, geen sessies in het geheugen
  zonder gedeelde store. Een container kan elk moment vervangen worden.

---

## 6. Images en tags

- Tag op de git-sha: `ghcr.io/wheel7/todoapp:9f3c1ab`. Daarmee weet je altijd precies
  welke code draait.
- Gebruik `latest` niet om te deployen. Als extra tag naast de sha mag het.
- Eén image per entry point. Een `.Worker` naast een `.Api` is een eigen image, geen
  schakelaar in dezelfde.
- Dezelfde image gaat naar elke omgeving. Opnieuw bouwen per omgeving betekent dat je op
  productie iets draait wat je nergens hebt getest.

---

## 7. Frontend

Een React-applicatie hoort hier meestal niet in een container. Het bouwresultaat is een
map met statische bestanden; die zet je op een CDN of static hosting. Een container met
nginx ervoor is extra onderdeel om te patchen zonder dat het iets oplost.

Draait de frontend wél mee in compose voor lokale ontwikkeling, dan met de dev-server en
een bind mount, niet met een productiebuild.

---

## 8. Checklist

1. `Dockerfile` naast het entry point, buildcontext is de root.
2. `.dockerignore` aanwezig en bijgewerkt.
3. Runtime-image is chiseled en draait als niet-root op poort 8080.
4. `docker compose up` geeft een werkende applicatie met database.
5. Configuratie komt uit omgevingsvariabelen, niet uit meegebakken bestanden.
6. Logs gaan naar stdout, health endpoints bestaan.
7. Image is getagd op de git-sha.
