# Domain Model — Initial Working Model

These entities are the initial concepts to investigate and refine. They are not all required in the first implementation, and exact aggregate ownership remains an implementation decision.

| Entity | Meaning | Boundary |
|---|---|---|
| Property | Hotel/business context | Core |
| RoomType | Sellable room category | Core |
| Room | Physical room/unit, if physical-level tracking is needed | Core / PMS-adjacent |
| RatePlan | Pricing and stay/cancellation/occupancy rules | Core |
| Inventory | Sellable quantity by room type and date | Core |
| Reservation | Booking aggregate and lifecycle | Core |
| ReservationRoom | Reservation line/allocation for room type, quantity, dates | Core |
| Guest | Guest identity/contact information associated with a reservation | Core |
| Payment | Payment information/status associated with a booking | Core/PMS-adjacent; scope to be refined |
| Channel | External distribution channel/OTA | Integration |
| ChannelProperty | Mapping/configuration between a property and a channel | Integration |
| ChannelRoomMapping | Internal room type ↔ external room mapping | Integration |
| ChannelRateMapping | Internal rate plan ↔ external rate mapping | Integration |
| SyncJob | A synchronization unit of work | Integration |
| SyncAttempt | An individual attempt and result for a sync job | Integration |
| WebhookEvent | Received external event envelope for idempotency/audit | Integration |
| OutboxEvent | Internal event waiting for reliable external delivery | Integration |

## Core Ownership
Core entities should model hotel business concepts without requiring an OTA-specific representation.

## Integration Ownership
Channel mappings, external identifiers, adapter payloads, sync attempts, and webhook envelopes belong at the integration boundary.

## Reservation Lifecycle
Candidate states include:

- NEW
- CONFIRMED
- MODIFIED
- CANCELLED
- NO_SHOW
- CHECKED_IN
- CHECKED_OUT

For the initial channel-manager scope, confirmed/new/modified/cancelled behavior is central. No-show, check-in, and check-out are primarily PMS/front-desk lifecycle concepts and should not be assumed to belong in the first channel-manager core without a concrete requirement.

## Questions to Resolve Before Implementation
- Is inventory tracked only at room-type level or also by physical room?
- Which rate dimensions are required initially?
- What payment information must the channel manager own versus merely receive/status-map?
- What reservation fields are mandatory across all target OTAs?
- Which lifecycle transitions are legal from each state?
- Which events require immediate inventory recalculation and outbound sync?
