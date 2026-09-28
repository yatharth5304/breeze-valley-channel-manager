# Canonical Domain Model

## Status
**PROPOSED** — domain shape for implementation review. Technology, persistence schema, and OTA contracts remain unspecified.

## Boundary

### Core domain
Property, RoomType, Room, RatePlan, Inventory, Reservation, ReservationRoom, Guest, Payment.

### Integration domain
Channel, ChannelProperty, ChannelRoomMapping, ChannelRateMapping, SyncJob, SyncAttempt, WebhookEvent, OutboxEvent.

The core domain must not depend on Agoda, MakeMyTrip, Goibibo, Booking.com, Airbnb, or any channel payload/model. Adapters translate external representations at the integration boundary.

## Property
Generic hotel/property context. The first configured property is Breeze Valley, but the entity is not property-specific.

Important fields: property_id, name, code, timezone, currency, structured address, status, created_at, updated_at.

Responsibilities: own the property's room types, rooms, rate plans, inventory, and reservations.

It must not contain OTA credentials, OTA room IDs, OTA rate IDs, or channel-specific configuration.

Proposed status: ACTIVE, INACTIVE.

## RoomType
Represents what is sold, such as Deluxe or Suite.

Important fields: room_type_id, property_id, name, code, description, max_occupancy, status.

Relationship: Property 1-to-many RoomType; RoomType 1-to-many Room.

## Room
Represents a physical room, such as 101, 102, 103.

Important fields: room_id, property_id, room_type_id, room_number/internal name, status, timestamps.

Proposed status values:
- ACTIVE — participates in normal hotel operations.
- OUT_OF_ORDER — temporarily unavailable and excluded from sellable capacity.
- OUT_OF_SERVICE — unavailable for a longer-term/service reason and excluded from sellable capacity.
- INACTIVE — retired/decommissioned or not part of current inventory.

These statuses are separated because temporary operational unavailability is different from retirement.

A channel manager may need Room for capacity derivation and reconciliation, but this does not make it a full PMS room-management system.

## RatePlan
A sellable pricing/package rule associated with a RoomType. Example: Deluxe -> Room Only; Deluxe -> Breakfast Included.

Important fields: rate_plan_id, property_id, room_type_id, name, code, meal_inclusion, cancellation_policy, occupancy_rules, status.

One RoomType can have many RatePlans. A RatePlan belongs to one RoomType in the initial model. A RatePlan is not inventory or a physical room.

Pricing for a date/occupancy belongs to the rate domain associated with the RatePlan. It must not be duplicated as the mutable current definition inside Reservation.

Cancellation policy, meal inclusion, and occupancy rules belong to RatePlan. Stay restrictions such as minimum length of stay, closed-to-arrival/departure, and stop-sell controls belong to the rate/availability domain; exact representation is TO VERIFY.

## Inventory
Answers: how many units of a RoomType can be sold for a given stay date.

Natural identity: property_id + room_type_id + stay_date.

Important concepts: inventory_id, property_id, room_type_id, stay_date, capacity/baseline, blocked_quantity, manual_adjustment, reserved_quantity, available_quantity, timestamps/version.

Local inventory is authoritative for the channel manager. OTAs receive availability; they are not authoritative for local availability.

Conceptual calculation:
available = sellable_capacity - reserved_quantity - blocked_quantity + applicable_manual_adjustments

This is not a final formula. It must avoid double-counting rooms excluded through Room status and support hotel-specific controls. Overbooking buffers, absolute overrides, and reservation holds are TO VERIFY.

Active Rooms can provide baseline capacity for a RoomType. Inventory remains independently controllable at RoomType/date level so the hotel can block sellable inventory without changing physical-room master data.

Reservation creation/modification/cancellation and inventory effects must be handled atomically.

## Reservation
Booking aggregate supporting OTA and direct/local bookings.

Important concepts: reservation_id, property_id, source_type, channel_id when externally sourced, external_reservation_id when externally sourced, status, created_at, updated_at, check_in, check_out, guest references, reservation-room lines, payment/financial references.

Relationships:
- belongs to one Property
- references Guest records
- owns one or more ReservationRoom lines
- may have zero or more Payment records
- may have external identity for channel-originated bookings

Do not copy mutable Guest, RoomType, RatePlan, or Payment master data wholesale into Reservation. Booking-time facts that must remain historically correct, such as booked price or occupancy, may require immutable snapshots/value objects; exact shape is TO VERIFY.

Direct bookings may have no channel_id or external_reservation_id.

## ReservationRoom
Line item within Reservation describing the room category, stay, and booked rate facts.

Important concepts: reservation_room_id, reservation_id, room_type_id, rate_plan_id, quantity, check_in, check_out, booked occupancy, booked price/amount, currency, applicable taxes/fees.

Physical Room assignment is optional and normally PMS/front-desk territory.

## Guest
Guest identity/contact data associated with reservations.

Important concepts: guest_id, name, required contact details, country/nationality where legitimately required, timestamps.

Store only data required for hotel/channel operations. Retention, privacy, and identity-verification fields are TO VERIFY.

## Payment
Minimal reservation payment state, not a full accounting or payment-processing ledger.

Important concepts: payment_id, reservation_id, status, amount, currency, non-sensitive payment method/type, external payment/reference identifier where needed, timestamps.

Do NOT store raw card numbers, CVV/CVC, PINs, full magnetic-stripe data, or equivalent authentication secrets.

