---
icon: lucide/boxes
---

# Core Backend Design

This page covers how `eliferpg-core` is built internally: its
event-sourcing conventions, module boundaries, how it coordinates
writes that span modules, and what consistency and failure guarantees
follow from those choices. For system topology and auth flows, see
[Architecture](architecture.md).

## Event-driven design

The Core Backend is built around **event sourcing on PostgreSQL** (via
Marten) — one event store per bounded module (accounts, characters,
banking, companies). Events aren't an external feed for other services
to consume; they're the source of truth the Core Backend's own domain
logic runs on. Each bounded module reconstructs its aggregates from an
append-only event stream, and projections build the read models from
those same events, using optimistic concurrency per stream:

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
  `Application`, used for two things: calling another module's small
  set of public query contracts (read-only lookups, never another
  module's domain types directly), or — for the narrower case of an
  operation that must write to two modules atomically — obtaining a
  repository from a factory the target module explicitly exposes (see
  [Cross-aggregate transactions](#cross-aggregate-transactions)).

This produces a strict dependency order — every module sits somewhere
in a DAG rooted at whichever module owns the most fundamental concept,
with everything else depending on it directly or transitively. A
module never depends on one further down the chain. Which module
depends on which is a property of the current module set, not a fixed
fact — it changes as modules are added and requirements shift, so it's
deliberately not enumerated here; check the actual project references
for the current graph.

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

Some operations span more than one module — e.g. an operation that
debits or reserves a limited resource in one module (money, stock,
anything finite) and grants something in exchange in another. The
deciding factor for how to coordinate that write is not which modules
are involved, but whether both modules' data lives in the same
physical database. Today it does for all of them, so the pattern is a
single-transaction **application-layer coordinator**, not a
saga/process manager:

- The coordinating handler lives in whichever of the two modules is
  more downstream in the dependency graph (see
  [Module boundaries](#module-boundaries)) — e.g. for an operation
  touching Module A and Module B where A depends on B, the handler
  lives in A.
- It validates against each involved module's public Application query
  contracts, the same rule used for ordinary cross-module reads.
- Each involved module opens its own Marten session, but all of them
  are enlisted into one shared Postgres transaction (not one shared
  Marten session) — every module's event store lives in the same
  PostgreSQL database, just a separate schema, so this is a real ACID
  commit, not a distributed transaction to solve. Every module that
  wants to participate exposes an explicitly named repository factory
  from its own Application layer for this purpose; the coordinator
  never reaches into another module's Domain/Infrastructure/Api
  directly.

A saga/process manager with compensating actions is reserved for the
case where atomicity like this genuinely isn't available — a real
external system, a physically separate database, or a step that must
wait across a genuine async/durable boundary (e.g. something waiting
on the Bridge across multiple requests). Reaching for a saga merely
because an operation crosses a module boundary is a mistake: as long
as both sides share one database, the atomic coordinator above is
strictly safer (no partial-success window to compensate for) and no
more complex to build.

## Consistency guarantees

Every module's projections currently run in Marten's **Inline** mode:
a projection is rebuilt synchronously, in the same database transaction
as the event append that triggered it — not by a background daemon.
That means there is no eventually-consistent read path anywhere in the
system today: once a write returns successfully, any subsequent read
(aggregate or projection) is guaranteed to reflect it, including for
the coordinated cross-module writes in
[Cross-aggregate transactions](#cross-aggregate-transactions), since
every session involved is enlisted in the same Postgres transaction.

This is a deliberate latency/simplicity trade-off, not a free lunch: a
write's latency includes synchronously rebuilding every projection
subscribed to that stream. If a specific projection ever becomes a
bottleneck (a hot aggregate, an expensive read model), the fix is to
move *that* projection to Async — at which point it stops being
strongly consistent and both this section and the affected endpoint's
documentation need to say so explicitly. Consistency here is a per-
projection configuration choice, not an architectural constant, so this
doc should stay in sync with that configuration as it changes.

## Failure semantics

- **Bridge crash (before or after the Core Backend acknowledges a
  write).** The Bridge owns no persistent state, so it doesn't queue or
  replay writes across a restart — an in-flight write that doesn't get
  an acknowledged response is simply dropped, not retried. On restart
  the Bridge re-fetches current state from the Core Backend rather than
  replaying anything, consistent with it holding no durable data of its
  own. Practically: the mod/gameserver side must treat "no response" as
  "assume it didn't happen," not as "eventually delivered" — a player
  action that lands right as the Bridge crashes can silently not have
  taken effect, and the player needs to be able to retry it.
- **Projection failure.** Not a distinct failure mode — see
  [Consistency guarantees](#consistency-guarantees). Because
  projections run Inline in the same transaction as the event append,
  a projection failure fails the whole write atomically; there's no
  state where an event is committed but its projection isn't.
- **PostgreSQL unavailable.** The Core Backend fails fast: an incoming
  write is rejected immediately with an error, not buffered or queued.
  There's no write-side resilience layer inside the Core Backend
  absorbing a database outage — the caller (Bridge or Web) sees the
  failure directly and is responsible for its own retry/backoff.
- **Keycloak unavailable.** The Core Backend validates JWTs locally
  (signature and expiry) rather than calling Keycloak per request, so
  already-issued tokens keep working through an outage. What breaks is
  anything needing a *live* Keycloak call: new Web logins, new Bridge
  client-credentials grants, and new token exchanges. A Bridge holding
  an already-exchanged player token can keep operating until that token
  expires; a newly connecting player can't be authenticated until
  Keycloak recovers.

## Open questions

Not yet decided, and deliberately not documented as fact above:

- **Event ingestion identifiers.** Whether event identity (`eventId`),
  request/command identity (`requestId`), and aggregate identity
  (`aggregateId` + `expectedVersion`) are tracked as distinct concepts
  in the ingestion path, rather than one id doing duty for idempotency,
  retries, and ordering all at once. The previous description of a
  batched `POST /api/events/batch` endpoint was removed from
  [Architecture](architecture.md) because it no longer matches how
  ingestion works — a corrected description belongs here once
  ingestion is finalized.
