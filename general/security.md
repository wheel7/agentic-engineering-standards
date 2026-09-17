# Security

Applies to every project, whatever the stack.

---

## 1. Secrets

**A secret never enters the repository.** Not in `appsettings.json`, not in a `.env`
file, not in test data, not in a comment, not "temporarily".

One documented exception: the local-only database password in `docker-compose.yml`, which
guards a throwaway container on your own machine and is written down as such in
[`../ops/containers.md`](../ops/containers.md). If you find yourself arguing that a second
secret is also harmless, it is not.

Where they live:

| Where | What holds them |
|---|---|
| Your machine | .NET user-secrets, never a file inside the repository |
| CI | GitHub Actions secrets, scoped to the workflow |
| Deployed environments | a GitHub Environment per environment, see [`../ops/environments.md`](../ops/environments.md) |

**Turn on secret scanning and push protection** on every repository. Both are free and
push protection is the one that matters, because it refuses the push instead of telling
you afterwards.

### When one leaks anyway

In this order, and the order is the point:

1. **Revoke it.** A rotated secret whose predecessor is still valid has not been dealt
   with.
2. **Issue a replacement** and deploy it.
3. **Work out what it could reach** and whether it was used.
4. **Report it** if personal data was exposed. That is a legal obligation with a clock on
   it, not a judgment call.

Rewriting git history afterwards is housekeeping, not remediation. Assume anything pushed
to a public repository was captured within minutes.

### Rotation

| What | How often |
|---|---|
| Client secrets at the identity provider | yearly |
| Database passwords | when someone with access leaves |
| Anything that leaked | immediately, see above |

---

## 2. Authentication and authorization

Authentication runs through an external provider, Kinde or Entra ID, chosen per project,
with a separate application registration per environment. The identity model behind it,
and why one shared registration is dangerous, is in
[`../ops/environments.md`](../ops/environments.md) and
[`../ops/database.md`](../ops/database.md).

**Tokens.** A short-lived JWT access token, minutes rather than hours, refreshed by the
provider's SDK. The API validates issuer, audience and signature, and rejects a token
minted for another environment.

**In the frontend, not in `localStorage`.** Anything in local storage is readable by any
script that gets injected into the page, and that is the whole payoff of an XSS bug. Keep
the access token in memory and let the refresh token live in an httpOnly cookie, so a
script can use neither.

**Authorization is policies, not role checks scattered around.** Register named policies
once, and let endpoints ask for a policy by name. `if (user.IsInRole("Admin"))` spread
through handlers is how a permission ends up meaning two different things in two places.

Claims come from the provider; what they entitle you to is our decision and lives in our
code.

**Service to service** uses the client credentials flow at the same provider, with its own
registration per service. Not a shared API key, because a shared key cannot be revoked for
one caller and cannot tell you which caller used it.

**CORS** is an explicit allowlist of origins, read from configuration so it differs per
environment. Never `AllowAnyOrigin` together with credentials; the browser will refuse it
anyway, and the version that does work is the one you did not want.

---

## 3. Dependencies

**Dependabot**, because it is part of GitHub and needs no account anywhere else. Grouped
weekly updates, so you review one pull request instead of eleven.

**Fix vulnerabilities on a clock:**

| Severity | Fixed within |
|---|---|
| Critical | 2 days |
| High | 1 week |
| Medium | the current sprint |
| Low | when you are in there anyway |

CI already fails on a vulnerable package, including transitive ones. See
[`../ops/ci-cd.md`](../ops/ci-cd.md). That check is the floor, not the policy: it tells
you about known advisories on the day you build.

**A new dependency needs a review.** Not because anyone is suspicious, but because a
dependency is permanent and its cost is paid later. The pull request is the moment to ask
whether the forty lines you need are worth a package, a transitive tree and an upgrade
every quarter.

---

## 4. Logging and data

**Never in a log:** passwords, tokens, secrets, full request bodies, and personal data.
That means no names, no email addresses, no addresses, no payment details.

Log the user id and the `traceId` instead. Both are meaningless to anyone who steals the
log file and sufficient for anyone debugging with the database next to them.

Logs go to stdout as structured JSON, see [`../ops/containers.md`](../ops/containers.md).

| Environment | Kept for |
|---|---|
| Test and acceptance | 30 days |
| Production | 90 days |

Longer only when something legal requires it, and then write down which obligation, because
"we might need it" is how a log store becomes the largest collection of personal data in
the company.

---

## 5. Input and headers

**Validate at the edge**, on the request type, before a handler sees it. A handler is
entitled to assume its input is well formed.

Validation is not authorization. Checking that an order id is a well-formed UUID says
nothing about whether this user may see that order, and the two get conflated often enough
to be worth a sentence.

**Security headers**, mostly on whatever serves the frontend:

| Header | Value |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | as tight as the application allows, and `frame-ancestors 'none'` |

An API that serves only JSON needs HSTS and `nosniff`; the rest are about documents in a
browser. Non-production environments additionally get `X-Robots-Tag: noindex`, see
[`../ops/environments.md`](../ops/environments.md).

---

## 6. Open questions

- [ ] Do we run a periodic security review or pentest, and who pays for it?
- [ ] Who is the point of contact for a security report? A public repository should carry
      a `SECURITY.md` saying where to send one and what to expect back.
