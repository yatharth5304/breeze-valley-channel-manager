# ADR 0003 — Reservation Identity and Idempotency

## Status
Proposed

## Context
OTAs may deliver the same reservation or event more than once. Duplicate delivery must not create duplicate local reservations.

## Decision
For externally sourced reservations, canonical external identity is Channel + external_reservation_id.

Where an OTA supplies an event identifier, WebhookEvent also deduplicates by that event identity within the relevant channel/property scope.

Reservation-level idempotency and event-level idempotency are complementary. Reprocessing the same delivery must be safe.

## Consequences
- External identifiers are never treated as internal reservation IDs.
- A uniqueness constraint is expected at the persistence layer once technology is selected.
- Reservation processing must be safe under retries and duplicate delivery.
