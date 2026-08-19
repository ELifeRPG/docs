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

## Components

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
```

| Component | Repo | Stack | Exposes | Talks to |
|---|---|---|---|---|
| Gameserver / Mod | — | ArmA Reforger + ELifeRPG mod | — | Bridge (local HTTP) |
| Bridge API | `eliferpg-reforger-bridge` | .NET, ASP.NET Core minimal API | Game-sliced REST endpoints (session lifecycle, banking, characters, companies) | Keycloak, Core Backend |
| Core Backend | `eliferpg-core` | .NET modulith (ASP.NET Core) | REST + OpenAPI | Keycloak, PostgreSQL |
| Web Portal | `eliferpg-webapp` | Nuxt 4 (Vue 3), Nitro BFF routes | Browser-facing pages, session-cookie backed | Keycloak, Core Backend |

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

## Authentication

Three distinct trust relationships run against the same Keycloak realm,
one per class of caller.

### Gameserver

The mod's calls into the Bridge are **local HTTP, no auth**. That's only
safe if the endpoint is actually unreachable off-host: the Bridge must
bind to `127.0.0.1` (not `0.0.0.0`), with the gameserver mod as the only
caller. This is a deployment requirement, not just an implementation
detail — "local, unauthenticated" becomes an unexpectedly large attack
surface the moment that binding is wrong.

### Bridge

**Service-to-service (Bridge → Core Backend):** OAuth2 **Client
Credentials**. The Bridge authenticates to Keycloak as its own
confidential client to get a service token, then calls the Core Backend
with a Bearer JWT.

#### Player impersonation (token exchange)

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

### Portal

**Human (Web Portal → Core Backend):** **OIDC** Authorization Code +
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

## Core Backend design

`eliferpg-core`'s internal conventions — event sourcing, module
boundaries, cross-module transactions, and consistency/failure
guarantees — are documented separately in
[Core Backend Design](core-backend-design.md), since they're internal
conventions rather than system topology.
