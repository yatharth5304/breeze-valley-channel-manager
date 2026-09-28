# Persistence Design

## Status

**PROPOSED** — reviewable persistence mapping. No production schema or migration is defined here.

The persistence model follows the approved domain model. PostgreSQL is the recommended relational database; exact physical schema remains an implementation step after this document is approved.

## General Rules

- Every aggregate/entity has an internal stable primary identity.
- Foreign keys enforce domain relationships.
- Natural uniqueness is enforced where duplicate records would violate business meaning.
- Current master data is separated from historical reservation facts.
- External identifiers never replace internal identifiers.
- Timestamps should be explicit and consistently interpreted.
- Monetary amounts use exact decimal semantics, never binary floating-point.
- Date-only stay concepts remain date-only; timestamps are used for events and audit records.
- Soft deletion is not the default. Historical reservations and integration records remain auditable.

## Entity Mapping

| Entity | Primary identity | Important relationships | Important uniqueness / indexes |
|---|---|---|---|
| Property | property_id | owns RoomType, Room, RatePlan, Inventory, Reservation; connects ChannelProperty | unique property code |
| RoomType | room_type_id | Property -> RoomType; Room and RatePlan reference it | unique(property_id, code); index(property_id, status) |
| Room | room_id | belongs to Property + RoomType | unique(property_id, room_number/code); index(room_type_id, status) |
| RatePlan | rate_plan_id | belongs to Property + RoomType | unique(room_type_id, code); index(room_type_id, status) |
| RateValue | rate_value_id or deterministic identity | belongs to RatePlan | unique(rate_plan_id, stay_date, occupancy_key); index(rate_plan_id, stay_date) |
| RateRestriction | rate_restriction_id or deterministic identity | belongs to RatePlan | unique(rate_plan_id, stay_date); index(rate_plan_id, stay_date) |
| Inventory | inventory_id or deterministic identity | Property + RoomType + stay_date | unique(property_id, room_type_id, stay_date); index(room_type_id, stay_date) |
| Reservation | reservation_id | Property; Channel when external; ReservationRoom/Guest/Payment/Change | unique(channel_id, external_reservation_id) for external reservations; indexes on property/status/stay dates |
| ReservationRoom | reservation_room_id | Reservation; RoomType + RatePlan | index(reservation_id); index(room_type_id, check_in/check_out) |
| Guest | guest_id | associated through ReservationGuest | conservative searchable indexes; no forced global uniqueness on name/email |
| ReservationGuest | reservation_guest_id | Reservation + Guest | uniqueness for reservation + role as appropriate; index(reservation_id) |
| ReservationChange | reservation_change_id | Reservation | unique(reservation_id, sequence/version); index(reservation_id, occurred_at) |
| Payment | payment_id | Reservation | index(reservation_id, status); external reference uniqueness only within known provider/scope |
| Channel | channel_id | ChannelProperty | unique(channel code) |
| ChannelProperty | channel_property_id | Channel + Property | unique(channel_id, property_id); unique(channel_id, external_property_id) where channel scope permits |
| ChannelRoomMapping | channel_room_mapping_id | ChannelProperty + RoomType | unique(channel_property_id, room_type_id); unique(channel_property_id, external_room_id) |
| ChannelRateMapping | channel_rate_mapping_id | ChannelProperty + RatePlan | unique(channel_property_id, rate_plan_id); unique(channel_property_id, external_rate_id) |
| SyncJob | sync_job_id | ChannelProperty; optionally OutboxEvent/target | indexes(status, available_at), channel_property_id, dedupe key where applicable |
| SyncAttempt | sync_attempt_id | SyncJob | unique(sync_job_id, attempt_number); index(sync_job_id, started_at) |
| WebhookEvent | webhook_event_id | ChannelProperty | unique(channel_property_id, external_event_id) when supplied; index(status, received_at) |
| OutboxEvent | outbox_event_id | local aggregate/entity reference | index(status, available_at, created_at); unique dedupe key where event semantics require it |

Conditional uniqueness should be used for optional external IDs: direct/local reservations do not have an external reservation ID.

## Historical vs Current Data

### Current master data
Property, RoomType, Room, RatePlan, Channel, ChannelProperty, and mappings describe current configuration.

### Historical reservation facts
ReservationRoom preserves booked RoomType/RatePlan labels and commercial values needed to understand the booking at creation/modification time.

