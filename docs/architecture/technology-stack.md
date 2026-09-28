# Technology Stack Proposal

## Status

**RECOMMENDED — pending human approval.**

This proposal is based on the approved domain model and the actual first-product requirements: one property, modular monolith, relational consistency, OTA webhooks, outbound synchronization, idempotency, transactional outbox, and a small development effort.

It intentionally does not optimize for hypothetical SaaS scale.

## Recommended Stack

| Area | Recommendation | Reason |
|---|---|---|
| Backend language/runtime | TypeScript on Node.js | Strong domain/application typing, good HTTP/integration ecosystem, productive for a small team |
| Application framework | NestJS | Provides explicit modules/providers/dependency injection and a structure well suited to the approved logical modules |
| Database | PostgreSQL | Strong relational integrity, transactions, constraints, row-level locking, and mature operational ecosystem |
| Data access | Drizzle ORM + PostgreSQL driver | SQL-close model, TypeScript typing, explicit transaction support, and less abstraction around concurrency-sensitive SQL |
| Migration tool | Drizzle Kit or equivalent SQL-first migration workflow | Fits schema-as-code while retaining visibility into SQL; exact tool use begins after approval |
| Testing | Vitest + integration tests against PostgreSQL | Fast TypeScript unit testing plus real database tests for invariants and transactions |
| Background processing | PostgreSQL-backed outbox polling in the same application | Avoids adding Redis/broker infrastructure at initial scale |
| Frontend | Not selected yet | Frontend is not required to resolve backend/persistence architecture |
| Deployment/runtime | Single deployable application + managed PostgreSQL or equivalent PostgreSQL service | Simple operational model matching a single-property system |

Current framework documentation confirms NestJS is TypeScript/Node.js-oriented and provides application modules, dependency injection, testing, scheduling, and queue integration. It can run on Express or Fastify. citeturn0search4turn0search11

PostgreSQL supports the transactional and row-level locking semantics needed for inventory correctness. citeturn2search0

Drizzle provides transactions and PostgreSQL transaction configuration, including isolation-level controls. citeturn1search3

Vitest provides TypeScript-friendly test APIs suitable for the proposed unit-test layer. citeturn1search9turn1search10

## Why TypeScript + NestJS

### Fit

The project has many explicit boundaries:
- property
- room
- rate
- inventory
- reservation
- guest
- payment
- channel
- mapping
- synchronization

NestJS's module/provider architecture maps naturally to these logical boundaries without requiring separate services. Its current documentation explicitly covers modules, dependency injection, testing, scheduling, queues, and HTTP capabilities. citeturn0search11

TypeScript gives the application contracts a strong type system while remaining practical for external API integration work.

### Tradeoffs

- More framework structure than a minimal Node HTTP framework.
- Decorator/module conventions add some ceremony.
- Node.js is not ideal for CPU-heavy workloads, but this product is predominantly I/O-bound.

### Alternative: FastAPI/Python

FastAPI offers strong request typing, validation, and automatic OpenAPI generation. citeturn0search9 It is a credible option, especially if the development team is substantially stronger in Python.

For this project, the main tradeoff is that the application has a large number of explicit modular contracts and long-lived integration boundaries; NestJS provides more opinionated application structure out of the box.

### Alternative: Go

Go's standard library has explicit relational transaction support through database/sql and sql.Tx. citeturn0search0turn0search1

Go would be attractive for a small, operationally simple service with strong concurrency characteristics. The tradeoff is more application-architecture plumbing and less direct alignment with a TypeScript-centric API/frontend ecosystem if that becomes relevant later.

## Why PostgreSQL

This is the strongest part of the recommendation.

The domain is relational:
- parent/child entities
- foreign keys
- uniqueness constraints
- date-based inventory
- reservation lines
- mappings
- durable synchronization records

Most importantly, inventory correctness requires transactional coordination under concurrency. PostgreSQL supports row-level locks and transaction semantics appropriate for this invariant. citeturn2search0

