# Testing: React

> **DRAFT - still to be reviewed by the team**
>
> No conventions have been agreed on this yet. A brief setup as a starting point.

Belongs with [`ARCHITECTURE.md`](ARCHITECTURE.md).

---

## 1. Principles

- Test what the user sees and does, not how the component works internally.
  No tests on state variables or implementation details; those break on every refactor
  without anything being broken.
- Select on accessible properties (role, label, text), not on CSS classes.
- Mock network traffic at the level of the HTTP layer, not by replacing your own
  modules.

## 2. Proposed setup

| Kind | With what | What |
|---|---|---|
| Unit / component | Vitest + React Testing Library | Does it render, does it respond to interaction |
| API mocking | MSW (Mock Service Worker) | Intercept requests without mocking your own code |
| End-to-end | Playwright | The few critical user flows |

The emphasis is on component tests. E2E only for flows that really must not break; those
are slow and brittle.

## 3. Example (indicative)

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TodoList } from './TodoList';

test('shows todos and marks one as completed', async () => {
  render(<TodoList todos={[{ id: '1', title: 'Buy groceries', isCompleted: false }]} />);

  expect(screen.getByText('Buy groceries')).toBeInTheDocument();

  await userEvent.click(screen.getByRole('checkbox', { name: /buy groceries/i }));

  expect(screen.getByRole('checkbox', { name: /buy groceries/i })).toBeChecked();
});
```

## 4. Open questions

- [ ] Vitest or Jest? (Vitest is the obvious choice with Vite, but that depends on the
      framework choice in `ARCHITECTURE.md`.)
- [ ] Do we bring in MSW, or do we mock the data layer?
- [ ] Do we do E2E, and if so: which flows and where do they run?
- [ ] Do we hold to a coverage standard, or do we steer on review?
- [ ] Do we need a React equivalent of the .NET architecture tests
      (for example ESLint rules on import boundaries between features)?
