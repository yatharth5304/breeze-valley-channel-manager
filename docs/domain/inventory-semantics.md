# Inventory Semantics

## Status

**DECIDED:** RoomType + stay date is the sellable inventory grain.  
**DECIDED:** local inventory is authoritative.  
**PROPOSED:** the calculation model below.  
**TO VERIFY:** whether Breeze Valley wants overbooking, and if so, the permitted allowance.

## Definitions

For a Property, RoomType, and stay date:

- **Physical capacity:** count of Rooms in the RoomType that are not INACTIVE.
- **Out-of-order quantity:** active physical rooms currently OUT_OF_ORDER.
- **Out-of-service quantity:** active physical rooms currently OUT_OF_SERVICE.
- **Operational capacity:** physical capacity minus rooms unavailable because of OUT_OF_ORDER or OUT_OF_SERVICE status.
- **Manual block:** explicit quantity the hotel removes from sale for that date without changing Room master data.
- **Reserved quantity:** quantity consumed by active reservations for that RoomType and stay date.
- **Available inventory:** quantity currently offered for sale after applying capacity, blocks, reservations, and any explicitly approved overbooking allowance.
- **Inventory override:** explicit hotel instruction that changes the sellable result for a date. It is separate from physical-room status and must be auditable.

## Baseline Formula

Without overbooking:

available = max(0, operational_capacity - manual_block - reserved_quantity)

operational_capacity = physical_capacity - out_of_order_quantity - out_of_service_quantity

A manual adjustment must not silently change physical capacity.

## Example

RoomType: Deluxe  
Physical rooms: 10  
Date: 2026-10-15

- Physical capacity = 10
- OUT_OF_ORDER = 1
- OUT_OF_SERVICE = 0
- Operational capacity = 9
- Manual block = 1
- Reservations consuming inventory = 4

Therefore:

available = 9 - 1 - 4 = 4

**Available inventory = 4 rooms.**

A room that is both physically unavailable and manually blocked must not be deducted twice.

## Reservation Effects

### New confirmed reservation

If a reservation adds 2 Deluxe rooms:

reserved_quantity: 4 -> 6

available: 4 -> 2

Reservation and inventory effects are one business operation.

### Reservation modification

If the reservation changes from 2 Deluxe rooms to 1:

reserved_quantity: 6 -> 5

available: 2 -> 3

If it changes room type, old RoomType/date inventory is released and new RoomType/date inventory is consumed as one coordinated operation.

If dates change, affected nights are released/consumed accordingly.

### Cancellation

If the reservation consuming 1 Deluxe room is cancelled:

reserved_quantity: 5 -> 4

available: 3 -> 4

Cancellation must not create inventory beyond what the reservation had consumed.

## Manual Blocks

A manual block reduces sellable inventory without changing Room state.

Example:
- Operational capacity = 9
- Reserved = 4
- Manual block = 2
- Available = 3

Removing the block restores those 2 units, subject to reservations and other restrictions.

## Inventory Overrides

**PROPOSED:** Support an explicit date-level override as a separate concept from the normal calculation.

It should record property, room type, stay date, requested sellable quantity or adjustment semantics, reason/source, timestamps, and actor/audit information.

The exact override semantics — absolute available quantity versus additive adjustment — are **TO VERIFY**. The implementation should choose one explicit semantic rather than ambiguous arithmetic.

## Overbooking

**TO VERIFY — no business policy has been established.**

The domain therefore uses the non-overbooking calculation as the safe baseline. If overbooking is later approved, model it explicitly as an allowance or policy rather than allowing negative availability accidentally.

Example:
- Operational capacity = 9
- Manual block = 0
- Reserved = 9
- Baseline available = 0

Without an approved allowance, another reservation must not reduce availability below zero.

## Reservation Holds

Temporary reservation holds are **TO VERIFY**. They must not be conflated with confirmed reservations or manual blocks until their lifecycle is defined.

## Invariants

1. Available inventory must not become negative unless an explicitly approved overbooking policy permits it.
2. Physical Room status must not be mutated merely to achieve a desired sellable quantity.
3. A reservation consumes inventory only for its RoomType and stay dates.
4. Modification/cancellation adjusts exactly the inventory previously consumed.
5. Inventory and reservation changes are transactionally consistent.
6. Manual blocks/overrides are attributable and auditable.