ReservationChange preserves the change sequence and relevant historical facts.

Changing a RatePlan or RoomType name must not rewrite historical reservation meaning.

## Reservation Persistence

Reservation stores current booking state and stable identity.

ReservationRoom stores booking room/rate lines and historical commercial facts. A reservation may have multiple lines.

The external reservation identity is unique at the Channel + external_reservation_id scope.

ReservationChange is append-oriented. It is not the source of truth for current reservation state; current Reservation/ReservationRoom state is.

Cancellation updates current state and records a cancellation change. The cancelled reservation remains persisted.

## Guest Persistence

Guest is reusable but not globally deduplicated by a guessed identity rule.

ReservationGuest supplies reservation-specific role, such as PRIMARY or ADDITIONAL.

Do not make email or name globally unique. A future explicit merge workflow can reconcile duplicate Guest records without changing the Reservation model.

## Inventory Persistence

The Inventory record is the coordination point for one Property + RoomType + stay date.

Persist:
- operational/baseline capacity input or an explicitly defined capacity snapshot
- manual blocked quantity
- reserved quantity
- override state/value if approved
- audit/version information

Available inventory is a derived business result.

Do not store independently editable available quantity as another source of truth.

Physical Room status remains in Room. Inventory does not duplicate individual room status.

### Reservation consumption

A reservation operation covering multiple nights identifies every affected stay-date Inventory row.

The transaction must:
1. determine affected RoomType/date rows;
2. acquire the required concurrency protection;
3. validate availability/business rules;
4. update reserved quantities;
5. update Reservation/ReservationRoom state;
6. append ReservationChange;
7. create OutboxEvent(s);
8. commit all changes together.

If any step fails, the entire business operation rolls back.

## Concurrent Reservation Attempts

The invariant is:

> Two concurrent reservation operations must not both consume the same last available inventory.

The persistence design therefore requires serialization around affected Inventory rows.

Possible PostgreSQL mechanisms, to be selected during implementation, include:
- row-level locking of Inventory rows;
- atomic conditional updates that only succeed when sufficient availability remains;
- SERIALIZABLE transactions with retry on serialization failure.

For multi-night/multi-room operations, affected Inventory rows must be locked/updated in a deterministic order to reduce deadlock risk.

No locking mechanism is implemented in this milestone.

## Inventory Override

**TO VERIFY:** absolute sellable quantity versus additive adjustment.

Until semantics are approved, persistence should reserve an explicit override concept without treating it as part of physical Room capacity.

If the override becomes a first-class record, it should retain property, room type, stay date, value/semantics, reason, actor, timestamps, and audit/version information.

## Rate Persistence

RatePlan stores stable package semantics.

RateValue stores date-specific amounts and occupancy context.

RateRestriction stores date-specific selling restrictions.

Avoid duplicating a complete RatePlan inside every date row. ReservationRoom snapshots only the commercial facts agreed for that booking.

## Integration Persistence

### Channel / mappings
External IDs are stored only in integration entities.

### WebhookEvent
Persist enough metadata to identify, deduplicate, process, retry, and audit an inbound event. Raw external payload retention is TO VERIFY because of privacy/retention requirements.

### OutboxEvent
Created in the same transaction as the local business change it represents.

### SyncJob / SyncAttempt
SyncJob represents durable work; SyncAttempt records execution attempts. A retry creates another SyncAttempt, not another business event.

## Transaction Boundaries

### Reservation + Inventory
One transaction.

### Reservation cancellation/modification + OutboxEvent
One transaction.

### Local rate/inventory change + OutboxEvent
One transaction.

### Webhook processing
Persist WebhookEvent before/within the processing boundary. Business application of a valid event and its resulting inventory/reservation/outbox changes should be one transaction after duplicate detection.

### Outbound OTA call
Never hold a database transaction open while waiting for an OTA network response.

The durable local transaction creates/claims work. The adapter call happens outside that transaction. Result state is recorded afterward.

## Indexing Principles

Indexes should serve:
- property and room-type lookups;
- stay-date availability queries;
- reservation retrieval by property/status/date;
- external reservation/event idempotency;
- mapping lookup by internal and external IDs;
- pending synchronization work;
- audit/history retrieval.

Do not create speculative indexes for every column.

## Migration Boundary

The eventual migration system is intentionally not selected/implemented in this milestone.

This document is a logical persistence design, not a production schema or migration plan.
