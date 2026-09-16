# Testen: React

> **CONCEPT - nog te reviewen door het team**
>
> Hier zijn nog geen afspraken over gemaakt. Beknopte opzet als startpunt.

Hoort bij [`ARCHITECTURE.md`](ARCHITECTURE.md).

---

## 1. Uitgangspunten

- Test wat de gebruiker ziet en doet, niet hoe het component intern werkt.
  Geen tests op state-variabelen of implementatiedetails; die breken bij elke refactor
  zonder dat er iets stuk is.
- Selecteer op toegankelijke eigenschappen (rol, label, tekst), niet op CSS-classes.
- Netwerkverkeer mock je op het niveau van de HTTP-laag, niet door je eigen modules
  te vervangen.

## 2. Voorgestelde opzet

| Soort | Waarmee | Wat |
|---|---|---|
| Unit / component | Vitest + React Testing Library | Rendert het, reageert het op interactie |
| API-mocking | MSW (Mock Service Worker) | Requests onderscheppen zonder je eigen code te mocken |
| End-to-end | Playwright | De paar kritieke gebruikersstromen |

Zwaartepunt op componenttests. E2E alleen voor stromen die echt niet stuk mogen; die
zijn traag en breken makkelijk.

## 3. Voorbeeld (indicatief)

```tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TodoList } from './TodoList';

test('toont todos en markeert er een als voltooid', async () => {
  render(<TodoList todos={[{ id: '1', title: 'Boodschappen', isCompleted: false }]} />);

  expect(screen.getByText('Boodschappen')).toBeInTheDocument();

  await userEvent.click(screen.getByRole('checkbox', { name: /boodschappen/i }));

  expect(screen.getByRole('checkbox', { name: /boodschappen/i })).toBeChecked();
});
```

## 4. Open punten

- [ ] Vitest of Jest? (Vitest ligt voor de hand bij Vite, maar dat hangt van de
      frameworkkeuze in `ARCHITECTURE.md` af.)
- [ ] Zetten we MSW in, of mocken we de datalaag?
- [ ] Doen we E2E, en zo ja: welke stromen en waar draaien ze?
- [ ] Hanteren we een dekkingsnorm, of sturen we op review?
- [ ] Hebben we een React-equivalent van de .NET-architectuurtests nodig
      (bijvoorbeeld ESLint-regels op import-grenzen tussen features)?
