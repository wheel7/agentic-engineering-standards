---
name: react-component
description: DRAFT - adds a new React component following our (not yet settled) frontend architecture. Use this skill whenever a new component, screen or feature has to be added to the React frontend of a project, for example "create a TodoList component" or "add a screen to show orders". The conventions behind it have not been confirmed by the team yet; always check first how the existing project does it.
---

# Adding a new React component

> **DRAFT - still to be reviewed by the team.**

> The `.standards/` paths below assume this project has the standards as a git submodule.
> Without it, the same documents are in the repository this skill was installed from.

>
> The underlying conventions in `@.standards/react/ARCHITECTURE.md` and
> `@.standards/react/testing.md` are still a draft. On a conflict, always follow what
> the existing project already does, and report the deviation.

## Before you start

1. Read `.standards/react/ARCHITECTURE.md` and `.standards/react/testing.md`.
2. Read the project's `CLAUDE.md` - it says which choices this project has already
   made (state, styling, data access).
3. Look at a comparable existing component and follow that style. That weighs more
   heavily than this skill, as long as the standard is a draft.
4. Decide where the component belongs: inside a feature (`src/features/<feature>/`) or
   shared (`src/components/`). Both paths are relative to the frontend folder,
   `src/<Product>.Web/`, and that is also where you run `npm`. When in doubt: start inside
   the feature and only move it once a second feature needs it.

## Steps

### 1. Placement and naming

- Component in PascalCase: `TodoList.tsx`.
- Hook in camelCase with a `use` prefix: `useTodos.ts`.
- A feature does not import directly from another feature.

### 2. Props and types

- TypeScript, explicit props type, no `any`.
- Types of the feature in `types.ts`; only export what is needed outside.

### 3. Separating presentation and data

- Keep the component itself presentation as much as possible: props in, UI out.
- Data access (fetch, caching) in a hook or in the feature's `api.ts`, not spread
  through the component.
- Use the data access mechanism this project already uses.

### 4. Accessibility

- Use semantic elements (`button`, not a `div` with `onClick`).
- Make sure interactive elements have an accessible name - that is also what the
  tests select on.

### 5. Test

See `@.standards/react/testing.md`. Test what the user sees and does:

- Select on role, label or text - not on CSS classes or implementation details.
- Cover at least the most important interaction.

### 6. Exporting

Include the component in the feature's `index.ts` if it is used outside the feature.

## Wrapping up

- Run the project's linter, the typecheck (`tsc --noEmit`) and the tests.
- Report which files you added.
- Report the test evidence in the format from `@.standards/general/testing.md`: the
  command you ran, the summary line it actually printed, which tests you added and what
  they assert, what this change touches that no test covers, and anything you could not
  verify here. Those last two are not allowed to be empty, and a summary of a run that
  did not happen is worse than saying you could not run it.

- Did you run into a choice that is still open in `react/ARCHITECTURE.md` (state,
  styling, forms)? Then report what you chose and why, so the team can take that along
  in the review of the standard.
