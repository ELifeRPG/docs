---
icon: lucide/network
---

# Architecture

ELifeRPG is split into a small set of independently deployed services that
share one identity provider (Keycloak) and one datastore (PostgreSQL, owned
by the Core Backend). The core request path is:

```
ArmA Reforger Gameserver → Mod → Bridge API → Core Backend
```

The gameserver mod never talks to the Core Backend directly. It only ever
talks to a local **Bridge API** process running alongside the gameserver,
which translates game-shaped calls into calls against the **Core Backend**
(`eliferpg-core`, internally also referred to as the Central API). A
separate **web portal** talks to the Core Backend independently, for
player self-service and staff administration.

## Component diagram

``` mermaid
graph TB
  subgraph Gameserver Host
    GS[ArmA Reforger Gameserver]
    Mod[ELifeRPG Mod]
    Bridge[Bridge API<br/>eliferpg-reforger-bridge]
    GS --- Mod
    Mod -- "local HTTP, no auth" --> Bridge
  end

  Bridge -- "OAuth2 client credentials<br/>+ token exchange (impersonation)" --> KC
  Bridge -- "REST, Bearer JWT" --> Core

  subgraph Core Services
    Core[Core Backend<br/>eliferpg-core, .NET modulith]
    PG[(PostgreSQL<br/>event store)]
    Core --> PG
  end

  KC[Keycloak]
  Core -- "validate JWT,<br/>provision users" --> KC

  Web[Web Portal<br/>eliferpg-webapp, Nuxt]
  Web -- "OIDC Authorization Code + PKCE" --> KC
  Web -- "REST, Bearer JWT" --> Core

  Theme[Keycloak Theme<br/>keycloak-theme-eliferpg]
  Theme -. "build artifact (JAR),<br/>deployed into" .-> KC
```

## Components

| Component | Repo | Stack | Exposes | Talks to |
|---|---|---|---|---|
| Gameserver / Mod | — | ArmA Reforger + ELifeRPG mod | — | Bridge (local HTTP) |
| Bridge API | `eliferpg-reforger-bridge` | .NET, ASP.NET Core minimal API | Game-sliced REST endpoints (session lifecycle, banking, characters, companies) | Keycloak, Core Backend |
| Core Backend | `eliferpg-core` | .NET modulith (ASP.NET Core) | REST + OpenAPI | Keycloak, PostgreSQL |
| Web Portal | `eliferpg-webapp` | Nuxt 4 (Vue 3), Nitro BFF routes | Browser-facing pages, session-cookie backed | Keycloak, Core Backend |
| Keycloak Theme | `keycloak-theme-eliferpg` | Keycloakify (React + Vite) | Build-only: theme JAR | Deployed into Keycloak, no runtime calls |

The Bridge API's only reason to exist as a separate service is that the
gameserver process needs a **local, low-latency, unauthenticated** endpoint
it can call synchronously from game logic. It owns no persistent state — a
restart drops in-memory session tracking — and holds no direct database
connection; everything durable lives behind the Core Backend.

The Bridge is an anti-corruption layer: it exists to keep Reforger's API
shapes, terminology, and connection/session model out of the Core Backend.
Reforger-specific concepts (e.g. a Reforger session or character id) stay
in the Bridge and get translated into domain commands at the boundary —
the Core Backend's domain types never depend on Reforger concepts, only
on its own.

"Local, unauthenticated" is only safe if the endpoint is actually
unreachable off-host: the Bridge must bind to `127.0.0.1` (not
`0.0.0.0`), with the gameserver mod as the only caller. This is a
deployment requirement, not just an implementation detail.

## Auth model

There are two distinct OAuth2 flows against the same Keycloak realm, one
per class of caller:

- **Service-to-service (Bridge → Core Backend):** OAuth2 **Client
  Credentials**. The Bridge authenticates to Keycloak as its own
  confidential client to get a service token, then calls the Core Backend
  with a Bearer JWT.
- **Human (Web Portal → Core Backend):** **OIDC** Authorization Code +
  PKCE — OAuth2 plus an identity layer, since the portal needs to know
  *who* is signed in, not just that a call is authorized. The browser
  never holds a token; the Nuxt server exchanges the code at Keycloak's
  OpenID Connect token endpoint and keeps the resulting ID and access
  tokens in an httpOnly session cookie. Role/staff checks (`isStaff`,
  `isAdmin`) are derived from claims on those tokens (a client scope and
  a realm role), and the access token is replayed as
  `Authorization: Bearer <token>` when the Nuxt server proxies calls to
  the Core Backend. Because the browser is authenticated purely via
  cookie, state-changing BFF routes need CSRF protection (`SameSite`
  cookie policy plus a CSRF token/double-submit check) — session cookies
  alone don't imply the request came from the portal's own UI.

### Player impersonation (token exchange)

Most Bridge calls act *on behalf of a specific player*, not as the Bridge
itself. Rather than have the Core Backend trust an arbitrary player id
supplied by the Bridge, the Bridge exchanges its own client-credentials
token for a player-scoped token, using RFC 8693 token exchange against
Keycloak:

``` mermaid
sequenceDiagram
  participant Bridge
  participant Keycloak
  participant Core as Core Backend

  Bridge->>Keycloak: client_credentials grant
  Keycloak-->>Bridge: service access token

  Bridge->>Keycloak: token exchange<br/>requested_subject = player username
  Note over Bridge,Keycloak: requires "impersonation" role<br/>on the Bridge's client
  Keycloak-->>Bridge: player-scoped access token

  Bridge->>Core: REST call, Bearer player token
  Core-->>Bridge: response
```

