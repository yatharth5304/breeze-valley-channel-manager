# Project State

## Current Phase
**Phase 0 — Domain Model and Application Boundaries**

## Product
Custom channel manager for Breeze Valley, initially one property in Panchgani/Mahabaleshwar, Maharashtra, India.

The domain model remains property-generic so additional properties can be supported later without hard-coding Breeze Valley into core entities. Multi-property workflows are not being implemented yet.

## Target Channels
- Agoda
- Booking.com
- MakeMyTrip
- Goibibo
- Airbnb

## Completed Work

### DECIDED / EXISTING ARCHITECTURAL CONSTRAINTS
- Core hotel domain is independent from OTA-specific code.
- OTA integrations are isolated behind adapters.
- Local inventory is authoritative for the channel manager.
- External reservation identity must be idempotent.
- Reservation/inventory effects require transactional consistency.
- Synchronization must be retryable and auditable.
- The architecture remains a simple modular application rather than premature microservices/infrastructure.

### PROPOSED / DOMAIN DESIGN
- Property -> RoomType -> Room hierarchy.
- RoomType is the canonical sellable inventory unit.
- RatePlan belongs to a RoomType and represents a pricing/package rule.
- Inventory is modeled per Property + RoomType + stay date.
- Reservation owns ReservationRoom lines and references Guest and Payment.
- ChannelProperty connects a generic Channel to a Property.
- ChannelRoomMapping maps RoomType to an external room/listing ID.
- ChannelRateMapping maps RatePlan to an external rate-plan ID.
- WebhookEvent is the inbound durable/idempotency boundary.
- OutboxEvent is the outbound intent boundary.
- SyncJob represents synchronization work; SyncAttempt records each execution.
- MODIFIED is treated primarily as an auditable reservation change/version, not a permanent lifecycle state.
- NO_SHOW, CHECKED_IN, and CHECKED_OUT are PMS-adjacent and not required by the initial channel-manager core.

### PROPOSED / MODULE BOUNDARIES
Logical modules are property, room, rate, inventory, reservation, guest, payment, channel, mapping, and synchronization.

API/external entry points call application services. Core domain rules sit below application orchestration. Infrastructure and OTA adapters implement integration/persistence contracts.

OTA adapters may call core application contracts but may not directly mutate core persistence.

## Open Questions

### TO VERIFY
- Exact inventory override and overbooking semantics.
- Whether physical Room-level availability is needed beyond capacity derivation/status.
- Reservation snapshot/version strategy for booked prices, taxes, and occupancy.
- Guest merge/deduplication policy.
- Exact cancellation-policy and rate-restriction structures.
- Required payment statuses and OTA payment semantics.
- Webhook payload retention/privacy requirements.
- Exact OTA adapter ports after actual channel contracts are reviewed.
- Persistence uniqueness/transaction implementation after technology selection.

## Current Objective
Review and confirm the proposed domain model and module boundaries before implementation technology is selected.

## Next Milestone
After domain review, choose the implementation stack based on the approved domain/application boundaries, then design persistence and application interfaces without yet implementing OTA integrations.
