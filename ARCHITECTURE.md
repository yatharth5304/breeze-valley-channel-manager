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

This is a conceptual architecture, not a commitment to a particular framework or deployment topology.

## Domain Boundary

### Core domain
Property, RoomType, Room, RatePlan, Inventory, Reservation, ReservationRoom, Guest, Payment.

### Integration domain
Channel, ChannelProperty, ChannelRoomMapping, ChannelRateMapping, SyncJob, SyncAttempt, WebhookEvent, OutboxEvent.

Core entities do not depend on OTA-specific payloads, IDs, or adapter implementations.

## Dependency Direction

API / external entry points
  -> Application services
  -> Domain rules/entities
  -> Infrastructure implementations

OTA adapters sit at the infrastructure/integration boundary. They translate external payloads and invoke application-level contracts. They do not directly manipulate core database tables.

## Module Boundaries

Detailed logical module responsibilities and allowed/prohibited dependencies are documented in docs/architecture/MODULE_BOUNDARIES.md.

The modules are property, room, rate, inventory, reservation, guest, payment, channel, mapping, and synchronization. These are logical modules, not separate deployable services.

## Synchronization

### OTA to Core
OTA event/API
  -> OTA adapter
  -> WebhookEvent / normalized input
  -> idempotency
  -> application service
  -> reservation/inventory business operation
  -> local event/audit records

### Core to OTA
Core domain change
  -> transactional OutboxEvent
  -> SyncJob
  -> OTA adapter
  -> OTA API
  -> SyncAttempt / result

## Inventory Authority

The channel manager maintains the authoritative local representation of sellable inventory.

Inventory is modeled around Property + RoomType + stay date. Physical Room status can contribute to capacity, while explicit inventory controls can adjust sellable quantity.

OTAs are external distribution channels, not peer databases for the internal domain.

## Idempotency

For external reservations, canonical identity is Channel + external_reservation_id.

Inbound event IDs, when supplied, are separately deduplicated at the WebhookEvent boundary.

## Failure Model

The design accounts for API unavailability, timeouts, duplicate events, partial synchronization, invalid mappings, authentication failures, rate limits, reservation modifications/cancellations, retry/backoff, and reconciliation.

## Simplicity Constraint

Start as a simple application with clear module boundaries. Introduce queues, separate services, or infrastructure components only when a concrete reliability, scale, or operational requirement justifies them.

## Current Design Status

Domain entities and module boundaries are PROPOSED pending review. Technology choices and persistence schema remain intentionally open.
