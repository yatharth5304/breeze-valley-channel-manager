# PostgreSQL Database Schema Design

## Status

**PROPOSED — DESIGN ONLY.**

This document defines the concrete PostgreSQL schema proposal derived from the approved domain and persistence models. It does not create tables, migrations, application code, or a production database.

Approved implementation direction:
- TypeScript + Node.js
- NestJS
- PostgreSQL
- Drizzle
- Vitest
- PostgreSQL-backed transactional outbox initially
- one modular application
- no Redis, Kafka, Kubernetes, or microservices initially

## Design principles

1. Internal UUID identifiers are stable local identities and are never replaced by OTA identifiers.
2. date is used for stay-date concepts; timestamptz is used for event/audit timestamps.
3. Monetary values use numeric, never floating point.
4. Status values are stored as text with CHECK constraints rather than PostgreSQL enums initially.
5. Current master data and historical reservation facts are separate.
6. Historical reservation facts remain interpretable if current RatePlan, RateValue, RateRestriction, cancellation policy, meal/package definition, or names change.
7. Optional external identifiers use partial/conditional uniqueness.
8. No hard delete is permitted for reservations, reservation history, webhook events, sync history, or outbox history.
9. jsonb is used only for intentionally extensible data whose exact business shape remains TO VERIFY.
10. The schema does not encode unresolved policies such as overbooking, temporary holds, child-age pricing, OTA sequencing, payment/refund semantics, guest retention, or timezone rules.

## Proposed relationship map

Property
- RoomType
  - Room
  - RatePlan
    - RateValue
    - RateRestriction
- Inventory
- Reservation
  - ReservationRoom
  - ReservationGuest -> Guest
  - ReservationChange
  - Payment

Channel
- ChannelProperty -> Property
  - ChannelRoomMapping -> RoomType
  - ChannelRateMapping -> RatePlan
  - WebhookEvent
  - SyncJob -> SyncAttempt

Business transaction -> OutboxEvent -> SyncJob -> SyncAttempt

---

# 1. Core property and room tables

## 1.1 Property

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| property_id | uuid | NO | PK; application-generated UUID |
| code | varchar(64) | NO | Stable internal property code |
| name | varchar(200) | NO | Current property name |
| timezone | varchar(64) | NO | IANA timezone identifier; exact OTA/date interpretation TO VERIFY |
| status | text | NO | ACTIVE / INACTIVE |
| created_at | timestamptz | NO | Creation timestamp |
| updated_at | timestamptz | NO | Last modification timestamp |

Constraints:
- PK property_id
- UNIQUE code
- CHECK status in ACTIVE, INACTIVE

Indexes:
- unique index on code
- no speculative status index

Deletion/retention:
- deactivate rather than hard-delete once dependent operational data exists.

## 1.2 RoomType

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| room_type_id | uuid | NO | PK |
| property_id | uuid | NO | FK Property |
| code | varchar(64) | NO | Stable property-local code |
| name | varchar(200) | NO | Current display name |
| status | text | NO | ACTIVE / INACTIVE |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK room_type_id
- FK property_id -> Property
- UNIQUE property_id, code
- CHECK status in ACTIVE, INACTIVE

Indexes:
- property_id, status

Deletion:
- deactivate; retain if referenced by historical data.

## 1.3 Room

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| room_id | uuid | NO | PK |
| property_id | uuid | NO | FK Property |
| room_type_id | uuid | NO | FK RoomType |
| room_number | varchar(64) | NO | Property-local physical room identifier |
| status | text | NO | ACTIVE / OUT_OF_ORDER / OUT_OF_SERVICE / INACTIVE |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK room_id
- FKs to Property and RoomType
- UNIQUE property_id, room_number
- CHECK status in the four approved states

Indexes:
- room_type_id, status
- property_id, status

Deletion:
- use INACTIVE once operational/history references exist.

---

# 2. Rate tables

## 2.1 RatePlan

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| rate_plan_id | uuid | NO | PK |
| property_id | uuid | NO | FK Property |
| room_type_id | uuid | NO | FK RoomType |
| code | varchar(64) | NO | Stable property-local rate-plan code |
| name | varchar(200) | NO | Current commercial name |
| meal_package_code | varchar(64) | YES | Stable internal package code if used; exact model TO VERIFY |
| meal_package_label | varchar(200) | YES | Current descriptive label |
| cancellation_policy | jsonb | YES | Structured policy representation; exact fields TO VERIFY |
| occupancy_rules | jsonb | YES | Extensible occupancy definition; exact dimensions TO VERIFY |
| status | text | NO | ACTIVE / INACTIVE |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK rate_plan_id
- FKs to Property and RoomType
- UNIQUE room_type_id, code
- CHECK status in ACTIVE, INACTIVE

