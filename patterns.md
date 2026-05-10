# Patterns

## SOLID

Five OOP design principles coined by Robert C. Martin. They guide you toward code that's easier to extend, test, and refactor.

- **S — Single Responsibility Principle.** A class should have one reason to change.
  *Example:* a `FileManager` doing both file I/O and ZIP compression mixes two responsibilities — split it into `FileManager` and `ZipFileManager`.

- **O — Open/Closed Principle.** Open for extension, closed for modification — add new behavior by introducing new code, not by editing existing classes.
  *Example:* a `Shape` class with `if/elif` over shape types in `__init__` and `calculate_area` must be edited every time a shape is added. Replace it with an abstract `Shape` and subclasses (`Circle`, `Rectangle`, …) that implement `calculate_area` themselves.

- **L — Liskov Substitution Principle.** Subtypes must be usable wherever their base type is expected, without breaking behavior.
  *Example:* `Square(Rectangle)` that overrides `__setattr__` to force `width == height` breaks the assumption that you can change them independently. Better: make `Rectangle` and `Square` independent subclasses of a common `Shape`.

- **I — Interface Segregation Principle.** Clients shouldn't depend on methods they don't use — prefer many small interfaces over one fat one.
  *Example:* a single `Printer` interface forcing `OldPrinter` to implement `fax()` and `scan()` (raising `NotImplementedError`). Split into `Printer`, `Fax`, `Scanner`; `OldPrinter` implements only `Printer`.

- **D — Dependency Inversion Principle.** High-level modules should depend on abstractions, not on concrete low-level modules.
  *Example:* a `FrontEnd` that instantiates a concrete `BackEnd` is hard to swap or test. Introduce a `DataSource` abstraction with `Database` / `API` implementations, and inject it into `FrontEnd`.

In Python this commonly looks like: small classes with one job, dependency injection via constructor arguments, `Protocol` types instead of concrete classes, and avoiding `isinstance` checks that branch on type.

