# Entity Relationship Diagram

## Conceptual ERD

Mermaid ER relationships:

PROPERTY ||--o{ ROOM_TYPE : contains
ROOM_TYPE ||--o{ ROOM : has
ROOM_TYPE ||--o{ RATE_PLAN : offers
RATE_PLAN ||--o{ RATE_VALUE : priced_by_date
RATE_PLAN ||--o{ RATE_RESTRICTION : restricted_by_date

PROPERTY ||--o{ INVENTORY : controls
ROOM_TYPE ||--o{ INVENTORY : has

PROPERTY ||--o{ RESERVATION : receives
RESERVATION ||--|{ RESERVATION_ROOM : contains
RESERVATION ||--o{ RESERVATION_GUEST : has
GUEST ||--o{ RESERVATION_GUEST : participates
ROOM_TYPE ||--o{ RESERVATION_ROOM : booked_as
RATE_PLAN ||--o{ RESERVATION_ROOM : priced_by
RESERVATION ||--o{ PAYMENT : has
RESERVATION ||--o{ RESERVATION_CHANGE : records

CHANNEL ||--o{ CHANNEL_PROPERTY : connects
PROPERTY ||--o{ CHANNEL_PROPERTY : distributes_through
CHANNEL_PROPERTY ||--o{ CHANNEL_ROOM_MAPPING : maps
ROOM_TYPE ||--o{ CHANNEL_ROOM_MAPPING : maps_to
CHANNEL_PROPERTY ||--o{ CHANNEL_RATE_MAPPING : maps
RATE_PLAN ||--o{ CHANNEL_RATE_MAPPING : maps_to

SYNC_JOB ||--o{ SYNC_ATTEMPT : attempts
CHANNEL_PROPERTY ||--o{ SYNC_JOB : executes_for
CHANNEL_PROPERTY ||--o{ WEBHOOK_EVENT : receives
PROPERTY ||--o{ OUTBOX_EVENT : emits

## Interpretation

Inventory is identified by Property + RoomType + stay date.

RatePlan is the stable commercial definition. RateValue and RateRestriction are date-specific representations associated with RatePlan.

ReservationRoom references RoomType and RatePlan and stores historical booking facts. It does not require physical Room assignment.

ReservationGuest associates reusable Guest records with a Reservation and identifies the primary guest without forcing guest data into Reservation itself.

ReservationChange is lightweight history for synchronization/auditability. It is not event sourcing.

ChannelRoomMapping maps the sellable RoomType; ChannelRateMapping maps RatePlan.

OutboxEvent represents committed outbound intent. SyncJob and SyncAttempt are integration operational records. WebhookEvent is the inbound durability/idempotency boundary.

## Simplified Flow

Property -> RoomType -> Room / RatePlan
RatePlan -> RateValue / RateRestriction
Property -> Inventory(RoomType + date)
Property -> Reservation -> ReservationRoom / ReservationGuest / Payment / ReservationChange

Property -> ChannelProperty -> Channel
ChannelProperty -> ChannelRoomMapping -> RoomType
ChannelProperty -> ChannelRateMapping -> RatePlan

Core transaction -> OutboxEvent -> SyncJob -> SyncAttempt -> OTA Adapter
OTA -> Webhook/API -> WebhookEvent -> Application Service
