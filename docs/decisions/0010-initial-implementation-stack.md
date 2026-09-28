# ADR 0010 — Initial Implementation Stack

## Status
Proposed — pending human approval

## Context
The approved domain requires a relational persistence model, strong transactional consistency around inventory/reservations, durable synchronization, webhooks, idempotency, and modest operational complexity for a small development effort.

## Decision
Propose:
- TypeScript on Node.js
- NestJS as the modular application framework
- PostgreSQL as the relational database
- Drizzle as the SQL-close typed data-access layer
- Vitest for unit/application testing
- PostgreSQL-backed outbox/synchronization polling initially
- one modular application deployment with PostgreSQL

No Redis, Kafka, Kubernetes, microservices, or separate job broker is required initially.

## Rationale
PostgreSQL directly supports the relational constraints and concurrency controls required by the inventory model. NestJS provides explicit application module and dependency-injection structure matching the approved logical modules. Drizzle keeps concurrency-sensitive SQL visible while retaining TypeScript typing. A PostgreSQL-backed outbox avoids introducing another stateful infrastructure component before workload justifies it.

## Alternatives Considered
- FastAPI/Python: credible; particularly attractive if the team is substantially stronger in Python.
- Go: credible and operationally simple, with strong explicit transaction support, but requires more application-architecture plumbing.
- Prisma: credible TypeScript/PostgreSQL ORM; less attractive here because the design benefits from SQL-visible concurrency behavior and the current major-version transition should be avoided as a reason to choose it.
- Kysely: credible SQL-first TypeScript alternative; remains a valid substitution if the team prefers a thinner query-builder layer.

## Consequences
Positive:
- strong fit for transactional inventory;
- clear modular monolith;
- low initial infrastructure footprint;
- OTA adapter work can remain isolated;
- typed application contracts.

Tradeoffs:
- NestJS adds framework ceremony;
- Node/TypeScript is not intended for CPU-heavy workloads;
- PostgreSQL-backed job polling will eventually need evaluation against real synchronization volume;
- Drizzle requires engineers to understand SQL and database behavior rather than hiding it.

## Reconsider If
Re-evaluate the stack if team expertise strongly favors another runtime, actual OTA requirements impose a different runtime, synchronization workload requires a dedicated queue, or operational requirements materially change.