Indexes:
- room_type_id, status
- property_id, status only if operational queries require it

Historical rule:
- mutable policy/rule definitions here are never used to reconstruct an old reservation.

## 2.2 RateValue

One row represents one date plus pricing context for one RatePlan.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| rate_value_id | uuid | NO | PK |
| rate_plan_id | uuid | NO | FK RatePlan |
| stay_date | date | NO | Applicable night |
| occupancy_key | varchar(128) | NO | Canonical serialized occupancy key; exact dimensions TO VERIFY |
| amount | numeric(19,4) | NO | Exact nightly amount |
| currency | char(3) | NO | ISO-4217-style code |
| pricing_context | jsonb | YES | Extensible context only where required |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK rate_value_id
- FK rate_plan_id
- UNIQUE rate_plan_id, stay_date, occupancy_key
- CHECK amount >= 0
- currency format check for three uppercase letters

Indexes:
- rate_plan_id, stay_date

The occupancy key is intentionally not decomposed into invented adult/child dimensions until those rules are verified.

## 2.3 RateRestriction

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| rate_restriction_id | uuid | NO | PK |
| rate_plan_id | uuid | NO | FK RatePlan |
| stay_date | date | NO | Applicable night |
| min_los | integer | YES | TO VERIFY |
| max_los | integer | YES | TO VERIFY |
| closed_for_sale | boolean | NO | Default false |
| closed_to_arrival | boolean | NO | Default false |
| closed_to_departure | boolean | NO | Default false |
| min_advance_days | integer | YES | TO VERIFY |
| max_advance_days | integer | YES | TO VERIFY |
| additional_restrictions | jsonb | YES | Extension point for unresolved channel requirements |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FK rate_plan_id
- UNIQUE rate_plan_id, stay_date
- min_los >= 1 when present
- max_los >= 1 when present
- min_los <= max_los when both present
- advance values >= 0 when present

Indexes:
- rate_plan_id, stay_date

closed_for_sale remains distinct from inventory availability.

---

# 3. Inventory table

## 3.1 Inventory

Canonical grain: one Property + RoomType + stay date.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| inventory_id | uuid | NO | PK |
| property_id | uuid | NO | FK Property |
| room_type_id | uuid | NO | FK RoomType |
| stay_date | date | NO | Canonical inventory date |
| operational_capacity | integer | NO | Persisted coordination input/snapshot used by availability calculation |
| manual_block_quantity | integer | NO | Default 0 |
| reserved_quantity | integer | NO | Default 0 |
| override_mode | text | YES | Reserved for future approved semantics |
| override_value | integer | YES | Reserved for future approved semantics |
| version | bigint | NO | Optimistic/audit version; starts at 1 |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK inventory_id
- FKs to Property and RoomType
- UNIQUE property_id, room_type_id, stay_date
- operational_capacity >= 0
- manual_block_quantity >= 0
- reserved_quantity >= 0
- override_value >= 0 when present
- override_mode remains NULL until override semantics are approved

Indexes:
- room_type_id, stay_date
- property_id, stay_date

Availability is derived. No independently editable available_quantity is proposed.

operational_capacity must remain consistent with the approved inventory semantics: physical capacity less OUT_OF_ORDER/OUT_OF_SERVICE. The application is responsible for synchronizing it with Room state.

Overbooking is not represented by accidental negative availability.

---

# 4. Reservation and historical tables

## 4.1 Reservation

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| reservation_id | uuid | NO | PK |
| property_id | uuid | NO | FK Property |
| channel_id | uuid | YES | FK Channel; NULL for direct/local reservation |
| external_reservation_id | varchar(255) | YES | OTA reservation identity |
| status | text | NO | NEW / CONFIRMED / CANCELLED |
| confirmation_code | varchar(128) | YES | Local/customer-facing reference |
| booked_at | timestamptz | YES | Source booking timestamp if known |
| check_in_date | date | NO | Overall arrival |
| check_out_date | date | NO | Overall departure |
| currency | char(3) | YES | Reservation-level currency |
| total_amount | numeric(19,4) | YES | Historical booked total |
| source | text | NO | Initial vocabulary may include CHANNEL / DIRECT / LOCAL |
| external_version | varchar(255) | YES | Latest known OTA version/sequence token |
| external_updated_at | timestamptz | YES | OTA source modification timestamp |
| created_at | timestamptz | NO | Local creation |
| updated_at | timestamptz | NO | Local modification |

