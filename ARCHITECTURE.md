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

## Persistence Design

The persistence design is documented in docs/architecture/persistence-design.md.

The recommended database is PostgreSQL. The logical model uses:
- foreign-key relationships;
- natural uniqueness where required;
- historical reservation snapshots separate from current master data;
- Property + RoomType + stay date as the inventory grain;
- durable integration records for WebhookEvent, OutboxEvent, SyncJob, and SyncAttempt.

No production schema or migration is defined yet.

## Application Interfaces

Application-level contracts are documented in docs/architecture/application-interfaces.md.

The application layer owns:
- reservation lifecycle;
- inventory allocation/release;
- rate lookup;
- mapping resolution;
- inbound normalized reservation processing;
- webhook processing;
- outbound synchronization orchestration;
- idempotency;
- transactional outbox behavior.

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

The persistence layer must serialize or atomically guard affected Inventory rows. Candidate mechanisms include row-level locking, conditional updates, or SERIALIZABLE transactions with retry. Multi-row operations should use deterministic ordering to reduce deadlock risk.

No locking implementation is part of this milestone.

## Transaction Boundaries

Reservation + inventory changes are one transaction.

Reservation/rate/inventory business changes + their OutboxEvent are one transaction.

Inbound WebhookEvent processing and resulting business effects are transactionally coordinated after duplicate detection.

Outbound OTA calls occur outside the local transaction.

## Recommended Technology Stack

The current proposal is documented in docs/architecture/technology-stack.md and ADR 0010.

Proposed:
- TypeScript + Node.js
- NestJS
- PostgreSQL
- Drizzle
- Vitest
- PostgreSQL-backed outbox/synchronization polling initially
- one modular application deployment

This is a recommendation, not yet an approved implementation choice.

## Failure Model

The design accounts for API unavailability, timeouts, duplicate events, partial synchronization, invalid mappings, authentication failures, rate limits, reservation modifications/cancellations, retry/backoff, and reconciliation.

## Simplicity Constraint

Start as a simple modular application with one relational database.

Do not introduce Redis, Kafka, microservices, Kubernetes, service mesh, or other operational complexity without a demonstrated requirement.

## Current Design Status

Domain model: approved in principle.

Persistence design: PROPOSED.

Application interfaces: PROPOSED.

Technology stack: RECOMMENDED / pending human approval.

Production schema, migrations, APIs, frontend, and OTA implementations remain future milestones.
