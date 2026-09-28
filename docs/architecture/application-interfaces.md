# Application Interfaces

## Status

**PROPOSED** — application-level contracts only. These are logical interfaces, not implemented APIs.

The application layer owns business operations. Infrastructure implements persistence and integration ports. OTA-specific request/response models remain inside adapters.

## Reservation Commands

### ReservationService

Conceptual operations:

- createReservation(command)
- confirmReservation(reservationId)
- modifyReservation(reservationId, command)
- cancelReservation(reservationId, reason/source)
- getReservation(reservationId)

Commands contain domain-level concepts such as property, source, guest references, room/rate lines, stay dates, occupancy, and commercial facts.

They do not accept OTA-specific payload objects.

### Reservation result

Returns the local reservation identity and current state plus the relevant domain representation.

## Inventory

### InventoryAvailability

- getAvailability(propertyId, roomTypeId, stayDate)
- getAvailabilityRange(propertyId, roomTypeId, dateRange)

Returns calculated sellable availability and the relevant business restrictions.

### InventoryAllocation

- reserve(inventoryRequest)
- release(inventoryRequest)
- changeAllocation(oldRequest, newRequest)

These operations are designed to execute inside the same business transaction as the Reservation change.

They must enforce the inventory invariant and be safe under concurrent requests.

## Rate

### RateService

- getRate(ratePlanId, stayDate, occupancy)
- getRates(ratePlanId, dateRange, occupancy)
- getRestrictions(ratePlanId, dateRange)

The contract returns normalized internal rate/restriction concepts, never OTA-specific structures.

## Mapping

### ChannelMappingService

- getRoomMapping(channelPropertyId, roomTypeId)
- getRateMapping(channelPropertyId, ratePlanId)
- validateMappings(channelPropertyId, targets)
- resolveInternalRoomType(channelPropertyId, externalRoomId)
- resolveInternalRatePlan(channelPropertyId, externalRateId)

External identifiers remain integration-domain values.

## Inbound OTA Reservation Processing

### ExternalReservationProcessor

Conceptual input:

- channel
- channel property
- external event identity when available
- normalized external reservation command
- external version/sequence when available

Responsibilities:
1. establish event-level idempotency;
2. resolve Channel + external reservation identity;
3. determine whether the delivery is new, duplicate, newer, or ambiguous;
4. invoke ReservationService;
5. coordinate inventory effects;
6. create resulting OutboxEvent(s);
7. record processing outcome.

The input contract is an integration boundary. OTA adapters convert their payloads into it.

## Webhook/Event Processing

### WebhookProcessor

- acceptInboundEvent(eventEnvelope)
- markProcessing(eventId)
- markProcessed(eventId)
- markFailed(eventId, error)

The processor must distinguish duplicate delivery from business processing failure.

A webhook transport handler must not contain reservation business logic.

## Outbound Synchronization

### InventorySyncService

- publishInventory(target)
- publishInventoryForChange(change)
- reconcileInventory(target)

### RateSyncService

- publishRates(target)
- publishRestrictions(target)
- reconcileRates(target)

These operate on internal domain values plus mapping/configuration. OTA-specific request construction belongs to the adapter.

## OTA Adapter Port

### ChannelAdapter

Conceptual capability methods:

- receive/normalize reservations
- publish inventory
- publish rates/restrictions
- acknowledge/process channel events where required
- reconcile external state where supported

Not every channel must implement every capability. The adapter contract should be capability-based rather than forcing one universal OTA API.

The adapter returns normalized integration results/errors, not core persistence mutations.

## Idempotency

### IdempotencyService

Conceptual operations:

- checkOrCreateExternalEvent(identity)
- resolveExternalReservation(channelId, externalReservationId)
- recordProcessingOutcome(identity, outcome)

The persistence implementation must enforce uniqueness; the application contract should not rely on an in-memory check.

## Transaction Boundary

### TransactionManager

Conceptual operation:

- execute(work)

Application services use this abstraction when one business operation must atomically update multiple repositories and create outbox intent.

Infrastructure supplies the actual transaction implementation.

No application service should open an independent transaction for one step and assume another service can roll it back.

## Outbox

### OutboxService

- append(event)
- claimPending(limit)
- markDispatched(eventId)
- reschedule(eventId, nextAttemptAt, reason)
- markFailed(eventId, reason)

Outbox append occurs inside the same transaction as the business change.

Outbound network calls occur after durable work has been committed.

## Persistence Ports

Repositories should be defined around aggregate/business needs rather than generic CRUD.

Examples:
- PropertyRepository
- RoomTypeRepository
- RoomRepository
- RatePlanRepository
- RateValueRepository
- RateRestrictionRepository
- InventoryRepository
- ReservationRepository
- GuestRepository
- PaymentRepository
- ChannelRepository
- ChannelPropertyRepository
- MappingRepository
- ReservationChangeRepository
- SyncJobRepository
- SyncAttemptRepository
- WebhookEventRepository
- OutboxRepository

Repository interfaces must not leak ORM-specific types into the domain/application layer.

## Dependency Rules

Allowed:

API/entry point -> application contract -> domain -> infrastructure implementation

Integration:

OTA adapter -> normalized integration contract -> application service -> core domain

Prohibited:

OTA adapter -> repository/table -> direct mutation

Prohibited:

core domain -> OTA SDK/client

## Error Categories

Application contracts should distinguish at least:
- validation/business-rule failure
- not found
- duplicate/idempotent replay
- concurrency/conflict
- mapping invalid
- retryable infrastructure failure
- non-retryable integration failure

Exact exception/error representation is implementation-specific.

## Scope

No HTTP routes, DTO framework classes, controllers, ORM models, or OTA client code are defined by this document.