Constraints:
- PK
- FK Property
- nullable FK Channel
- partial UNIQUE channel_id, external_reservation_id where both are non-null
- CHECK status in approved three states
- CHECK check_out_date > check_in_date
- CHECK total_amount >= 0 when present

Indexes:
- property_id, status, check_in_date
- property_id, check_out_date
- unique partial external identity index
- property_id, created_at

Identity:
- reservation_id is internal identity
- channel_id + external_reservation_id is external reservation identity
- neither is webhook/event identity

Deletion:
- never hard-delete operational reservations.

## 4.2 ReservationRoom

ReservationRoom is the historical commercial snapshot boundary.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| reservation_room_id | uuid | NO | PK |
| reservation_id | uuid | NO | FK Reservation |
| room_type_id | uuid | YES | Current master reference; nullable to preserve history if retired |
| rate_plan_id | uuid | YES | Current master reference; nullable to preserve history |
| room_type_code_snapshot | varchar(64) | NO | Booking-time value |
| room_type_name_snapshot | varchar(200) | NO | Booking-time value |
| rate_plan_code_snapshot | varchar(64) | NO | Booking-time value |
| rate_plan_name_snapshot | varchar(200) | NO | Booking-time value |
| quantity | integer | NO | Rooms booked |
| adults | integer | NO | Occupancy snapshot |
| children | integer | NO | Occupancy snapshot |
| check_in_date | date | NO | Line arrival |
| check_out_date | date | NO | Line departure |
| currency | char(3) | NO | Line currency |
| total_amount | numeric(19,4) | NO | Historical line total |
| meal_package_snapshot | jsonb | YES | Booking-time meal/package facts |
| cancellation_policy_snapshot | jsonb | YES | Booking-time cancellation policy facts |
| occupancy_snapshot | jsonb | YES | Exact booking occupancy facts |
| pricing_snapshot | jsonb | YES | Nightly/tax/fee/discount facts |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FK Reservation
- optional FKs RoomType and RatePlan
- quantity > 0
- adults >= 0
- children >= 0
- check_out_date > check_in_date
- total_amount >= 0

Indexes:
- reservation_id
- room_type_id, check_in_date, check_out_date
- rate_plan_id, check_in_date, check_out_date

Historical rationale:
- snapshot columns are required even when current references exist
- pricing_snapshot remains extensible until exact taxes/fees/discount model is verified
- current RateValue rows are never required to reconstruct historical economics

For multi-night lines, pricing_snapshot should contain one nightly fact entry per stay date rather than requiring a later lookup of current RateValue.

## 4.3 Guest

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| guest_id | uuid | NO | PK |
| first_name | varchar(120) | YES | |
| last_name | varchar(120) | YES | |
| email | varchar(320) | YES | Not globally unique |
| phone | varchar(64) | YES | Not globally unique |
| address | jsonb | YES | Minimal extensible address/contact data |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- no global unique name/email/phone constraint

Indexes:
- email only if lookup is required
- phone only if lookup is required

Retention/privacy policy remains TO VERIFY.

## 4.4 ReservationGuest

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| reservation_guest_id | uuid | NO | PK |
| reservation_id | uuid | NO | FK Reservation |
| guest_id | uuid | NO | FK Guest |
| role | text | NO | PRIMARY / ADDITIONAL |
| first_name_snapshot | varchar(120) | YES | Booking-time value |
| last_name_snapshot | varchar(120) | YES | Booking-time value |
| email_snapshot | varchar(320) | YES | Booking-time value |
| phone_snapshot | varchar(64) | YES | Booking-time value |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FKs Reservation and Guest
- CHECK role in PRIMARY, ADDITIONAL
- UNIQUE reservation_id, guest_id, role
- partial unique reservation_id where role = PRIMARY to enforce at most one primary guest

Indexes:
- reservation_id, role
- guest_id

## 4.5 ReservationChange

