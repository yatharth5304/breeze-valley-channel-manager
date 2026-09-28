# Project State

## Current Phase
**Phase 1 — Persistence, Application Interfaces, and Technology Proposal**

## Product
Custom channel manager for Breeze Valley, initially one property in Panchgani/Mahabaleshwar, Maharashtra, India.

The domain remains property-generic. Multi-property workflows are not being implemented yet.

## Target Channels
- Agoda
- Booking.com
- MakeMyTrip
- Goibibo
- Airbnb

## Completed Work

### DECIDED / ARCHITECTURAL CONSTRAINTS
- Core hotel domain is independent from OTA-specific code.
- OTA integrations are isolated behind adapters.
- Local inventory is authoritative.
- External reservation identity must be idempotent.
- Reservation/inventory effects require transactional consistency.
- Synchronization must be retryable and auditable.
- Architecture is a modular monolith, not a microservice system.
- Event sourcing, Kafka, Kubernetes, and unnecessary infrastructure are explicitly out of scope.

### DECIDED / DOMAIN MODEL
- RoomType is the canonical sellable inventory unit.
- Room represents physical rooms and has ACTIVE, OUT_OF_ORDER, OUT_OF_SERVICE, and INACTIVE status.
- RatePlan is a stable commercial definition attached to one RoomType.
- Inventory is Property + RoomType + stay date.
- Reservation current status is NEW, CONFIRMED, or CANCELLED.
- MODIFIED is represented through ReservationChange history.
- External reservation identity is Channel + external_reservation_id.
- Guest identity is reusable through ReservationGuest.
- Payment excludes raw card data and authentication secrets.
- RateValue and RateRestriction separate date-specific commercial values from RatePlan.
- PMS stay states remain outside the initial Channel Manager core.

### PROPOSED / PERSISTENCE
- PostgreSQL is the proposed relational persistence technology.
- Inventory rows are the concurrency coordination point for Property + RoomType + stay date.
- Reservation + inventory effects + required outbox intent form one local transaction.
- Historical reservation commercial facts remain separate from mutable master data.
- External identities and mapping identifiers use explicit uniqueness constraints.
- Available inventory is derived rather than an independently editable source of truth.
- Synchronization records are durable and retryable.

### PROPOSED / APPLICATION INTERFACES
- ReservationService
- InventoryAvailability
- InventoryAllocation
- RateService
- ChannelMappingService
- ExternalReservationProcessor
- WebhookProcessor
- InventorySyncService
- RateSyncService
- ChannelAdapter capability ports
- IdempotencyService
- TransactionManager
- OutboxService
- business-oriented repository ports

OTA-specific models remain behind adapters.

### RECOMMENDED / TECHNOLOGY STACK
- TypeScript + Node.js
- NestJS
- PostgreSQL
- Drizzle
- Vitest
- PostgreSQL-backed outbox/synchronization polling initially
- one modular application deployment

This recommendation is pending human approval and is documented in docs/architecture/technology-stack.md and ADR 0010.

## Important Invariants

1. Two concurrent reservations must not consume the same last available inventory.
2. Reservation and inventory changes must not commit independently.
3. Outbox intent must commit with the business change it represents.
4. Duplicate external reservations/events must not create duplicate business effects.
5. Historical reservation commercial meaning must survive master-data changes.
6. OTA adapters must not directly mutate core persistence.
7. Available inventory must not be independently edited as a second source of truth.
8. Outbound network calls must not hold local database transactions open.

## Remaining TO VERIFY

### Business policy
- Whether overbooking is permitted and the allowance/policy.
- Exact inventory override semantics.
- Whether temporary reservation holds are required.

### Commercial model
- Exact occupancy dimensions and child-age pricing.
- Exact cancellation-policy structure.
- Whether advance-booking restrictions are required initially.
- Exact MinLOS/MaxLOS/CTA/CTD requirements.

### Reservation/integration
- External modification/version sequencing available from each OTA.
- Treatment of reservations received already cancelled.
- Exact timezone/date semantics per channel.
- Required payment/refund fields per channel.
- Guest retention/privacy and deduplication/merge policy.
- Exact OTA adapter capability contracts after channel documentation review.

### Implementation
- Exact migration tooling/workflow.
- Exact PostgreSQL deployment/provider.
- Exact runtime/deployment topology.
- Concrete concurrency mechanism after implementation tests.

## Conflicts Discovered

No conflict with the approved architecture was found.

The persistence design adds no new deployment boundary and preserves the existing core/integration separation.

## Current Objective

Review and approve the persistence design, application interfaces, and recommended technology stack before implementation.

## Next Milestone

After human approval:
1. lock the technology choices;
2. design the concrete persistence schema and migration strategy;
3. establish the application skeleton/module structure;
4. implement core domain/application behavior and database access;
5. test inventory concurrency/idempotency;
6. only afterward begin isolated OTA adapter implementation.
