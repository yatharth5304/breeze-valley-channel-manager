# OTA Integrations

Target channels currently include Agoda, Booking.com, MakeMyTrip, Goibibo, and Airbnb.

No adapter is implemented in the foundation phase.

## Adapter Responsibilities
An adapter is responsible for channel-specific concerns such as:
- authentication and credentials handling
- request/response formats
- external identifiers
- webhook/event parsing
- rate-limit behavior
- channel-specific retries where required
- translating internal inventory/rates/reservations to external contracts and back

## Core Boundary
Adapters must not own core hotel business rules. They should translate and delegate to application/domain services.

Each channel should be independently implementable and testable so one integration does not contaminate the others.