Append-oriented history; not event sourcing.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| reservation_change_id | uuid | NO | PK |
| reservation_id | uuid | NO | FK Reservation |
| sequence_number | bigint | NO | Monotonic per reservation |
| change_type | text | NO | CREATED / MODIFIED / CANCELLED |
| source | text | NO | Origin |
| external_event_id | varchar(255) | YES | Source event identity |
| external_version | varchar(255) | YES | Source version/sequence token |
| occurred_at | timestamptz | NO | Business/source event time |
| recorded_at | timestamptz | NO | Local persistence time |
| changed_facts | jsonb | NO | Historical facts/diff/snapshot |
| created_at | timestamptz | NO | Local record timestamp |

Constraints:
- PK
- FK Reservation
- UNIQUE reservation_id, sequence_number
- CHECK change_type in CREATED, MODIFIED, CANCELLED
- changed_facts is a JSON object

Indexes:
- reservation_id, sequence_number
- reservation_id, occurred_at

## 4.6 Payment

Minimal channel-manager payment state; not an accounting ledger.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| payment_id | uuid | NO | PK |
| reservation_id | uuid | NO | FK Reservation |
| amount | numeric(19,4) | NO | Amount represented |
| currency | char(3) | NO | Currency |
| status | text | NO | PENDING / AUTHORIZED / CAPTURED / FAILED / REFUNDED / PARTIALLY_REFUNDED |
| method_type | varchar(64) | YES | Non-sensitive classification |
| provider | varchar(128) | YES | External payment provider/channel |
| external_payment_id | varchar(255) | YES | External payment reference |
| external_reference | varchar(255) | YES | Other provider reference |
| metadata | jsonb | YES | Non-secret provider metadata |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FK Reservation
- CHECK amount >= 0
- CHECK status in documented initial vocabulary
- no raw card/PIN/CVV/authentication secrets

Indexes:
- reservation_id, status
- partial unique provider, external_payment_id where both are non-null

Payment/refund semantics remain TO VERIFY.

---

# 5. Channel and mapping tables

## 5.1 Channel

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| channel_id | uuid | NO | PK |
| code | varchar(64) | NO | Internal stable channel code |
| name | varchar(200) | NO | Display name |
| status | text | NO | ACTIVE / INACTIVE |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- UNIQUE code
- CHECK status in ACTIVE, INACTIVE

## 5.2 ChannelProperty

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| channel_property_id | uuid | NO | PK |
| channel_id | uuid | NO | FK Channel |
| property_id | uuid | NO | FK Property |
| external_property_id | varchar(255) | YES | OTA property identifier |
| status | text | NO | ACTIVE / INACTIVE / ERROR |
| credentials_ref | varchar(255) | YES | Secret/config reference; never secret material |
| configuration | jsonb | YES | Non-secret channel config |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FKs Channel and Property
- UNIQUE channel_id, property_id
- partial unique channel_id, external_property_id where non-null
- CHECK status in ACTIVE, INACTIVE, ERROR

Indexes:
- property_id, status
- channel_id, status

## 5.3 ChannelRoomMapping

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| channel_room_mapping_id | uuid | NO | PK |
| channel_property_id | uuid | NO | FK ChannelProperty |
| room_type_id | uuid | NO | FK RoomType |
| external_room_id | varchar(255) | NO | OTA room identifier |
| status | text | NO | ACTIVE / INACTIVE |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FKs ChannelProperty and RoomType
- UNIQUE channel_property_id, room_type_id
- UNIQUE channel_property_id, external_room_id
- CHECK status in ACTIVE, INACTIVE

Indexes:
- channel_property_id, status
- room_type_id, status

## 5.4 ChannelRateMapping

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| channel_rate_mapping_id | uuid | NO | PK |
| channel_property_id | uuid | NO | FK ChannelProperty |
| rate_plan_id | uuid | NO | FK RatePlan |
| external_rate_id | varchar(255) | NO | OTA rate identifier |
| status | text | NO | ACTIVE / INACTIVE |
| created_at | timestamptz | NO | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FKs ChannelProperty and RatePlan
- UNIQUE channel_property_id, rate_plan_id
- UNIQUE channel_property_id, external_rate_id
- CHECK status in ACTIVE, INACTIVE

Indexes:
- channel_property_id, status
- rate_plan_id, status

---

# 6. Synchronization and idempotency tables

## 6.1 WebhookEvent

