# Security

> **Still to be decided by the team.** The headings below give the structure; the
> TODOs are the decisions we still have to make.

Applies to every project, whatever the stack.

---

## 1. Secrets

- TODO: where do secrets belong per environment (local, test, production)? Think of
  .NET user-secrets locally and a key vault everywhere else.
- TODO: record that secrets never end up in the repo - not in `appsettings.json`,
  `.env` files or test data either.
- TODO: turn on secret scanning for the repos.
- TODO: procedure for when a secret has leaked anyway (revoke, rotate, report).
- TODO: how often do we rotate keys and certificates?

## 2. Authentication and authorization

- TODO: which identity provider do we use by default?
- TODO: token conventions: type, lifetime, where do you keep them in the frontend?
- TODO: authorization model: roles, claims, or policies?
- TODO: how do we secure service-to-service traffic?
- TODO: conventions for CORS.

## 3. Dependencies

- TODO: do we use Dependabot or Renovate for updates?
- TODO: how fast do vulnerabilities have to be fixed, per severity?
- TODO: do we run `dotnet list package --vulnerable` and `npm audit` in CI?
- TODO: may anyone add a new package, or does that need a review?

## 4. Other

- TODO: input validation and handling user input.
- TODO: what do we log and what never (no personal data, no tokens).
- TODO: how long do we keep logs, and where?
- TODO: security headers in the API and the frontend.

## 5. Open questions

- [ ] Do we run a periodic security review or pentest?
- [ ] Who is the point of contact for a security report?
