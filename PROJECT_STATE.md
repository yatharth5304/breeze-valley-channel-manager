# Project State

## Current Phase
**Phase 0 — Domain Model Resolution**

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

### DECIDED / EXISTING ARCHITECTURAL CONSTRAINTS
- Core hotel domain is independent from OTA-specific code.
- OTA integrations are isolated behind adapters.
- Local inventory is authoritative.
- External reservation identity must be idempotent.
- Reservation/inventory effects require transactional consistency.
- Synchronization must be retryable and auditable.
- Architecture remains a simple modular application rather than premature microservices/infrastructure.

### DECIDED / DOMAIN MODEL
- RoomType is the canonical sellable inventory unit.
- Room represents physical rooms and has ACTIVE, OUT_OF_ORDER, OUT_OF_SERVICE, and INACTIVE status.
- RatePlan is a stable commercial definition attached to one RoomType.
- Inventory is modeled per Property + RoomType + stay date.
- Reservation current status is NEW, CONFIRMED, or CANCELLED.
- MODIFIED is represented through reservation history rather than a permanent current status.
- External reservation identity is Channel + external_reservation_id.
- OTA event idempotency is separate from reservation idempotency.
- Guest identity is reusable; ReservationGuest associates guests to reservations and supports primary/additional roles.
- Payment is limited to reservation/channel-management information; raw card data and authentication secrets are excluded.
- Date-specific commercial values and restrictions are separated from RatePlan.
- RateValue represents date/occupancy-specific pricing.
- RateRestriction represents date-specific selling controls.
- PMS states NO_SHOW, CHECKED_IN, CHECKED_OUT remain outside the initial Channel Manager core.

### PROPOSED / DOMAIN SEMANTICS
- Inventory available quantity uses operational physical capacity minus manual blocks and reserved quantity, bounded at zero unless an approved overbooking policy exists.
- Inventory overrides are explicit and auditable.
- ReservationChange provides lightweight history rather than event sourcing.
- ReservationRoom stores historical commercial booking facts required to understand the reservation.
- Exact persistence representation remains open until technology selection.

## Remaining TO VERIFY

### Business policy
- Whether overbooking is permitted and the allowance/policy.
- Exact inventory override semantics: absolute sellable quantity versus additive adjustment.
- Whether temporary reservation holds are required.

### Commercial model
- Exact occupancy dimensions and child-age pricing.
- Exact cancellation-policy structure.
- Whether advance-booking restrictions are required initially.
- Exact MinLOS/MaxLOS/CTA/CTD capabilities required across target channels.

### Reservation/integration
- External modification/version sequencing available from each OTA.
- Treatment of reservations received already cancelled.
- Exact timezone/date semantics for stay nights and channel timestamps.
- Required payment/refund fields per channel.
- Guest retention, privacy, and deduplication policy.
- Exact OTA adapter capability contracts after actual channel documentation is reviewed.
- Persistence uniqueness/transaction implementation after technology selection.

## Conflicts Discovered
No conflict with the established architecture was found.

One refinement was required: the earlier model treated Guest as directly referenced by Reservation and left historical reservation facts unresolved. This milestone introduces ReservationGuest and lightweight ReservationChange, which preserve the existing architecture without adding event sourcing or OTA-specific dependencies.

## Current Objective
Complete domain clarification before persistence and technology selection.

## Next Milestone
Human review/approval of the resolved domain semantics. After approval, select the implementation technology stack and design persistence/application interfaces. Do not implement OTA integrations until those foundations are approved.