Webhook identity is distinct from reservation identity.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| webhook_event_id | uuid | NO | Local event identity |
| channel_property_id | uuid | NO | FK ChannelProperty |
| external_event_id | varchar(255) | YES | Provider event ID when supplied |
| event_type | varchar(128) | NO | Normalized/provider event type |
| received_at | timestamptz | NO | Local receipt time |
| occurred_at | timestamptz | YES | Provider event time |
| status | text | NO | RECEIVED / PROCESSING / PROCESSED / FAILED / IGNORED |
| attempt_count | integer | NO | Processing attempts |
| external_reservation_id | varchar(255) | YES | Associated reservation ID if known |
| payload | jsonb | YES | Raw/provider payload only if retention policy permits |
| payload_hash | varchar(128) | YES | Integrity/deduplication aid |
| last_error_code | varchar(128) | YES | Normalized error |
| last_error_message | text | YES | Safe diagnostic |
| processed_at | timestamptz | YES | |
| created_at | timestamptz | NO | |

Constraints:
- PK
- FK ChannelProperty
- CHECK status in approved values
- CHECK attempt_count >= 0
- partial UNIQUE channel_property_id, external_event_id where external_event_id is non-null

Indexes:
- channel_property_id, received_at
- status, received_at
- external_reservation_id

Duplicate webhook handling:
- first durable insert wins
- duplicate external event resolves to existing event
- no second reservation/inventory effect

Raw payload retention is TO VERIFY.

## 6.2 OutboxEvent

OutboxEvent is durable local intent created in the same transaction as the business change.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| outbox_event_id | uuid | NO | PK; event identity |
| aggregate_type | varchar(64) | NO | RESERVATION / INVENTORY / RATE etc. |
| aggregate_id | uuid | NO | Internal aggregate/reference ID |
| event_type | varchar(128) | NO | Semantic event type |
| channel_property_id | uuid | YES | Target integration scope |
| payload | jsonb | NO | Versioned outbound intent |
| status | text | NO | PENDING / CLAIMED / DISPATCHED / FAILED / DEAD |
| dedupe_key | varchar(255) | YES | Logical duplicate-suppression key |
| available_at | timestamptz | NO | Earliest dispatch time |
| attempt_count | integer | NO | Dispatch attempts |
| last_error_code | varchar(128) | YES | |
| last_error_message | text | YES | Safe diagnostic |
| created_at | timestamptz | NO | Transaction-local event timestamp |
| claimed_at | timestamptz | YES | |
| dispatched_at | timestamptz | YES | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- optional FK ChannelProperty
- CHECK status in approved values
- CHECK attempt_count >= 0
- dedupe_key uniqueness only where event semantics explicitly require it

Indexes:
- status, available_at, created_at
- channel_property_id, status, available_at
- aggregate_type, aggregate_id, created_at

Payload must be versioned by event type and contain facts needed by the adapter without secrets.

## 6.3 SyncJob

SyncJob is durable work derived from an OutboxEvent or reconciliation request.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| sync_job_id | uuid | NO | PK |
| outbox_event_id | uuid | YES | FK OutboxEvent; null for reconciliation |
| channel_property_id | uuid | NO | FK ChannelProperty |
| job_type | varchar(128) | NO | PUBLISH_INVENTORY / PUBLISH_RATES / RECONCILE etc. |
| aggregate_type | varchar(64) | YES | Target aggregate category |
| aggregate_id | uuid | YES | Target internal identity |
| status | text | NO | PENDING / CLAIMED / SUCCEEDED / RETRY / FAILED / DEAD |
| dedupe_key | varchar(255) | YES | Work coalescing key |
| available_at | timestamptz | NO | Next eligible processing time |
| attempt_count | integer | NO | |
| last_error_code | varchar(128) | YES | |
| last_error_message | text | YES | |
| created_at | timestamptz | NO | |
| claimed_at | timestamptz | YES | |
| completed_at | timestamptz | YES | |
| updated_at | timestamptz | NO | |

Constraints:
- PK
- FKs OutboxEvent and ChannelProperty
- CHECK status in approved values
- CHECK attempt_count >= 0

Indexes:
- status, available_at, created_at
- channel_property_id, status, available_at
- outbox_event_id
- optional unique channel_property_id, dedupe_key only where coalescing is explicitly correct

A failed job does not create a new reservation/inventory event.

