# Reservation Model

## Status

**DECIDED:** Reservation is the core booking aggregate.  
**DECIDED:** current status is separate from change history.  
**PROPOSED:** historical commercial snapshot described below.  
**TO VERIFY:** exact field-level requirements after target OTA contracts are reviewed.

## Purpose

Reservation represents a booking regardless of whether it originated from an OTA or a direct/local source.

It must remain understandable after current Property, RoomType, RatePlan, cancellation policy, or pricing definitions change.

## Current Reservation Identity

Internal reservation_id is the stable local identity.

For channel-originated bookings, channel_id + external_reservation_id is the canonical external identity and must be unique within the relevant integration scope.

Direct/local bookings may have no channel or external reservation ID.

## Current State

The persisted current state should remain minimal:
- NEW
- CONFIRMED
- CANCELLED

MODIFIED is not a permanent current status. A modification updates current reservation facts and creates an auditable history record.

NO_SHOW, CHECKED_IN, and CHECKED_OUT remain PMS-adjacent and are not required by the initial Channel Manager.

## Reservation History

**PROPOSED:** Maintain a lightweight ReservationChange/history concept rather than event sourcing.

A change record should capture:
- reservation_id
- change sequence/version
- change type: CREATED, MODIFIED, CANCELLED
- occurred_at
- source
- external event/reference when applicable
- changed commercial/booking facts or a safe snapshot reference
- processing/audit metadata

The purpose is synchronization, auditability, and reconciliation, not replaying the entire system from events.

## Commercial Snapshot

The reservation must preserve the commercial facts agreed at booking time.

For each ReservationRoom, preserve at minimum:
- booked RoomType identity and name/code snapshot
- booked RatePlan identity and name/code snapshot
- quantity
- adults
- children
- check-in/check-out
- nightly price by applicable night
- currency
- taxes/fees required to explain the booked total
- discounts applied
- total booked amount
- meal/package inclusion
- cancellation-policy snapshot
- source/channel
- external reservation ID when applicable

Current RoomType and RatePlan remain references to master data for current operations. Their mutable definitions must not be relied upon to reconstruct historical booking terms.

## Avoiding Duplication

Do not copy entire current RoomType or RatePlan master records into every reservation.

Instead:
- retain references to current internal entities where useful;
- store historical commercial facts needed to understand the booking;
- snapshot values that can change over time.

This is a historical value representation, not a second master-data system.

## Reservation Modification

A modification changes the current reservation and creates a new history/change record.

The application must compare old and new booking facts before applying inventory effects.

Examples:

Quantity change:
2 Deluxe -> 3 Deluxe:
- retain existing 2
- consume 1 additional Deluxe unit for affected nights

Date change:
10 Oct–12 Oct -> 11 Oct–13 Oct:
- release 10 Oct
- retain 11 Oct
- consume 12 Oct
- consume 13 Oct

Room-type change:
Deluxe -> Suite:
- release Deluxe inventory
- consume Suite inventory
- update ReservationRoom
- preserve the modification in history

The business result must be atomic.

## Cancellation

Cancellation changes current status to CANCELLED and releases exactly the inventory consumed by the reservation.

The historical reservation remains readable, including original commercial terms and cancellation-policy snapshot.

A cancelled reservation must not be deleted.

## Duplicate Delivery

Duplicate external delivery is resolved using external reservation/event identity before creating a new business effect.

A duplicate confirmation must be idempotent.

A repeated modification must not apply the same inventory delta twice.

A repeated cancellation must not release inventory twice.

## Late Modification

**PROPOSED:** A late modification is accepted only if it can be safely reconciled against current reservation version/state.

If an incoming change is older than the latest known change, it must not blindly overwrite newer state. Compare external sequencing when available; otherwise route ambiguity to reconciliation/manual handling rather than guessing.

Exact OTA sequencing guarantees are **TO VERIFY**.

## Cancellation After Modification

Cancellation applies to the reservation's current consumed inventory, regardless of prior modifications.

Example:
- original: 1 Deluxe, 10–12 Oct
- modified: 2 Deluxe, 11–13 Oct
- cancelled

The final cancellation releases the 2 Deluxe units for the final reservation nights. History retains the original booking, modification, and cancellation.

## Payment Information

Reservation references Payment records for limited payment state required by channel management. Payment is not an accounting ledger.

Never store raw card number, CVV/CVC, PIN, or equivalent authentication data.

## Open Questions

- **TO VERIFY:** exact snapshot field list required by each target OTA.
- **TO VERIFY:** external modification/version sequencing available from each channel.
- **TO VERIFY:** treatment of reservations received already cancelled.
- **TO VERIFY:** exact timezone/date semantics for stay nights and channel timestamps.
- **TO VERIFY:** payment/refund information required for each channel.
