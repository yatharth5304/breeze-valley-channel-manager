# Canonical Domain Model

## Status
**PROPOSED** — domain shape for implementation review. Technology, persistence schema, and OTA contracts remain unspecified.

## Core Domain
Property, RoomType, Room, RatePlan, RateValue, RateRestriction, Inventory, Reservation, ReservationRoom, Guest, ReservationGuest, Payment, ReservationChange.

## Integration Domain
Channel, ChannelProperty, ChannelRoomMapping, ChannelRateMapping, SyncJob, SyncAttempt, WebhookEvent, OutboxEvent.

Core entities must not depend on OTA-specific payloads, IDs, or adapter implementations.

## Property
Generic hotel/property context. Breeze Valley is the initial configured property but is not hard-coded.

Important concepts: property_id, name, code, timezone, currency, address, status, timestamps.

## RoomType
Sellable room category such as Deluxe or Suite.

Property 1-to-many RoomType.

## Room
Physical room such as 101 or 102.

RoomType 1-to-many Room.

Statuses:
- ACTIVE
- OUT_OF_ORDER
- OUT_OF_SERVICE
- INACTIVE

OUT_OF_ORDER and OUT_OF_SERVICE both remove physical capacity; they are distinct operational states. INACTIVE removes the room from the active property inventory model.

## RatePlan
Stable commercial package/rule attached to one RoomType.

Contains package identity, meal inclusion, cancellation-policy definition, occupancy rules, and status.

One RoomType can have many RatePlans.

RatePlan is not a physical room and does not contain one permanent date-independent price.

## RateValue
Date-specific commercial price associated with a RatePlan.

Minimum concepts: rate_plan_id, stay_date, occupancy context, amount, currency, timestamps/version.

## RateRestriction
Date-specific selling restrictions associated with a RatePlan.

Minimum concepts: rate_plan_id, stay_date/date scope, MinLOS, MaxLOS, closed_for_sale, CTA, CTD, advance-booking limits.

Restrictions are explicit and are not encoded through price or inventory values.

## Inventory
Sellable quantity for Property + RoomType + stay date.

Inventory semantics are defined in docs/domain/inventory-semantics.md.

Core distinction:
- physical capacity comes from physical Room state;
- manual blocks/overrides are sellability controls;
- reserved quantity comes from active reservations;
- available quantity is a calculated business result.

## Reservation
Booking aggregate for OTA and direct/local bookings.

Current status:
- NEW
- CONFIRMED
- CANCELLED

MODIFIED is represented through ReservationChange/history rather than a permanent current state.

External identity for OTA bookings is Channel + external_reservation_id.

## ReservationRoom
Reservation line containing booked RoomType, RatePlan, quantity, stay dates, occupancy, and historical commercial facts.

It does not require a physical Room assignment.

## Guest
Reusable guest identity/contact record.

Guest is intentionally smaller than a CRM/PMS guest profile.

## ReservationGuest
Association between Reservation and Guest.

It supports primary guest, additional guests, and a guest appearing across multiple reservations.

It is preferred over embedding one Guest directly into Reservation because guest identity is reusable while reservation-specific role/context remains local to the association.

## ReservationChange
Lightweight reservation history record.

Captures creation/modification/cancellation history, sequence/version, source, timestamps, external references, and relevant changed facts.

It is an audit/synchronization aid, not event sourcing.

## Payment
Minimal reservation payment state required for channel management.

Contains amount, currency, status, non-sensitive payment method/type, and external payment/reference information where required.

Never store raw card numbers, CVV/CVC, PINs, or equivalent authentication secrets.

## Integration Domain
Channel and ChannelProperty model the external distribution connection. ChannelRoomMapping and ChannelRateMapping translate internal RoomType/RatePlan identifiers to external identifiers.

WebhookEvent provides inbound durability/idempotency. OutboxEvent provides reliable outbound intent. SyncJob and SyncAttempt provide retryable, auditable synchronization work.

## Core Relationships

Property
  -> RoomType
      -> Room
      -> RatePlan
          -> RateValue
          -> RateRestriction
  -> Inventory
  -> Reservation
      -> ReservationRoom
      -> ReservationGuest -> Guest
      -> Payment
      -> ReservationChange

Channel
  -> ChannelProperty -> Property
      -> ChannelRoomMapping -> RoomType
      -> ChannelRateMapping -> RatePlan

Core transaction -> OutboxEvent -> SyncJob -> SyncAttempt -> OTA adapter
OTA -> WebhookEvent -> application service

## Open Questions
- TO VERIFY: exact occupancy dimensions and child-age rules.
- TO VERIFY: exact cancellation-policy structure.
- TO VERIFY: exact OTA restriction capabilities and sequencing.
- TO VERIFY: inventory override/overbooking policy.
- TO VERIFY: reservation timezone/date semantics per channel.
- TO VERIFY: guest retention and privacy requirements.
- TO VERIFY: payment/refund fields required per channel.