## 6.4 SyncAttempt

SyncAttempt is immutable execution history for one SyncJob attempt.

| Column | PostgreSQL type | Null | Notes |
|---|---|---:|---|
| sync_attempt_id | uuid | NO | PK |
| sync_job_id | uuid | NO | FK SyncJob |
| attempt_number | integer | NO | Starts at 1 |
| started_at | timestamptz | NO | |
| finished_at | timestamptz | YES | |
| status | text | NO | RUNNING / SUCCEEDED / FAILED |
| request_reference | varchar(255) | YES | Safe provider request/correlation reference |
| response_code | varchar(64) | YES | Provider result code |
| error_code | varchar(128) | YES | Normalized error |
| error_message | text | YES | Safe diagnostic |
| response_metadata | jsonb | YES | Non-secret provider metadata |
| created_at | timestamptz | NO | |

Constraints:
- PK
- FK SyncJob
- UNIQUE sync_job_id, attempt_number
- CHECK attempt_number >= 1
- CHECK status in RUNNING, SUCCEEDED, FAILED

Indexes:
- sync_job_id, started_at
- status, started_at

Attempt rows are retained for audit/reconciliation.

---

# 7. Foreign-key deletion policy

Default FK behavior is restrictive, not cascading deletion, for historical and integration-sensitive records.

In particular:
- deleting Property must not cascade through reservations
- deleting RoomType/RatePlan must not destroy historical reservation facts
- deleting Reservation must not erase ReservationChange, Payment, or synchronization evidence
- deleting Channel/ChannelProperty must not erase webhook/sync history

For master configuration, deactivation is preferred.

---

# 8. Inventory concurrency strategy

## Transaction boundary

A reservation create, modification, or cancellation is one database transaction containing:

1. reservation identity/version validation;
2. determination of all affected Property + RoomType + stay-date Inventory rows;
3. deterministic locking of all affected rows;
4. availability validation;
5. reserved-quantity changes;
6. current Reservation/ReservationRoom update;
7. ReservationChange append;
8. required OutboxEvent insertion;
9. commit.

No external OTA HTTP call occurs inside this transaction.

## Affected-row set

For a reservation line covering [check_in, check_out), affected Inventory rows are every stay date from check-in through the night before check-out.

For multiple reservation lines:
- union the affected Property + RoomType + stay-date keys
- remove duplicates
- sort deterministically by room_type_id, then stay_date, then inventory_id
- lock/update in that order

This applies to multi-night, multi-room, room-type changes, date changes, quantity changes, and cancellation.

## Preferred initial mechanism

Use PostgreSQL row-level locking on existing Inventory rows, with an invariant that every affected row is locked in deterministic order before availability is evaluated.

Where appropriate, the final quantity update can also be guarded by a conditional predicate such as:

reserved_quantity + requested_delta <= sellable_capacity

The exact SQL belongs to implementation and is deliberately omitted here.

## Insufficient availability

If any affected night lacks sufficient inventory:
- the business transaction fails
- no Reservation change commits
- no reserved quantity changes
- no outbox event is created for the failed operation
- the application returns a domain-level insufficient-availability result

Overbooking remains TO VERIFY. Until explicitly approved, availability must not become negative.

## Multi-night and multi-room

All affected nights and room-type/date rows participate in one transaction. A partial reservation is not committed.

## Modification

Compute new consumption minus old consumption as a per-room-type/per-date delta.

A modification that releases some nights and consumes others locks the complete union of old and new rows before applying any delta.

## Cancellation

Lock the currently consumed inventory rows, verify the reservation is not already cancelled, release exactly the reservation's current consumption, set status CANCELLED, append ReservationChange, and create resulting OutboxEvent(s) in one transaction.

Repeated cancellation is idempotent and cannot release inventory twice.

## Concurrent requests

Two transactions competing for the same last unit serialize on the Inventory row.

The first successful transaction consumes the unit. The second re-evaluates availability against the latest committed state and fails with insufficient availability rather than overselling.

If PostgreSQL reports a deadlock or serialization failure under the chosen implementation strategy, the application transaction boundary may be retried with bounded retry/backoff.

Retries rerun the complete business transaction and must not independently emit duplicate business effects.

---

# 9. Reservation identity, idempotency, and late changes

Three separate identities exist:

