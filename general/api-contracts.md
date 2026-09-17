# API contracts

Shared conventions between frontend and backend. Deliberately stack-independent: both
sides have to stick to it, otherwise it is not a contract.

If it is about how you build an endpoint, that belongs in
[`../dotnet/ARCHITECTURE.md`](../dotnet/ARCHITECTURE.md). This file only covers what
goes over the wire.

---

## 1. Paths

- **Plural for collections.** `/todos`, `/todos/{id}`. An item is an element of its
  collection, so the name does not change when you address one of them.
- **kebab-case, lowercase.** `/order-lines`, not `/orderLines`. Paths get typed by hand
  and pasted into tickets, and casing is where that goes wrong.
- **Nest at most one level**, and only when the child has no identity outside its parent.
  `/orders/{id}/lines` is fine, because a line without an order means nothing.
  `/customers/{id}/orders` is not: an order has its own id, so it is
  `/orders?customerId=`.
- **Actions that are not CRUD get a sub-resource with a verb.**
  `POST /todos/{id}/complete`. Do not bend `PATCH` into meaning "complete it", because
  then the next reader has to guess which fields that touches.

### Versioning

No version in the path until the first breaking change. A `/v1` that never gets a `/v2`
is a prefix you type for years for nothing, and with one frontend that you deploy
yourself, a breaking change is a coordination problem rather than a compatibility one.

When it does happen, the version goes in the path: `/v2/todos`. Both run side by side
until the last client has moved, and then the old one goes. Same shape as the
expand-contract rule in [`../ops/database.md`](../ops/database.md), for the same reason:
during a deploy, old and new exist at the same time.

Most changes need no version at all. Adding a field is not breaking. Removing one,
renaming one, or changing its type is.

---

## 2. Errors

**RFC 7807**, served as `application/problem+json`. It is what `Results.Problem()` and
`Results.ValidationProblem()` already produce in .NET, so following the standard costs
nothing and the alternative is inventing a format per project.

```json
{
  "type": "https://example.com/errors/insufficient-credit",
  "title": "Insufficient credit",
  "status": 409,
  "detail": "The order total exceeds the remaining credit on this account.",
  "instance": "/orders/0192f4c1",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

- **`type` is the machine-readable code.** The frontend switches on this and never on
  `title` or `detail`, because those are prose and prose gets reworded.
- **`traceId` is always there.** It comes from `Activity.Current`, and it is what turns a
  support ticket into a log line instead of an afternoon.
- **`detail` is for a human** and never carries a stack trace, a SQL fragment or an
  internal identifier.

### Validation errors

The `errors` dictionary from `ValidationProblem`, keyed on the field as it appears in the
request body, so camelCase:

```json
{
  "type": "https://example.com/errors/validation",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "title": ["Title is required."],
    "dueDate": ["Due date cannot be in the past."]
  }
}
```

### Status codes

| Code | When |
|---|---|
| 200 | a query succeeded and returns something |
| 201 | something was created, with a `Location` header pointing at it |
| 204 | a command succeeded and there is nothing to return |
| 400 | the request is malformed, or it fails validation |
| 401 | not authenticated, or the token has expired |
| 403 | authenticated, but not allowed to do this |
| 404 | it does not exist, or you are not allowed to know that it does |
| 409 | it conflicts with the current state, including a concurrency token mismatch |

Two of those need saying out loud.

**400 and not 422 for validation.** ASP.NET Core produces 400 out of the box, the
distinction gets argued about more than it earns, and a frontend that handles both ends
up with two code paths doing the same thing.

**404 rather than 403 where existence is itself information.** Telling someone that order
12345 exists but is not theirs tells them their competitor has an order 12345. Return 404
and let them conclude nothing.

---

## 3. Dates and times

- **Everything UTC, ISO 8601, with the `Z`**: `2026-09-17T14:30:00Z`. Never a local time,
  never an offset the receiver has to interpret.
- **A date without a time travels without a time**: `2026-09-17`. No midnight, no zone.
  This matches the `date` and `DateOnly` rule in
  [`../ops/database.md`](../ops/database.md). A day that travels as a timestamp arrives
  on the wrong day somewhere.
- **Conversion to local time happens in the frontend only.** The API does not know where
  the user is sitting and should not guess.

---

## 4. Pagination

Page and size, with a total, because the screens we build have page numbers and a counter.

```
GET /todos?page=1&size=20&sort=-createdDate
```

```json
{
  "items": [],
  "page": 1,
  "size": 20,
  "total": 143
}
```

- **`page` counts from one.** Users do, and the off-by-one belongs in one place on the
  server rather than in every client.
- **`size` defaults to 20 and is capped at 100.** A request above the cap is clamped
  rather than rejected, and the response says which size was actually used. That is what
  `size` in the response is for.
- **`sort` takes a field name, with a leading minus for descending.** Only fields on an
  allowlist. Sorting on whatever a caller names invites a scan of a table that has no
  index for it.
- **Filtering is an explicit parameter per field**, like `?isCompleted=false`. No generic
  query language over the wire: that is a database with extra steps and no way to reason
  about what a request costs.

Cursor-based pagination is the better answer for very large or fast-moving datasets. Move
to it when deep pages get slow, not before.

---

## 5. JSON

- **camelCase for properties.** The frontend is JavaScript and .NET serializes this way by
  default.
- **A response always carries the field**, with `null` when there is no value. Leaving it
  out means a generated client sees an optional property that is not actually optional.
- **In a request, missing and `null` mean different things.** Missing is "do not change
  this", `null` is "clear it". That distinction is the only thing that makes a partial
  update expressible, so do not collapse it.
- **No wrapper around a successful response.** The resource is the body. Errors have their
  own media type, which is difference enough to tell them apart.

---

## 6. OpenAPI is the contract

The API publishes an OpenAPI document, generated from the code rather than kept by hand.
.NET produces it out of the box.

**Commit the generated document.** That one habit answers who guards the contract when
frontend and backend live in different repositories: a change to a response shape turns
up as a diff in a file a reviewer can read, instead of as a surprise in somebody's
browser console. A CI step regenerates it and fails when it differs from what was
committed.

The frontend generates its client from that document rather than hand-writing the types.
Types written down twice drift apart.

---

## 7. Open questions

- [ ] Do we want contract tests on top of the committed OpenAPI document, or is the diff
      enough? See [`testing.md`](testing.md).
- [ ] Will there be a .NET client (Blazor, console, another service)? Then the rules for
      `.Contracts` in [`../dotnet/solution-layout.md`](../dotnet/solution-layout.md)
      apply. A React frontend reads the contract from OpenAPI and does not need that
      project.
