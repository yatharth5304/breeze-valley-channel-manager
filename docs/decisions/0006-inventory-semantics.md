# ADR 0006 — Inventory Semantics

## Status
Proposed, pending business confirmation of overbooking/override policy.

## Decision
Use Property + RoomType + stay date as the inventory grain.

Physical capacity is based on active physical Rooms. OUT_OF_ORDER and OUT_OF_SERVICE rooms are excluded from operational capacity. Manual blocks reduce sellable quantity without changing Room master data.

Without approved overbooking:

available = max(0, operational_capacity - manual_block - reserved_quantity)

Inventory overrides are explicit and auditable rather than hidden inside physical room state.

Overbooking is not enabled by assumption. Any allowance must be an explicit business policy.

## Consequences
Reservation creation, modification, and cancellation must update inventory atomically. Inventory remains authoritative locally while OTA availability is derived from it.