1. Internal reservation identity: reservation_id
2. External reservation identity: channel_id + external_reservation_id
3. Webhook/event identity: channel_property_id + external_event_id when supplied

Database enforcement:
- partial unique Reservation index on channel_id + external_reservation_id
- partial unique WebhookEvent index on channel_property_id + external_event_id
- unique ReservationChange on reservation_id + sequence_number
- unique SyncAttempt on sync_job_id + attempt_number

Duplicate reservation delivery:
- resolve by channel + external reservation ID
- do not create another reservation
- compare source version/event identity
- return idempotent outcome when already represented
- apply modification only when it is a genuinely newer accepted state

Duplicate webhook delivery:
- unique event identity makes the first event durable
- duplicate resolves to existing event
- reservation/inventory effects are not repeated

Late changes:
- external version/sequence is retained when available
- older changes do not overwrite newer accepted state
- when reliable sequencing is unavailable, ambiguous changes are retained for reconciliation rather than guessed

Exact OTA sequencing remains TO VERIFY.

---

# 10. Outbox -> SyncJob -> SyncAttempt

## Atomic creation

A local business transaction that requires outbound synchronization inserts OutboxEvent in the same transaction.

Example:
Reservation change + Inventory change + ReservationChange + OutboxEvent

commit together.

This prevents business success without durable synchronization intent.

## Outbox identity

outbox_event_id is immutable event identity.

aggregate_type + aggregate_id identifies the local subject.

event_type identifies the semantic change.

dedupe_key is optional and only unique when the event semantics genuinely require coalescing.

## Payload

Payload is jsonb but is a versioned application contract, not arbitrary storage.

It contains the event schema/version, event type, aggregate identity, target channel where applicable, and business facts required by the adapter.

It contains no credentials or provider secrets.

It is not an event-sourced system.

## Claiming work

Initial workers use PostgreSQL-backed claiming.

Conceptually:
- select eligible PENDING/RETRY work ordered by available_at and created_at
- claim rows using row-level locking with SKIP LOCKED
- update claim state/lease metadata
- commit the claim
- perform OTA network I/O outside the transaction
- record success/failure afterward

A stale-claim recovery mechanism is required so a crashed worker cannot leave work permanently claimed.

## Retry

Retryable failures:
- increment attempt count
- record SyncAttempt
- set next available_at using bounded backoff
- retain the same business event/job

Non-retryable failures:
- record failure
- stop automatic retries when policy says the error is permanent
- retain evidence for reconciliation/manual action

No Redis queue is required.

## Attempt history

Every actual external execution gets one SyncAttempt row.

Retrying a job creates a new attempt number; it does not create a new reservation or business event.

---

# 11. Historical commercial snapshots

Historical reservation meaning must not depend on today's master data.

At booking/modification time, ReservationRoom stores:
- RoomType identity plus code/name snapshot
- RatePlan identity plus code/name snapshot
- quantity
- adults/children
- stay dates
- currency
- nightly pricing facts
- taxes/fees/discounts needed to explain the total
- meal/package facts
- cancellation policy
- occupancy facts

Current RateValue and RateRestriction are operational master data and are never the source of truth for historical reservation economics.

Because exact occupancy, child-age, cancellation, tax/fee, discount, and meal-package structures are still being verified, these historical fact groups use jsonb snapshots where necessary. Stable reservation-line fields remain relational.

ReservationChange retains change-specific historical facts/diffs sufficient for audit and reconciliation.

---

# 12. Indexing strategy

The schema starts with indexes supporting FK joins, uniqueness, reservation status/date queries, inventory date-range access, mapping resolution, webhook idempotency, pending-work claiming, and attempt history.

No speculative reporting, full-text, or high-cardinality indexes are proposed.

High-value indexes:
- RoomType: property_id, status
- Room: room_type_id, status
- RateValue: rate_plan_id, stay_date
- RateRestriction: rate_plan_id, stay_date
- Inventory: unique property_id, room_type_id, stay_date
- Inventory: room_type_id, stay_date
- Reservation: partial unique channel_id, external_reservation_id
- Reservation: property_id, status, check_in_date
- ReservationRoom: reservation_id
- ReservationRoom: room_type_id, check_in_date, check_out_date
- ReservationChange: unique reservation_id, sequence_number
- ReservationChange: reservation_id, occurred_at
- ReservationGuest: reservation_id, role
- ChannelProperty: partial unique channel_id, external_property_id
- ChannelRoomMapping: unique channel_property_id, room_type_id
- ChannelRoomMapping: unique channel_property_id, external_room_id
- ChannelRateMapping: unique channel_property_id, rate_plan_id
- ChannelRateMapping: unique channel_property_id, external_rate_id
- WebhookEvent: partial unique channel_property_id, external_event_id
- WebhookEvent: status, received_at
- OutboxEvent: status, available_at, created_at
- SyncJob: status, available_at, created_at
- SyncAttempt: unique sync_job_id, attempt_number