**Source:** [SOLID Principles: Improve Object-Oriented Design in Python — Real Python](https://realpython.com/solid-principles-python/)

---

## Domain-Driven Design (DDD)

DDD is an approach to building complex software where the **domain model** (the business reality) drives the design of the code. It separates strategic concerns (how to break a big domain into manageable parts) from tactical ones (how to express each part in code).

### Strategic patterns

- **Ubiquitous Language** — one shared vocabulary between domain experts, devs, and code. If the business calls it a "Wallet", the class is `Wallet`, not `AccountBalanceHolder`.
- **Bounded Context** — an explicit boundary inside which a model has one consistent meaning. "Customer" in *Billing* and "Customer" in *Support* may have different fields and rules — they live in different bounded contexts.
- **Context Map** — describes how bounded contexts relate (shared kernel, customer–supplier, anti-corruption layer, etc.).

### Tactical patterns

- **Entity** — has identity that persists over time (`User#42`). Two entities with the same fields but different IDs are different.
- **Value Object** — defined entirely by its attributes; no identity. `Money(100, "GBP")` equals any other `Money(100, "GBP")`. Immutable.
- **Aggregate** — a cluster of entities + value objects treated as a single unit for changes. The **Aggregate Root** is the only entity outsiders may reference; it enforces invariants for the whole cluster (e.g. `Order` is the root, `OrderLine` is inside).
- **Repository** — collection-like abstraction for persisting/loading aggregates. Hides the DB.
- **Domain Event** — something meaningful that happened in the domain (`OrderPlaced`, `PaymentFailed`). Other parts of the system react to it asynchronously.
- **Domain Service** — logic that doesn't naturally belong on any single entity (e.g. money transfer between two accounts).

DDD pairs naturally with microservices: **one bounded context → one service**. The service owns its data, exposes its model only via APIs/events, and uses anti-corruption layers when integrating with foreign models.

When **not** to use DDD: simple CRUD apps. The overhead doesn't pay off when the domain has no real complexity.

**Articles:**

- [Domain-Driven Design (DDD) Demystified: Bounded Contexts, Aggregates & Entities Explained — Medium](https://medium.com/@priyansu011/domain-driven-design-ddd-demystified-bounded-contexts-aggregates-entities-explained-43676d981953)
- [Domain Driven Design — The Bounded Context — Medium (John Boldt)](https://medium.com/@johnboldt_53034/domain-driven-design-the-bounded-context-1a5ea7bcb4a4)
- [bliki: Bounded Context — Martin Fowler](https://martinfowler.com/bliki/BoundedContext.html)

---

## Stability Patterns

A toolkit for building services that survive partial failures in distributed systems. Most are from Michael Nygard's *Release It!*.

### Timeout

Every remote call must have a finite, **explicitly set** timeout. Default-infinite timeouts are how slow downstreams kill upstreams. Rule: a caller's timeout should be ≤ the callee's max latency budget.

### Retry (with backoff and jitter)

Retry transient failures (`5xx`, network blip, timeout). Two essentials:

- **Exponential backoff** — wait `base * 2^attempt`. Don't hammer a struggling service.
- **Jitter** — add randomness so retries from many clients don't synchronize and create a thundering herd.

Don't retry: `4xx` (it'll fail again), non-idempotent operations (you'll double-charge), or after the operation deadline expired.

### Circuit Breaker

Wraps a remote call. Three states:

```text
CLOSED   ─ failures > threshold ─→  OPEN
  ↑                                  │
  │                              wait timeout
  │                                  │
  └── success ── HALF-OPEN ←─────────┘
                  (probe with limited requests)
```

- **Closed** — normal operation; track failure rate.
- **Open** — fail fast for a cooldown period; don't even try the broken service.
- **Half-Open** — let a few requests through; if they succeed, close; if they fail, re-open.

Why: protects the broken service from being hammered while it's down, and protects callers from waiting on long timeouts.

### Bulkhead

Isolate failures by partitioning resources, like watertight compartments on a ship. Per-dependency thread pools / connection pools / semaphore limits. If service A becomes slow, only the A pool fills up — calls to B keep working.

### Fallback

Define what to do when a dependency is unavailable: cached value, default response, degraded feature. Better than failing the whole request.

### Rate Limiting / Load Shedding

Cap incoming traffic before it overwhelms downstream — token bucket / leaky bucket. Better to reject 5% with `429` than collapse 100%.

These patterns **compose**: typical chain is `Bulkhead → CircuitBreaker → Timeout → Retry → Fallback`. Libraries: **Resilience4j** (Java), **Polly** (.NET), **tenacity** + **circuitbreaker** (Python).

**Articles:**

- [Making REST Microservices Resilient: Bulkhead, Retry & Circuit Breaker in Practice — dev.to](https://dev.to/athulmr/making-rest-microservices-resilient-bulkhead-retry-circuit-breaker-in-practice-1mpl)
- [Resilience in Microservices: Bulkhead vs Circuit Breaker — Medium](https://medium.com/@parserdigital/resilience-in-microservices-bulkhead-vs-circuit-breaker-54364c1f9d53)
- [Stability Patterns: Circuit Breakers — Medium](https://medium.com/@maxi.gkd/stability-patterns-circuit-breakers-80aa69a06cb0)

---

## CQRS — Command Query Responsibility Segregation

Split the model used to **change** state (commands) from the model used to **read** state (queries). Two separate code paths, often two separate data stores.

```text
                ┌──────────────┐
   Command  ──→ │  Write model │ ──── events ────┐
                └──────────────┘                 │
                                                 ▼
                ┌──────────────┐         ┌──────────────┐
   Query    ──→ │  Read model  │ ←────── │  Projections │
                └──────────────┘         └──────────────┘
```

Why split:

- **Different shapes** — writes need a normalised model with strong invariants; reads need denormalised, query-optimised views (per-screen).
- **Different scaling** — read load usually dwarfs writes; scale them independently.
- **Different tech** — write side might be Postgres + DDD aggregates; read side might be Elasticsearch / Redis / a materialised view.

Trade-offs:

- **Eventual consistency** — read model lags behind write model (the read projection is updated asynchronously). User clicks "save" → re-fetches → sees old value. Plan for this.
- **More moving parts** — two models, projection workers, message bus.
- **Schema evolution** — both models must evolve in lockstep.

**When to use**: complex domains with high read/write asymmetry, collaborative domains, reporting needs that hurt the write model's performance. **When not**: simple CRUD — adds risk without benefit.

CQRS is often paired with **Event Sourcing** but they are independent: you can use either alone.

**Articles:**

- [CQRS Design Pattern in Microservices Architectures — Medium (Mehmet Ozkaya)](https://medium.com/design-microservices-architecture-with-patterns/cqrs-design-pattern-in-microservices-architectures-5d41e359768c)
- [The Command and Query Responsibility Segregation (CQRS) Pattern — Medium](https://henriquesd.medium.com/the-command-and-query-responsibility-segregation-cqrs-pattern-16cb7704c809)
- [bliki: CQRS — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)

---

## Event Sourcing

Don't store **current state**; store the **sequence of events** that led to it. Current state is derived by replaying events.

```text
Account 42 events:
  AccountOpened(balance=0)
  Deposited(100)
  Deposited(50)
  Withdrawn(30)
  → state: balance = 120
```

Compare with classic CRUD: a CRUD account just stores `balance = 120` — you've lost *how* it got there.

Benefits:

- **Full audit trail** — every state change is a first-class, queryable record. Free for compliance/finance.
- **Time travel** — replay events up to any historical moment.
- **Multiple projections** — derive different read models from the same events (one for users, one for analytics, one for fraud).
- **Natural fit for messaging** — events are exactly what other services subscribe to.

Costs:

- **Schema evolution is hard** — events are immutable; renaming a field means versioned event types and upcasting on read.
- **Rebuild from scratch is slow** — million events take time to replay; use **snapshots** (periodic state checkpoints) to speed up.
- **Eventual consistency** — projections lag behind the event store.
- **Steep learning curve** — most teams underestimate operational complexity.

Pairs naturally with CQRS: events update read-model projections asynchronously.

Use it for: ledgers, banking, order workflows, anything where "what happened, in order" is core to the domain. Avoid it for: simple CRUD; teams without the discipline to evolve event schemas.

**Articles:**

- [Event Sourcing Pattern for Beginners — Medium](https://medium.com/@subham11/event-sourcing-pattern-for-beginners-ea4ca173a733)
- [Event Sourcing Pattern in Microservices Architectures — Medium](https://medium.com/design-microservices-architecture-with-patterns/event-sourcing-pattern-in-microservices-architectures-e72bf0fc9274)
- [Event Sourcing — Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
