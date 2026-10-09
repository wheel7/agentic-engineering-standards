# Local ports

Every project runs on fixed ports on `localhost`, see [`containers.md`](containers.md)
chapter 3 and the project setup. Fixed, because the identity provider only redirects to a
registered URL, port included. But fixed per project is not enough: two projects on one
machine that picked the same port do not both start, and the one that starts second fails
with an error that says nothing about the other project.

So the ports are handed out here, one block per project. The project setup reads this
file, proposes the next free block, and adds the new project in the same pull request as
the rest of the standards changes, or the project records its block in its own `CLAUDE.md`
and this file gets it in the next standards pull request.

## The blocks

A block is ten ports, `x0` to `x9`, used the same way in every project:

| Port | What |
|---|---|
| `x3` | Aspire dashboard (one below the frontend) |
| `x4` | Frontend, Vite dev server: the callback URL at the identity provider |
| `x5` | API, under Aspire, `dotnet run` and as the host port in compose |
| `x6`–`x9` | Anything else that needs a fixed port: a mock of an outside system, a second service |

The `npm run preview` server of the end-to-end journeys gets `4x4`, so it moves with the
block as well.

## Handed out

| Project | Block | Dashboard | Frontend | API | Other |
|---|---|---|---|---|---|
| Havenmeester | 5040 | 5043 | 5044 | 5045 | 5050 Afas mock (compose); preview 4044 |
| Bobinera | 5047–5049, its own layout | 5047 | 5048 | 5049 | preview 4173 (before this register) |
| *next* | 5060 | 5063 | 5064 | 5065 | |

Bobinera and the Afas mock of Havenmeester predate the blocks and keep their ports: moving
them means changing the redirect URLs at the identity provider. A new project takes the next
free block of ten after the highest port in use, and never one inside or between these.

## Avoid

The defaults of the tools, which every other project on the machine wants too:

| Port | Taken by |
|---|---|
| 5000, 5001 | `dotnet run` without launch settings, AirPlay on macOS |
| 5173, 4173 | Vite dev and preview |
| 5432 | Postgres |
| 3000 | many Node tools |
| 8080, 8443 | container examples, and the port inside the API image |
| 5050 | a locally installed pgAdmin, and Havenmeester's Afas mock |
