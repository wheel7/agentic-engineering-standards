# Architecture: React

> **DRAFT - still to be reviewed by the team**
>
> No conventions have been agreed on this yet. This is a first, deliberately brief
> setup as a starting point for that discussion. Do not follow it as settled policy and
> do not extend it without review.

---

## 1. Principles

- Organize **by feature**, not by technical type. So `features/todos/`, and not
  separate `components/`, `hooks/` and `types/` folders with everything mixed together.
  (This deliberately mirrors the .NET side, see [`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md).)
- Keep components small and focused.
- Separate **presentation** (what it looks like) from **data access** (where it comes from).
- Only add abstractions once you need them.

## 2. Folder structure

```
src/
├── features/
│   └── todos/
│       ├── components/      Components of this feature
│       ├── api.ts           Data access (fetch / client)
│       ├── types.ts         Types of this feature
│       └── index.ts         What the feature exposes to the outside
├── components/              Shared, feature-independent UI
├── lib/                     Generic helpers
└── app/ or routes/          Routing and page structure
```

Rule: a feature imports from `components/` and `lib/`, but not directly from another
feature. If it has to, the shared part belongs one level up.

## 3. Conventions (proposal)

| Part | Naming | Example |
|---|---|---|
| Component | PascalCase | `TodoList.tsx` |
| Hook | `use` + camelCase | `useTodos.ts` |
| Type | PascalCase | `Todo`, `TodoDto` |

- TypeScript, no `any`.
- Function components with hooks.
- Explicit props types, no implicit `any`.

## 4. Open questions

This is still for the team to decide:

- [ ] **Data access**: TanStack Query, RTK Query, or plain `fetch` in a hook?
- [ ] **State**: what goes into server state, what into client state, and with which tool?
- [ ] **Styling**: Tailwind, CSS Modules, or something else?
- [ ] **Forms and validation**: React Hook Form, Zod?
- [ ] **Routing**: React Router, or the framework we pick (Next.js, Vite + router)?
- [ ] **Component library**: do we build our own, or do we take one (shadcn/ui, MUI)?
- [ ] **Folder structure**: does the feature layout above fit what we do in practice?
- [ ] How does this connect to [`../general/api-contracts.md`](../general/api-contracts.md)?

While these questions are open: look at what the existing project already does and do
not deviate from it without a reason.
