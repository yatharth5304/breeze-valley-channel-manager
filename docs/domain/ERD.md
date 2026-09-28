# Entity Relationship Diagram

## Conceptual ERD

    PROPERTY ||--o{ ROOM_TYPE : contains
    ROOM_TYPE ||--o{ ROOM : has
    ROOM_TYPE ||--o{ RATE_PLAN : offers
    PROPERTY ||--o{ INVENTORY : controls
    ROOM_TYPE ||--o{ INVENTORY : has
    PROPERTY ||--o{ RESERVATION : receives
    RESERVATION ||--|{ RESERVATION_ROOM : contains
    ROOM_TYPE ||--o{ RESERVATION_ROOM : booked_as
    RATE_PLAN ||--o{ RESERVATION_ROOM : priced_by
    RESERVATION ||--o{ GUEST : references
    RESERVATION ||--o{ PAYMENT : has

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

The above is Mermaid ER syntax; it can be placed inside a Mermaid code fence by documentation renderers.

## Interpretation

Inventory is identified by Property + RoomType + stay date.

ReservationRoom points to RoomType and RatePlan and does not require physical Room assignment.

ChannelRoomMapping maps the sellable RoomType, not each physical Room, unless a later channel requirement proves otherwise.

OutboxEvent represents a committed local change awaiting outbound processing. SyncJob and SyncAttempt are integration operational records.

WebhookEvent is the inbound durability/idempotency boundary.

## Simplified Flow

Property -> RoomType -> Room / RatePlan / Inventory
Property -> Reservation -> ReservationRoom -> RoomType + RatePlan
Reservation -> Guest + Payment

Property -> ChannelProperty -> Channel
ChannelProperty -> ChannelRoomMapping -> RoomType
ChannelProperty -> ChannelRateMapping -> RatePlan

Core transaction -> OutboxEvent -> SyncJob -> SyncAttempt -> OTA Adapter
OTA -> Webhook/API -> WebhookEvent -> Application Service
