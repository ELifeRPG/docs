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
  the Core Backend.

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

## Event-driven design

The Core Backend is built around **event sourcing on PostgreSQL** — one
event store per bounded module (accounts, characters, banking,
companies). Events aren't an external feed for other services to
consume; they're the source of truth the Core Backend's own domain logic
runs on. Each aggregate is reconstructed from its append-only event
stream, and projections build the read models from those same events,
using optimistic concurrency per stream.

The Bridge (and other producers) push events to the Core Backend via a
batched, idempotent ingestion endpoint (`POST /api/events/batch`,
deduplicated by client-generated GUID).