The Core Backend provisions a real Keycloak user for every player on first
contact so this exchange always has a subject to target. The Core Backend
itself never performs the exchange — Keycloak requires it to be
authenticated by the same client the subject token was originally issued
to, so the Bridge does it directly.

Because the Bridge's Keycloak client can impersonate any player, it's a
high-value credential: a compromised Bridge instance can act as any
player who has connected to it. The client should carry the minimal
Keycloak roles needed (impersonation only, nothing broader), issue
short-lived tokens, and token-exchange activity should be audited on the
Core Backend side, ideally tagged with the originating Bridge instance.

## Event-driven design

The Core Backend is built around **event sourcing on PostgreSQL** (via
Marten) — one event store per bounded module (accounts, characters,
banking, companies). Events aren't an external feed for other services
to consume; they're the source of truth the Core Backend's own domain
logic runs on. Each aggregate is reconstructed from its append-only
event stream, and projections build the read models from those same
events, using optimistic concurrency per stream.

``` mermaid
graph TB
  Req[HTTP / game request] --> Cmd[Command]
  Cmd --> App[Application handler]
  App --> Agg[Aggregate]
  Agg -- reject --> Err[Error response]
  Agg -- emit --> Ev[Events]
  Ev --> Store[(Event Store)]
  Store --> Proj[Projections]
  Proj --> Read[Read API]
```

### Domain events vs. technical events

Not everything that happens is worth event-sourcing. `MoneyDeposited`,
`CharacterCreated`, `CompanyFounded`, and `VehiclePurchased` are domain
events: they change aggregate state and belong in the append-only
stream, because the economy and a character's history need to be
answerable from the log itself (e.g. "why does this player have
€47,320"). `PlayerConnected`, `PlayerDisconnected`, and
`HeartbeatReceived` are technical/integration events — useful
operationally (session tracking, monitoring) but not domain history, so
they stay outside the event-sourced aggregates as transient messages or
logs rather than get appended to a stream.

## Module boundaries

Inside the Core modulith, each bounded module (`Accounts`, `Characters`,
`Banking`, `Companies`) owns its own `Domain`/`Application`/
`Infrastructure`/`Api` project set. This isn't just a naming convention —
project references enforce it:

- A module's `Domain`, `Infrastructure`, and `Api` projects can only
  reference their own module. Reaching into another module at those
  layers doesn't compile.
- The only permitted cross-module reference is `Application` →
  `Application`, and only to call that module's small set of public
  query contracts (e.g. `CompanyMemberPermissionsQuery`) — never another
  module's domain types directly.

This produces a strict dependency order: `Accounts` is the root
(depended on by `Characters`), and `Banking`/`Companies` depend on
`Characters` (`Companies` also depends on `Banking`). A module never
depends on one further down the chain.

Within a module's own Application layer, handler visibility is enforced
by convention and review, not by the compiler — the project-reference
rule above is what's structurally guaranteed.

## Integration boundary

The Bridge and the Web Portal call an integration API surface, not the
domain directly. REST DTOs at that boundary map into application-layer
commands, and domain events are a separate set of types — the two never
collapse into one. That indirection is what lets the domain model
evolve (renaming an aggregate field, splitting a module, reshaping an
event) without forcing a breaking change on Bridge or Web contracts, and
keeps game/HTTP-shaped payloads from becoming the vocabulary the domain
reasons in.

## Cross-aggregate transactions

Some operations span more than one module — buying a company share with
funds from a bank account touches `Banking`, `Companies`, and
`Characters`. The intended pattern is a single-transaction
**application-layer coordinator**, not a saga/process manager:

- The coordinating handler lives in the *most downstream* module
  involved, per the dependency order in [Module boundaries](#module-boundaries)
  (e.g. a "buy company share" handler lives in `Companies`, which
  already depends on `Banking` and `Characters`).
- It validates against each involved module's public Application query
  contracts — the same cross-module contract rule used everywhere else.
- It appends events to every affected aggregate stream (e.g. the bank
  account's and the company's) inside a single Marten
  session/transaction. This is atomic because every module's event
  store lives in the same PostgreSQL database — there's no distributed
  transaction to solve.

A saga/process manager is deliberately not used here: sagas exist to
coordinate steps across a real async or durable boundary (a remote
system, a step that may not complete for hours), which doesn't apply
when everything involved is one process and one database. That pattern
is reserved for the day a cross-aggregate operation genuinely needs to
span such a boundary (e.g. something waiting on the Bridge across
multiple requests) — introducing saga machinery before then would be
unwarranted complexity for a same-transaction problem.

## Open questions

Not yet decided, and deliberately not documented as fact above:

- **Consistency guarantees per operation.** Which reads are strongly
  consistent (e.g. character creation, a deposit) versus eventually
  consistent (e.g. a projection-backed leaderboard), stated explicitly
  per endpoint/operation.
- **Failure semantics.** What happens when the Bridge crashes before vs.
  after the Core Backend acknowledges an event; whether a projection
  failure blocks reads or is allowed to lag behind its event stream;
  how the Bridge and Core Backend each behave when PostgreSQL or
  Keycloak is unavailable (reject, buffer, degrade).
- **Event ingestion identifiers.** Whether event identity (`eventId`),
  request/command identity (`requestId`), and aggregate identity
  (`aggregateId` + `expectedVersion`) are tracked as distinct concepts
  in the ingestion path, rather than one id doing duty for idempotency,
  retries, and ordering all at once. The previous description of a
  batched `POST /api/events/batch` endpoint was removed from this doc
  because it no longer matches how ingestion works — a corrected
  description belongs here once ingestion is finalized.
