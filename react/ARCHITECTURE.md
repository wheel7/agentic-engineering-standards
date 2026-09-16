# Architectuur: React

> **CONCEPT - nog te reviewen door het team**
>
> Hier zijn nog geen afspraken over gemaakt. Dit is een eerste, bewust beknopte opzet
> als startpunt voor die discussie. Volg het niet als vaststaand beleid en breid het
> niet uit zonder review.

---

## 1. Uitgangspunten

- Organiseer **per feature**, niet per technisch type. Dus `features/todos/`, en niet
  losse mappen `components/`, `hooks/` en `types/` met alles door elkaar.
  (Dit spiegelt bewust de .NET-kant, zie [`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md).)
- Houd componenten klein en gefocust.
- Scheid **presentatie** (hoe het eruitziet) van **datatoegang** (waar het vandaan komt).
- Voeg abstracties pas toe als je ze nodig hebt.

## 2. Mappenstructuur

```
src/
├── features/
│   └── todos/
│       ├── components/      Componenten van deze feature
│       ├── api.ts           Datatoegang (fetch / client)
│       ├── types.ts         Types van deze feature
│       └── index.ts         Wat de feature naar buiten toe aanbiedt
├── components/              Gedeelde, feature-onafhankelijke UI
├── lib/                     Generieke helpers
└── app/ of routes/          Routing en pagina-opbouw
```

Regel: een feature importeert uit `components/` en `lib/`, maar niet rechtstreeks uit
een andere feature. Moet dat toch, dan hoort het gedeelde deel omhoog.

## 3. Conventies (voorstel)

| Onderdeel | Naamgeving | Voorbeeld |
|---|---|---|
| Component | PascalCase | `TodoList.tsx` |
| Hook | `use` + camelCase | `useTodos.ts` |
| Type | PascalCase | `Todo`, `TodoDto` |

- TypeScript, geen `any`.
- Functiecomponenten met hooks.
- Props-types expliciet, geen impliciete `any`.

## 4. Open punten

Dit moet het team nog beslissen:

- [ ] **Datatoegang**: TanStack Query, RTK Query, of gewoon `fetch` in een hook?
- [ ] **State**: wat gaat er in server-state, wat in client-state, en met welk hulpmiddel?
- [ ] **Styling**: Tailwind, CSS Modules, of iets anders?
- [ ] **Formulieren en validatie**: React Hook Form, Zod?
- [ ] **Routing**: React Router, of het framework dat we kiezen (Next.js, Vite + router)?
- [ ] **Componentbibliotheek**: bouwen we zelf, of nemen we er een (shadcn/ui, MUI)?
- [ ] **Mapstructuur**: past de feature-indeling hierboven bij wat we in praktijk doen?
- [ ] Hoe sluit dit aan op [`../general/api-contracts.md`](../general/api-contracts.md)?

Zolang deze punten openstaan geldt: kijk naar wat het bestaande project al doet en
wijk daar niet zonder reden van af.
