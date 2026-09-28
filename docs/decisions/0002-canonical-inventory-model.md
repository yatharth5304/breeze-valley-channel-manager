# ADR 0002 — Canonical Inventory Model

## Status
Proposed

## Context
The channel manager must answer sellable availability by room type and date while accounting for physical capacity, reservations, and hotel-controlled blocks.

## Decision
Use RoomType as the canonical sellable inventory unit. Inventory is represented per Property + RoomType + stay date.

Physical Room records provide capacity/status information but are not themselves the OTA sellable inventory unit.

Inventory should distinguish baseline/sellable capacity, reserved quantity, blocked quantity, and explicit manual adjustments rather than assuming a single permanently derived number.

Available inventory is a calculated business result. The exact handling of overbooking buffers, absolute overrides, and reservation holds remains TO VERIFY.

## Consequences
- OTA mappings target RoomType.
- Physical-room assignment is not required for normal channel reservations.
- Reservation and inventory changes must be coordinated transactionally.
- Hotel-specific inventory controls can be represented without corrupting room master data.
