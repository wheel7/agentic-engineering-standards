# API contracts

> **Still to be decided by the team.** The headings below give the structure; the
> TODOs are the decisions we still have to make.

Shared conventions between frontend and backend. This file is deliberately
stack-independent: both sides have to stick to it, otherwise it is not a contract.

If it is about how you build an endpoint, that belongs in
[`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md). This file only covers what
goes over the wire.

---

## 1. Endpoint naming

- TODO: plural or singular in paths? (`/todos/{id}` or `/todo/{id}`)
- TODO: casing in paths: kebab-case, camelCase?
- TODO: how deep do we nest resources? (`/orders/{id}/lines` or a flat structure)
- TODO: how do we model actions that are not CRUD? (`POST /todos/{id}/complete`)
- TODO: do we version, and if so: in the path, a header, or not at all?

## 2. Error format

- TODO: do we use RFC 7807 (`application/problem+json`)? That fits with
  `Results.Problem()` in .NET, so it is the obvious choice.
- TODO: agree on fixed fields (for example `type`, `title`, `status`, `detail`,
  `traceId`).
- TODO: how do we return validation errors per field?
- TODO: which status codes do we use for what? (400 vs 422, 401 vs 403, 404 vs 204)
- TODO: do we return an error code that the frontend can translate, or only text?

## 3. Dates and times

- TODO: record that everything is UTC and goes out in ISO 8601 (`2026-09-16T14:30:00Z`).
  The .NET side stores every timestamp in UTC already, so that fits.
- TODO: how do we pass a date without a time?
- TODO: where does the conversion to local time happen - only in the frontend?

## 4. Pagination

- TODO: which style? Page/size, offset/limit, or cursor-based?
- TODO: fixed naming for the parameters.
- TODO: what does the response look like - only items, or a total and a next page too?
- TODO: default and maximum value for the page size.
- TODO: conventions for sorting and filtering.

## 5. Other

- TODO: casing of JSON properties (camelCase is the obvious choice for a
  JavaScript frontend).
- TODO: do we leave `null` out or send it explicitly?
- TODO: do we publish an OpenAPI specification, and does the frontend generate its
  client from it?

## 6. Open questions

- [ ] Who guards the contract when frontend and backend live in different repos?
- [ ] How do we roll out a breaking change without breaking the frontend?
- [ ] Will there be a .NET client (Blazor, console, another service)? Then the rules for
      `.Contracts` from [`../dotnet/solution-layout.md`](../dotnet/solution-layout.md)
      apply. A React frontend reads the contract from OpenAPI and does not need that
      project.
