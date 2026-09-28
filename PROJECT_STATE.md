# Project State

## Current Phase
**Phase 0 — Architecture and Foundation**

## Product
Custom channel manager for Breeze Valley, a single hotel property in Panchgani/Mahabaleshwar, Maharashtra, India.

The implementation should not hard-code the property as a permanent domain constraint; future properties should be possible without redesigning the core model.

## Target Channels
- Agoda
- Booking.com
- MakeMyTrip
- Goibibo
- Airbnb

## Current Objective
Establish the internal architecture, domain boundaries, synchronization model, and development conventions before implementing OTA adapters.

## Status
- Repository initialized.
- Architecture and domain foundation documentation is being established.
- OTA integrations are intentionally not implemented yet.

## Working Principles
- Internal inventory representation is authoritative locally.
- OTA integrations are adapters around a common internal model.
- External reservation identities are idempotent.
- Inventory and reservation changes must remain transactionally consistent.
- Synchronization must tolerate retries and failures.
- Important operations should be auditable.
- Secrets never belong in Git.
- Prefer a simple architecture over premature infrastructure.

## Open Implementation Decisions
The concrete programming language, framework, database access technology, queue/worker mechanism, deployment model, and exact OTA API contracts are not locked yet. Do not invent these decisions without evidence or an explicit project decision.

## Next Milestone
Define the initial domain model and application boundaries, then choose implementation technologies based on the actual requirements.
