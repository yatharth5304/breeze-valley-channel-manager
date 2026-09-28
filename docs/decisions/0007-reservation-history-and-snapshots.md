# ADR 0007 — Reservation History and Commercial Snapshots

## Status
Proposed

## Decision
Keep a minimal current Reservation state and a lightweight ReservationChange history rather than event sourcing.

Current status is NEW, CONFIRMED, or CANCELLED. Modifications update current facts and append a change record.

ReservationRoom preserves historical commercial facts needed to understand the booking, including booked room/rate identity, quantity, occupancy, nightly prices, taxes/fees, discounts, total, currency, meal/package inclusion, and cancellation-policy snapshot.

Current master data remains separate and can change without changing historical reservation meaning.

## Consequences
- Historical bookings remain understandable.
- Synchronization and auditability are supported.
- The design avoids the complexity of full event sourcing.
- Exact OTA version/sequence handling remains an integration concern.
