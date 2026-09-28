# Architecture

## Conceptual Model

CHANNEL MANAGER
  Reservation Engine
  Inventory Engine
  Rates Engine
  Core Domain
  Relational Database
  Sync / Event Layer
  OTA Adapters

This is a conceptual architecture, not a commitment to multiple deployable services.

## Domain Boundary

### Core domain
Property, RoomType, Room, RatePlan, RateValue, RateRestriction, Inventory, Reservation, ReservationRoom, Guest, ReservationGuest, Payment, ReservationChange.

### Integration domain
Channel, ChannelProperty, ChannelRoomMapping, ChannelRateMapping, SyncJob, SyncAttempt, WebhookEvent, OutboxEvent.

Core entities do not depend on OTA-specific payloads, IDs, or adapter implementations.

## Dependency Direction

API / external entry points
  -> Application services
  -> Domain rules/entities
  -> Infrastructure implementations

OTA adapters sit at the integration boundary. They translate external payloads and invoke application-level contracts. They do not directly manipulate core database tables.

## Logical Modules

Detailed responsibilities are documented in docs/architecture/MODULE_BOUNDARIES.md.

Modules:
property, room, rate, inventory, reservation, guest, payment, channel, mapping, synchronization.

These are logical modules inside one application, not separate deployable services.

## Persistence and Concrete Schema Design

The logical persistence model is documented in docs/architecture/persistence-design.md.

The concrete PostgreSQL schema proposal is documented in docs/architecture/database-schema-design.md.

The schema preserves:
- foreign-key relationships
- explicit uniqueness and idempotency constraints
- historical reservation snapshots separate from current master data
- Property + RoomType + stay date as the inventory grain
- durable WebhookEvent, OutboxEvent, SyncJob, and SyncAttempt records
- PostgreSQL row-level concurrency coordination for Inventory

No production tables or migrations are created by the schema-design milestone.

## Application Interfaces

Application-level contracts are documented in docs/architecture/application-interfaces.md.

The application layer owns:
- reservation lifecycle
- inventory allocation/release
- rate lookup
- mapping resolution
- inbound normalized reservation processing
- webhook processing
- outbound synchronization orchestration
- idempotency
- transactional outbox behavior

Repositories and infrastructure implement contracts; OTA adapters do not bypass them.

## Synchronization

### OTA to Core

OTA event/API
  -> OTA adapter
  -> WebhookEvent / normalized input
  -> idempotency
  -> application service
  -> reservation/inventory business operation
  -> OutboxEvent

### Core to OTA

Core domain change
  -> transactional OutboxEvent
  -> SyncJob
  -> OTA adapter
  -> OTA API
  -> SyncAttempt / result

An outbound network call must not hold a database transaction open.

## Inventory Authority and Concurrency

The channel manager maintains the authoritative local representation of sellable inventory.

Inventory is Property + RoomType + stay date.

The key invariant is:

> Two concurrent reservation operations must not both consume the same last available inventory.

The concrete schema design requires affected Inventory rows to be locked in deterministic order before availability is evaluated. PostgreSQL row-level locking is the proposed initial mechanism; bounded transaction retry handles transient deadlock/serialization failures where required.

Multi-night and multi-room operations lock the union of all affected rows in deterministic order.

## Transaction Boundaries

Reservation + inventory changes are one transaction.

Reservation/rate/inventory business changes + their OutboxEvent are one transaction.

Inbound WebhookEvent processing and resulting business effects are transactionally coordinated after duplicate detection.

Outbound OTA calls occur outside the local transaction.

## Approved Technology Direction

The technology direction is approved in principle:

- TypeScript + Node.js
- NestJS
- PostgreSQL
- Drizzle
- Vitest
- PostgreSQL-backed outbox/synchronization polling initially
- one modular application deployment

No Redis, Kafka, Kubernetes, microservices, or frontend implementation is introduced by this milestone.

## Failure Model

The design accounts for API unavailability, timeouts, duplicate events, partial synchronization, invalid mappings, authentication failures, rate limits, reservation modifications/cancellations, retry/backoff, and reconciliation.

## Simplicity Constraint

Start as a simple modular application with one relational database.

Do not introduce Redis, Kafka, microservices, Kubernetes, service mesh, or other operational complexity without a demonstrated requirement.

## Current Design Status

Domain model: approved in principle.

Persistence design: approved in principle.

Application interfaces: approved in principle.

Technology direction: approved in principle.

Concrete PostgreSQL schema and migration strategy: PROPOSED / pending review.

Production schema, migration files, APIs, frontend, and OTA implementations remain future milestones.
