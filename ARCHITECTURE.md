# Architecture

## Conceptual Model

```
CHANNEL MANAGER
  Reservation Engine
  Inventory Engine
  Rates Engine
  Core Domain
  Relational Database
  Sync / Event Layer
  OTA Adapters
```

This is a conceptual architecture, not a commitment to a particular framework or deployment topology.

## Core Boundary
The core domain owns internal concepts such as properties, room types, inventory, rates, reservations, guests, and their business rules.

OTA adapters translate between external channel contracts and the internal model. OTA-specific field names, authentication, request formats, response parsing, and retry behavior belong at the integration boundary.

## Synchronization

### OTA to Core
```
OTA event/API
  -> OTA adapter
  -> normalized external event/reservation
  -> idempotency check
  -> reservation service
  -> inventory update
  -> internal sync/event records
```

### Core to OTA
```
Core inventory/rate/reservation change
  -> internal event/outbox representation
  -> sync worker/process
  -> OTA adapter
  -> OTA API
  -> sync attempt/audit result
```

## Inventory Authority
The channel manager maintains the authoritative local representation of sellable inventory.

Example:
- Total inventory: 10
- Reserved: 4
- Out of order / unavailable: 0
- Available: 6

OTAs are external distribution channels. They are not peer databases for the internal domain.

## Idempotency
An external reservation should be uniquely identified by at least:

`(channel, external_reservation_id)`

Receiving the same reservation or webhook more than once must not create duplicate local reservations.

## Failure Model
The design must account for:
- API unavailability
- timeouts and network failures
- duplicate reservations/events
- partial synchronization
- invalid channel mappings
- authentication failures
- rate limiting
- reservation modifications and cancellations
- retry and backoff
- reconciliation of uncertain outcomes

## Simplicity Constraint
Start as a simple application with clear module boundaries. Introduce queues, separate services, or infrastructure components only when a concrete reliability, scale, or operational requirement justifies them.
