# Git-workflow

> **Nog in te vullen door het team.** Onderstaande koppen geven de structuur; de
> TODO's zijn de beslissingen die we nog moeten nemen.

Geldt voor alle projecten, ongeacht stack.

---

## 1. Branching

- TODO: welk model? Trunk-based met korte feature branches, of GitFlow met
  `develop`/`release`-branches?
- TODO: naamgeving van branches vastleggen, bijvoorbeeld
  `feature/<ticket>-korte-omschrijving`, `bugfix/...`, `hotfix/...`.
- TODO: hoe lang mag een branch openstaan voordat we hem opsplitsen?
- TODO: wie mag rechtstreeks naar `main` pushen, en onder welke voorwaarden?

## 2. Commit-berichten

- TODO: gebruiken we Conventional Commits (`feat:`, `fix:`, `chore:`)? Zo ja, welke
  types staan we toe?
- TODO: Nederlands of Engels in commit-berichten? (Documentatie is Nederlands; voor
  commits is dat nog niet besloten.)
- TODO: verwijzen we naar het ticketnummer, en waar - in de titel of de body?
- TODO: maximale lengte van de titelregel.

## 3. Pull requests

- TODO: aantal verplichte reviewers.
- TODO: merge-strategie: squash, merge commit, of rebase?
- TODO: welke checks moeten groen zijn voordat je mag mergen (build, tests, linter)?
- TODO: maximale omvang van een PR - wanneer vragen we om opsplitsen?
- TODO: gebruiken we een PR-template? Zo ja, wat staat erin?
- TODO: branch protection rules op `main` instellen.

## 4. Submodule `.standards`

Dit ligt al wel vast:

- Bijwerken van `.standards` gebeurt via een eigen PR, zodat de wijziging in de
  standaarden zichtbaar is in de diff. Zie de [README](../README.md).
- Meng een submodule-update niet met functionele wijzigingen in dezelfde PR.

## 5. Open punten

- [ ] Hoe gaan we om met langlopende releases en hotfixes op productie?
- [ ] Taggen en versienummering.
