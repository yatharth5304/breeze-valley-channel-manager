# Module Boundaries

## Status
**PROPOSED** — logical application boundaries for review. These are not separate deployable services.

## Dependency Direction

API / external entry points -> Application services -> Domain model/rules -> Infrastructure, persistence, and external adapters.

Infrastructure implements contracts required by application/domain code. Core business modules must not import concrete OTA adapters.

## property
**Responsibility:** property identity and configuration.  
**Owns:** Property.  
**Services:** property lifecycle/configuration.  
**Allowed:** shared primitives and required application orchestration.  
**Prohibited:** OTA-specific logic and direct OTA calls.

## room
**Responsibility:** RoomType and physical Room master data/status.  
**Owns:** RoomType, Room.  
**Services:** room-type management, room management, room status changes.  
**Allowed:** property.  
**Prohibited:** OTA payloads and synchronization execution.

## rate
**Responsibility:** RatePlan and rate/restriction rules.  
**Owns:** RatePlan and rate value objects/rules.  
**Services:** rate-plan and rate operations.  
**Allowed:** property and room-type identity.  
**Prohibited:** direct channel calls and mapping storage.

## inventory
**Responsibility:** sellable RoomType availability by stay date.  
**Owns:** Inventory and inventory adjustment/availability rules.  
**Services:** availability, reservation consumption/release, manual adjustments, blocking.  
**Allowed:** property, room capacity/status, reservation application contracts/events.  
**Prohibited:** direct OTA calls and OTA payloads.

## reservation
**Responsibility:** booking aggregate and lifecycle.  
**Owns:** Reservation, ReservationRoom.  
**Services:** create, confirm, modify, cancel, retrieve.  
**Allowed:** property, room/rate identity, guest, payment, inventory application contract.  
**Prohibited:** direct OTA calls and OTA-specific models.

## guest
**Responsibility:** guest identity/contact data.  
**Owns:** Guest.  
**Services:** guest lifecycle and reservation-facing guest operations.  
**Prohibited:** OTA-specific guest objects.

## payment
**Responsibility:** minimal reservation payment state required by channel management.  
**Owns:** Payment.  
**Services:** payment status/reference handling and reservation payment summaries.  
**Prohibited:** card vaulting, raw card data, accounting ledger, direct OTA calls.

## channel
**Responsibility:** generic external channels and property-channel connections.  
**Owns:** Channel, ChannelProperty.  
**Services:** channel configuration and connection lifecycle.  
**Allowed:** property and integration infrastructure.  
**Prohibited:** direct mutation of core reservation/inventory state.

## mapping
**Responsibility:** internal-to-external sellable entity mappings.  
**Owns:** ChannelRoomMapping, ChannelRateMapping.  
**Services:** mapping lookup, validation, activation/deactivation.  
**Allowed:** channel, room, rate.  
**Prohibited:** assuming internal IDs equal external IDs or implementing transport.

## synchronization
**Responsibility:** inbound/outbound processing and reliability.  
**Owns:** SyncJob, SyncAttempt, WebhookEvent, OutboxEvent.  
**Services:** inbound handling, outbox dispatch, retries, failure classification, reconciliation orchestration.  
**Allowed:** application contracts from reservation, inventory, rate, channel, mapping, plus OTA adapter ports.  
**Prohibited:** core hotel rules and direct database/table mutation that bypasses application/domain services.

## Cross-Module Rules

1. Reservation changes may affect inventory through an application/domain service contract.
2. Inventory must not directly depend on Reservation implementation; orchestration coordinates the business operation.
3. Core modules depend on abstractions, not concrete infrastructure.
4. Integration modules may call core application contracts; core modules must not depend on a particular adapter.
5. Keep shared primitives small; do not create a catch-all shared business module.
6. OTA adapters cannot directly mutate core tables. They must invoke application/domain services.
7. These are logical modules inside one application, not instructions to create microservices.

## Transaction Guidance

Reservation and Inventory participate in one business operation when a booking changes sellable availability. The application layer must coordinate the operation so reservation state and inventory effect cannot silently diverge.

The exact aggregate implementation and persistence transaction mechanism are **TO VERIFY** after technology selection.

## OTA Adapter Boundary

Adapters should implement capability-specific ports such as reservation intake, inventory publication, rate/restriction publication, event handling, and reconciliation. Exact ports remain **TO VERIFY** against actual OTA contracts.
