# Security

> **Nog in te vullen door het team.** Onderstaande koppen geven de structuur; de
> TODO's zijn de beslissingen die we nog moeten nemen.

Geldt voor alle projecten, ongeacht stack.

---

## 1. Secrets

- TODO: waar horen secrets thuis per omgeving (lokaal, test, productie)? Denk aan
  .NET user-secrets lokaal en een key vault daarbuiten.
- TODO: vastleggen dat secrets nooit in de repo komen - ook niet in `appsettings.json`,
  `.env`-bestanden of testdata.
- TODO: secret scanning aanzetten op de repo's.
- TODO: procedure als er tóch een secret gelekt is (intrekken, roteren, melden).
- TODO: hoe vaak roteren we sleutels en certificaten?

## 2. Authenticatie en autorisatie

- TODO: welke identity provider gebruiken we standaard?
- TODO: token-afspraken: type, levensduur, waar bewaar je ze in de frontend?
- TODO: autorisatiemodel: rollen, claims, of policies?
- TODO: hoe beveiligen we service-to-service verkeer?
- TODO: afspraken over CORS.

## 3. Afhankelijkheden

- TODO: gebruiken we Dependabot of Renovate voor updates?
- TODO: hoe snel moeten kwetsbaarheden opgelost zijn, per ernst?
- TODO: draaien we `dotnet list package --vulnerable` en `npm audit` in CI?
- TODO: mag iedereen een nieuwe package toevoegen, of is daar review voor nodig?

## 4. Overige

- TODO: invoervalidatie en omgaan met gebruikersinvoer.
- TODO: wat loggen we wél en wat nooit (geen persoonsgegevens, geen tokens).
- TODO: hoe lang bewaren we logs, en waar?
- TODO: security headers in de API en de frontend.

## 5. Open punten

- [ ] Doen we periodiek een security review of pentest?
- [ ] Wie is aanspreekpunt bij een securitymelding?
