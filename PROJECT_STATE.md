# Project State

## Current Phase
**Phase 2 — Concrete PostgreSQL Schema and Migration Strategy (Design Only)**

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

### APPROVED IN PRINCIPLE / ARCHITECTURAL DIRECTION
- Core hotel domain is independent from OTA-specific code.
- OTA integrations are isolated behind adapters.
- Local inventory is authoritative.
- External reservation identity must be idempotent.
- Reservation/inventory effects require transactional consistency.
- Synchronization must be retryable and auditable.
- Architecture is a modular monolith, not a microservice system.
- Event sourcing, Kafka, Kubernetes, Redis initially, and unnecessary infrastructure are out of scope.
- Approved technology direction: TypeScript, Node.js, NestJS, PostgreSQL, Drizzle, Vitest, PostgreSQL-backed transactional outbox initially.

### APPROVED IN PRINCIPLE / DOMAIN MODEL
- RoomType is the canonical sellable inventory unit.
- Room represents physical rooms and has ACTIVE, OUT_OF_ORDER, OUT_OF_SERVICE, and INACTIVE status.
- RatePlan is a stable commercial definition attached to a RoomType.
- Inventory is Property + RoomType + stay date.
- Reservation current status is NEW, CONFIRMED, or CANCELLED.
- MODIFIED is represented through ReservationChange history.
- External reservation identity is Channel + external_reservation_id.
- Guest identity is reusable through ReservationGuest.
- Payment excludes raw card data and authentication secrets.
- RateValue and RateRestriction separate date-specific commercial values from RatePlan.
- PMS stay states remain outside the initial Channel Manager core.

## Concrete Schema Design

The concrete PostgreSQL proposal is documented in docs/architecture/database-schema-design.md.

Key schema decisions:
- UUID internal identities.
- date for stay dates; timestamptz for event/audit timestamps.
- exact monetary values via PostgreSQL numeric.
- text statuses with CHECK constraints rather than PostgreSQL enums initially.
- explicit partial uniqueness for optional external identifiers.
- default FK deletion behavior is restrictive; historical records are retained.
- no independently editable available-inventory column.
- ReservationRoom contains booking-time commercial snapshots.
- ReservationChange is append-oriented history, not event sourcing.
- WebhookEvent identity is distinct from Reservation identity.
- OutboxEvent -> SyncJob -> SyncAttempt is durable PostgreSQL-backed work state.

## Inventory Concurrency Design

The concrete design requires:
1. determine all affected Property + RoomType + stay-date Inventory rows;
2. union/deduplicate rows for multi-room and multi-night operations;
3. lock affected rows in deterministic order;
4. validate availability;
5. apply reservation/inventory changes atomically;
6. append ReservationChange;
7. insert required OutboxEvent;
8. commit.

A failed availability check commits none of the business operation.

Concurrent transactions serialize on shared Inventory rows. Bounded transaction retry is permitted for transient deadlock/serialization failures.

No implementation SQL has been created.

## Reservation / Idempotency Design

- Internal identity: reservation_id.
- External reservation identity: channel_id + external_reservation_id.
- Webhook identity: channel_property_id + external_event_id.
- ReservationChange identity: reservation_id + sequence_number.
- SyncAttempt identity: sync_job_id + attempt_number.

Duplicate delivery must not duplicate inventory effects.

Late external changes must not blindly overwrite newer state. Exact OTA sequencing remains TO VERIFY.

## Historical Data

Reservations retain booking-time facts independently of mutable master data:
- RoomType and RatePlan labels/identities
- occupancy
- dates and quantities
- nightly pricing
- currency
- taxes/fees/discounts
- meal/package facts
- cancellation-policy snapshot

Current RateValue/RateRestriction rows are not required to reconstruct historical reservation economics.

## Migration Strategy

Design only:
- migrations owned by the application repository and version-controlled;
- Drizzle is the approved migration-toolchain direction;
- migration SQL must be reviewed before production;
- production migrations run as explicit deployment steps;
- prefer expand/contract and forward fixes over destructive rollback;
- destructive changes require explicit review and staged handling;
- deterministic idempotent reference seeds;
- real PostgreSQL for integration/concurrency tests.

No migration files have been created.

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

### Implementation detail
- Exact Drizzle schema/configuration conventions.
- Exact PostgreSQL deployment/provider.
- Exact runtime/deployment topology.
- Final SQL form of concurrency guards and bounded retry policy after integration tests.

## Conflicts Discovered

No domain or architecture conflict was discovered.

The concrete schema preserves the approved domain semantics and does not introduce a new deployment boundary or business rule.

## Current Objective

Review the concrete PostgreSQL schema and migration strategy before any implementation begins.

## Next Milestone

No implementation milestone is authorized yet.

After this schema design is reviewed, the next milestone should be explicitly selected. A likely candidate is implementation planning/finalization of the Drizzle schema and migration workflow, but no such work should begin until this design is approved or corrected.
