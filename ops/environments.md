# Environments

Which environments exist, what is different about each one, and what has to be separate
between them. Known in Dutch practice as OTAP, for development, test, acceptance and
production.

Assumes the model in [`containers.md`](containers.md): containers behind a reverse proxy
on a host you control. The platform specifics of that host are not settled yet.

---

## 1. Start with two

Four environments is the end state, not the starting point. Each deployed environment is
a database to back up, a set of secrets to rotate, a certificate to renew and a place
where something can be subtly out of date.

- **Development and production** is where every project starts.
- **Add test** when you have journeys that need a deployed target, or when someone has to
  look at a change before it reaches customers.
- **Add acceptance** when someone outside the team signs off on a release. If nobody
  does, it is a second test environment with a nicer name.

| Environment | Used by | Lives | Database |
|---|---|---|---|
| Development | the developer | the laptop, via compose | thrown away at will |
| Test | the team, and CI | deployed | rebuilt on every deploy |
| Acceptance | the business | deployed | persists, refreshed from production |
| Production | customers | deployed | persists |

---

## 2. Naming

Nest the environment, do not prefix it:

```
example.com                 api.example.com          production
test.example.com            api.test.example.com     test
acc.example.com             api.acc.example.com      acceptance
```

`testapi.example.com` works fine until the fourth service arrives and you are inventing
`testadmin` and `testworker`. Nesting keeps one pattern no matter how many services there
are, and it groups every hostname of an environment under one label.

Production has no prefix. An environment marker on the production URL is a marker
somebody will eventually read as "not the real one".

### Non-production is not public

Test and acceptance are reachable by anyone who guesses the hostname, and they hold
data that looks real. At a minimum:

- An `X-Robots-Tag: noindex` header, so the environment does not turn up in search
  results.
- An IP allowlist or basic authentication in front of the proxy.

This is not paranoia. A test environment carrying a restore of production data, indexed
and reachable, is a data breach with a URL.

---

## 3. The database

Every environment gets its **own Postgres container**, not a shared instance with several
databases in it. A migration aimed at the wrong connection string can then only damage
its own environment, and you can run different major versions while upgrading one of
them. An idle Postgres container costs tens of megabytes, which is not an argument
against.

What differs is whether the data survives, and the reason is different per environment.

### Test is rebuilt

Its value is reproducibility. Every deploy drops the database, runs the migrations and
applies a seed dataset that is committed in the repository.

- Seed rows use fixed UUIDs, so a test can refer to them without looking them up.
- The seed includes the system user from [`database.md`](database.md).
- The seed is small enough to read. It is a fixture, not a copy of the business.

Let a test environment accumulate state and within a month the tests depend on rows
nobody can account for, and a red run cannot be reproduced. That is the whole reason the
environment exists, so protect it.

### Acceptance persists

Its value is realism. Someone from the business is halfway through checking something,
and the data has to have enough volume and enough mess to be worth looking at.

Refresh it periodically from a restore of production, and treat that refresh as a
deployment of its own: announced, scheduled, and never in the middle of a review.

### You cannot get both from one environment

Reproducibility and realism pull in opposite directions. A reproducible environment has
to be wiped, and a realistic one cannot be. That is the reason there are two letters in
the middle of OTAP, and it is the reason merging test and acceptance quietly costs you
one of the two.

### Anonymize before anyone can look

A restore of production carries real personal data, and the people who will click around
in acceptance have no business seeing it. Anonymization runs **as part of the restore**,
before the environment comes up, never as a step someone remembers to do afterwards.

At minimum, overwrite names, email addresses, phone numbers, postal addresses and payment
details. Do not forget free-text fields: notes, descriptions and comments are where
personal data hides from whoever wrote the anonymization script.

Developers do not connect to the production database. If you need to look at something,
look at it on a restore.

---

## 4. Authentication

**A separate application registration per environment.** Not one registration with three
sets of redirect URLs.

That shortcut is the dangerous one. With a single registration, a token minted for test
carries the same issuer and the same audience as a production token. A production API
that validates issuer and audience will accept it, and at that point anyone who can log
in to test has given themselves production access.

So, per environment:

| Separate | Why |
|---|---|
| Client id and secret | a leaked test secret buys nothing elsewhere |
| Redirect URIs | the provider only ever sends tokens to that environment |
| Audience | the API of one environment rejects tokens minted for another |
| Test accounts | a production account is never used to try something out |

Kinde has environments built in. In Entra ID it is a second app registration. Either way
the application reads which one it is from configuration, the same as everything else
that differs per environment.

There is no identity provider in the compose stack, so CI has a separate problem. The
options are in [`ci-cd.md`](ci-cd.md).

---

## 5. One host, several environments

Running test, acceptance and production as separate compose projects behind one reverse
proxy on one machine is a reasonable place to start. One directory per environment, each
with its own `.env` and its own named volumes, and a proxy that routes on hostname and
handles certificates.

What to know before you do it:

- **Set memory and CPU limits per container.** Without them a runaway process in test
  takes production down with it, and that is a strange way to find out they share a
  machine.
- **Watch the disk.** Logs and Postgres volumes from three environments fill a disk
  together, and a full disk stops production regardless of which environment filled it.
- **The proxy is a single point of failure** for everything on the host.

None of that is a reason not to do it. It is a reason to move production to its own host
once it is carrying something you would be called about at night.

---

## 6. What travels and what does not

**The image travels.** The same image that passed CI goes to test, then to acceptance,
then to production. Rebuilding per environment means the thing on production was never
the thing you tested. See [`containers.md`](containers.md).

**Configuration does not travel.** Connection strings, client ids, audiences, base URLs
and feature switches come from the environment, through environment variables.

**Secrets do not travel.** Use GitHub Environments so each one carries its own, with
required reviewers on production. A production secret exists in exactly one place and is
never reused anywhere.

The promotion flow that follows from this: a green CI run on `main` deploys to test on its
own, and acceptance and production are deliberate promotions of that same image. `main` is
the only branch that builds, and the `production` tag is moved to the commit that went to
production, so it always says what is live. See [`ci-cd.md`](ci-cd.md) and
[`../general/git-workflow.md`](../general/git-workflow.md).

---

## 7. Record it per project

The environments themselves are project-specific, so they belong in the project's own
`CLAUDE.md`, not here. Per environment: the URLs, whether the database is rebuilt or
persists, which app registration it uses, and who is allowed to deploy to it.

---

## 8. Checklist: a new environment

1. Own hostnames, nested under the environment label.
2. Own Postgres container and own volume.
3. Decided and written down: rebuilt on deploy, or persists.
4. If it persists and holds a production restore: anonymization runs inside the restore.
5. Own application registration at the identity provider, with its own audience.
6. Own secrets, in a GitHub Environment of its own.
7. Not production? Then `noindex` and something in front of it.
8. Memory and CPU limits on every container.
9. Written into the project `CLAUDE.md`.