Out of scope for now: accounting ledger, detailed settlement, invoicing, gateway processing, and full refund workflows unless explicitly added later.

# Integration Domain

## Channel
Generic external distribution channel.

Important concepts: channel_id, stable internal code such as AGODA, display name, adapter identifier, status.

No property-specific credentials or mappings belong here.

## ChannelProperty
Connects a Channel to a Property and stores channel-specific property configuration.

Important concepts: channel_property_id, channel_id, property_id, external property/hotel identifier, connection status, references to credential/configuration, sync settings.

Relationship: Channel 1-to-many ChannelProperty; Property 1-to-many ChannelProperty.

## ChannelRoomMapping
Maps an internal RoomType to an external channel room/listing identifier.

Important concepts: channel_room_mapping_id, channel_property_id, room_type_id, external room identifier, mapping status, timestamps.

Internal and external IDs are distinct. Never assume equality.

Mapping is at RoomType level initially because RoomType is the sellable inventory unit.

## ChannelRateMapping
Maps an internal RatePlan to an external channel rate-plan identifier.

Important concepts: channel_rate_mapping_id, channel_property_id, rate_plan_id, external rate-plan identifier, mapping status, timestamps.

Internal and external IDs are distinct. Mapping validity must be checked before synchronization.

## WebhookEvent
Durable record of an incoming external event before/while it is processed.

Important concepts: webhook_event_id, channel_property_id, external event identifier when supplied, event type, received timestamp, processing status, payload hash/raw-payload policy, error metadata.

Proposed lifecycle: RECEIVED -> PROCESSING -> PROCESSED or FAILED.

Duplicate detection happens before business effects. Event-level deduplication complements reservation-level idempotency.

## OutboxEvent
Reliable handoff from a committed local business change to outbound synchronization.

Important concepts: outbox_event_id, aggregate/entity reference, event type, event payload/version, created timestamp, processing state, retry/available time, deduplication key where needed.

The event should be created in the same transactional boundary as the business change it represents.

## SyncJob
A unit of outbound or reconciliation synchronization work for a channel/property and target resource/event.

Important concepts: sync_job_id, channel_property_id, job type, target/reference, status, priority if needed, scheduled/available time, completion/error metadata.

Conceptual lifecycle: PENDING -> RUNNING -> SUCCEEDED or FAILED, with retryable failures retaining job identity.

## SyncAttempt
One execution attempt for a SyncJob.

Important concepts: sync_attempt_id, sync_job_id, attempt number, started/completed timestamps, outcome, external response/reference where safe, error category/code, diagnostic metadata.

SyncJob represents work; SyncAttempt records each execution so retries are observable.

# Reservation Lifecycle

Channel-manager booking state should be distinguished from PMS operational stay state.

Core states:
- NEW — booking received/created but not yet in its normal confirmed state.
- CONFIRMED — active and accepted.
- CANCELLED — cancelled and no longer consumes inventory.

MODIFIED should be treated primarily as an auditable change/version event rather than a permanent status. This avoids a reservation remaining forever in MODIFIED.

PMS-adjacent states:
- NO_SHOW
- CHECKED_IN
- CHECKED_OUT

These may be useful later but are not required by the first channel-manager core. If introduced, their ownership and synchronization behavior must be explicitly defined.

# External Reservation Identity and Idempotency

For an OTA-originated reservation, canonical external identity is:
channel_id + external_reservation_id

This pair must be unique within the relevant integration scope. Repeated delivery must resolve to the existing reservation path rather than create a duplicate.

If an OTA supplies an event ID, that event ID is also deduplicated at WebhookEvent. Event-level and reservation-level idempotency are complementary.

# Entity Relationships

Property
  -> RoomType
      -> Room
      -> RatePlan
  -> Inventory (RoomType + stay date)
  -> Reservation
      -> ReservationRoom -> RoomType / RatePlan
      -> Guest
      -> Payment

Channel
  -> ChannelProperty -> Property
      -> ChannelRoomMapping -> RoomType
      -> ChannelRateMapping -> RatePlan

WebhookEvent -> inbound processing -> Reservation/Inventory application services

Core transaction -> OutboxEvent -> SyncJob -> SyncAttempt(s) -> OTA adapter

Cardinality:
- Property 1-to-many RoomType
- RoomType 1-to-many Room
- RoomType 1-to-many RatePlan
- Property 1-to-many Inventory records
- Property 1-to-many Reservation
- Reservation 1-to-many ReservationRoom
- Reservation 1-to-many Payment (or zero)
- Channel 1-to-many ChannelProperty
- Property 1-to-many ChannelProperty
- ChannelProperty 1-to-many ChannelRoomMapping
- ChannelProperty 1-to-many ChannelRateMapping
- SyncJob 1-to-many SyncAttempt

## Non-Goals
This model does not define a full PMS, housekeeping system, accounting ledger, payment-card vault, CRM, or OTA-specific domain model.

## Open Questions
- TO VERIFY: exact inventory override and overbooking semantics.
- TO VERIFY: whether Room-level availability is needed beyond capacity derivation and status.
- TO VERIFY: reservation snapshot/version strategy for booked rates, taxes, and occupancy.
- TO VERIFY: guest merge/deduplication policy.
- TO VERIFY: exact cancellation-policy structure and rate restriction vocabulary.
- TO VERIFY: required payment statuses and OTA payment semantics.
- TO VERIFY: event payload retention/privacy requirements.