No separate cache/database is justified initially.

## Why Drizzle Rather Than a Heavier ORM

The inventory and synchronization areas will eventually need deliberate SQL behavior.

Drizzle provides typed queries while keeping SQL concepts visible and supports explicit transaction configuration. citeturn1search3

That is preferable here to an ORM that encourages hiding important concurrency behavior behind generic CRUD abstractions.

### Alternative: Prisma

Prisma is a credible TypeScript/PostgreSQL option and provides transaction support. citeturn1search2

The tradeoff is abstraction around SQL and transaction configuration. Current Prisma documentation also shows significant version/API transition activity, with Prisma 8 documented as a current release candidate while Prisma 7 remains supported. citeturn1search5

That does not make Prisma unsuitable; it makes a SQL-close approach easier to justify for this particular inventory-heavy domain.

### Alternative: Kysely

Kysely is a strong typed SQL query-builder alternative. Its type safety is based on an explicit database interface. citeturn1search12

It would be reasonable if the team prefers even less ORM abstraction. Drizzle is recommended because it provides a useful middle ground between SQL control and schema/type ergonomics.

## Background Jobs: Do Not Add Redis Yet

The system already has OutboxEvent, SyncJob, and SyncAttempt.

For the first implementation, a database-backed worker can periodically claim pending SyncJob/Outbox work.

This avoids:
- Redis
- RabbitMQ
- Kafka
- another operational service

PostgreSQL row locking can support multiple workers later if needed; PostgreSQL documents SKIP LOCKED as suitable for queue-like access patterns, while noting that it is not appropriate as a general-purpose consistency mechanism. citeturn2search3

If synchronization volume, latency, or worker isolation later exceeds what PostgreSQL polling comfortably provides, a dedicated job system can be introduced behind the synchronization module.

NestJS has BullMQ integration available if that future requirement arises, but using it now would introduce Redis and additional infrastructure without a demonstrated need. citeturn0search3

## Frontend

**Not selected in this milestone.**

The domain/persistence/application design does not require a frontend technology decision.

When the admin UI becomes a separate milestone, a frontend stack can be selected based on actual workflows.

## Deployment

**PROPOSED:** one application deployment with one PostgreSQL database/service.

The application may run as a normal Node.js process or container. Containerization is an operational choice, not a requirement for the architecture.

Do not introduce Kubernetes or a service mesh.

## Testing Strategy

### Unit tests
- inventory calculation
- reservation lifecycle
- modification deltas
- cancellation
- rate/restriction evaluation
- mapping rules
- idempotency decisions

### PostgreSQL integration tests
- concurrent inventory consumption
- transaction rollback
- uniqueness/idempotency constraints
- outbox atomicity
- reservation + inventory consistency
- synchronization retry state

Vitest is recommended for the unit/application test layer. Database integration tests should use a real PostgreSQL instance rather than replacing transactional behavior with mocks.

## What Would Change This Recommendation?

Choose differently if:
- the development team has a substantially stronger Python or Go capability;
- the product becomes CPU-heavy rather than integration/I/O-heavy;
- a different database becomes a demonstrated requirement;
- synchronization volume requires a dedicated durable queue;
- a frontend becomes a primary product surface and changes the team/runtime strategy;
- actual OTA SDK requirements impose a materially different runtime constraint.

None of those conditions is established today.

## Explicit Non-Decisions

This milestone does not select:
- cloud provider
- hosting vendor
- frontend framework
- authentication provider
- Redis
- Kafka
- Kubernetes
- microservices
- OTA SDKs
- production migration layout

## Proposed Runtime Shape

One modular application:

API / webhook entry points
    |
Application modules
    |
Domain
    |
PostgreSQL repositories

Synchronization worker loop
    |
Outbox / SyncJob
    |
OTA adapter ports

The worker can initially run as part of the same deployable application/process model. It should remain logically separated from request handling even if deployed together.
