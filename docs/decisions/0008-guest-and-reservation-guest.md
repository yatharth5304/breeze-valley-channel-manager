# ADR 0008 — Guest and Reservation Guest Association

## Status
Proposed

## Decision
Use reusable Guest records plus a ReservationGuest association.

ReservationGuest supports a primary guest and additional guests while allowing the same Guest identity to appear on multiple reservations.

Keep the Guest model intentionally small for channel management. Do not build CRM/PMS guest functionality.

## Consequences
- Guest identity is not duplicated into every reservation.
- Reservation-specific role/context remains available.
- Guest deduplication/merge can remain conservative until business requirements justify stronger identity matching.
- Privacy and retention remain explicit concerns.