---

# 13. Migration strategy

## Ownership

Schema migrations are application-repository artifacts owned by the implementation codebase and reviewed with application changes.

Drizzle is the selected schema/migration toolchain direction. Exact Drizzle configuration and migration files are intentionally deferred.

A migration must be versioned, committed, deterministic, reviewed, applied in order, and identifiable by migration/version.

## Development workflow

1. Change the intended Drizzle schema definition.
2. Generate or author the corresponding SQL migration through the approved Drizzle workflow.
3. Review generated SQL, especially constraints, indexes, locks, and destructive operations.
4. Apply against a disposable development PostgreSQL database.
5. Run application/integration tests against PostgreSQL.
6. Verify upgrade from the immediately previous schema state.
7. Commit schema definition and migration together.

No migration file is created in this milestone.

## Production workflow

Production migrations should:
1. come from version-controlled migration artifacts;
2. run as an explicit deployment step before code that requires the new schema;
3. be observable and fail closed on migration error;
4. avoid long application locks where possible;
5. use expand/contract patterns for live-data changes.

Application startup should not silently mutate production schema.

## Rollback philosophy

Prefer forward fixes over automatic destructive rollback.

For additive changes:
- add nullable/new structures first
- deploy code using them
- backfill if required
- tighten constraints after data is compliant

For destructive changes:
- deprecate first
- stop writes
- verify no consumers remain
- remove in a later migration

An application rollback must not assume a destructive database rollback is safe.

## Destructive changes

Dropping columns/tables, tightening nullability, changing identifiers, or rewriting large tables requires explicit review, data-impact analysis, backup/restore consideration, and staged migration where practical.

## Seed/reference data

Reference data is distinct from test fixtures.

Potential reference data:
- Channel definitions for supported OTAs
- stable status/reference codes if not hard-coded

Seed data must be deterministic, idempotent, environment-aware, and free of secrets.

Property-specific operational data should not be silently inserted by generic migrations.

## Test database strategy

Integration tests should run against real PostgreSQL, not an SQLite substitute.

Preferred lifecycle:
- isolated database/schema per test suite or run
- apply the same versioned migrations used by the application
- load minimal deterministic fixtures
- test transaction/concurrency/idempotency invariants
- reset/destroy after the run

Concurrency tests must use separate PostgreSQL connections/transactions so locking behavior is real.

---

# 14. Unresolved business decisions intentionally preserved

The schema does not decide:
- whether overbooking is allowed or its allowance
- inventory override semantics
- temporary reservation holds
- occupancy dimensions
- child-age pricing
- exact cancellation-policy structure
- advance-booking restriction semantics
- OTA modification/version sequencing
- OTA timezone/date semantics
- OTA payment/refund semantics
- guest retention/privacy
- guest deduplication/merge

The schema provides explicit extension points without turning unresolved policies into accidental architecture.

---

# 15. Domain/architecture conflict check

**No contradiction was discovered.**

The concrete schema preserves:
- RoomType + stay date as inventory grain
- physical Room separate from sellable inventory
- Reservation current states NEW/CONFIRMED/CANCELLED
- MODIFIED as history
- Channel + external reservation ID as external reservation identity
- reusable Guest + ReservationGuest
- RatePlan vs date-specific RateValue/RateRestriction separation
- transactional Reservation + Inventory + Outbox behavior
- adapter/application boundaries
- no event sourcing
- one PostgreSQL-backed modular application

The only intentionally extensible areas are business rules explicitly marked TO VERIFY in the approved domain model.

---

# 16. Review outcome and next milestone

This milestone produces the concrete schema and migration strategy only.

After human review, the next milestone should be chosen explicitly. A reasonable candidate is to finalize the schema design and then prepare implementation planning for Drizzle schema definitions and migrations.

No migration, table, repository, service, API, frontend, or OTA implementation is created by this milestone.
