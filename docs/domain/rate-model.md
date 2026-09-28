# Rate and Restriction Model

## Status

**DECIDED:** RatePlan is the stable commercial definition attached to a RoomType.  
**DECIDED:** date-specific rates/restrictions are separate from RatePlan definition.  
**PROPOSED:** minimum date-specific model below.  
**TO VERIFY:** exact restriction vocabulary and channel-specific capabilities.

## RatePlan Definition

A RatePlan answers: what commercial package/rule is being sold?

Master concepts:
- rate_plan_id
- property_id
- room_type_id
- name/code
- meal/package inclusion
- cancellation policy
- occupancy rules
- status

Example:

Deluxe
- Room Only
- Breakfast Included

A RatePlan does not contain one permanent price because price varies by stay date and may vary by occupancy.

## Date-Specific Rate

A date-specific rate answers: what is the price for this RatePlan on this stay date and occupancy context?

Minimum proposed concepts:
- rate_plan_id
- stay_date
- occupancy context where pricing varies by occupancy
- base/nightly amount
- currency
- applicable pricing adjustment/value if required
- timestamps/version

Do not create a revenue-management engine. The model only needs commercial values that must eventually be synchronized.

## Occupancy-Based Pricing

**PROPOSED:** Support a rate value that can vary by occupancy.

Example:
- 1 adult: INR 4,000
- 2 adults: INR 4,500
- 2 adults + 1 child: INR 5,000

Exact child-age and occupancy calculation rules are **TO VERIFY**.

## Meal / Package Inclusion

Meal inclusion belongs to RatePlan because it describes the package being sold.

If a channel requires date-specific package pricing, the date-specific rate carries the amount while the package meaning remains part of RatePlan.

## Cancellation Policy

Cancellation policy belongs to the RatePlan commercial definition, with the reservation storing a historical snapshot of the policy applicable at booking time.

Represent policy as structured business data, not an OTA-specific blob.

Exact rules such as free-cancellation deadline, percentage/fixed penalties, and no-show treatment are **TO VERIFY**.

## Restrictions

Restrictions are date-specific selling controls and must not be mixed into Room master data.

Minimum proposed restrictions:
- minimum length of stay (MinLOS)
- maximum length of stay (MaxLOS)
- closed for sale
- closed to arrival (CTA)
- closed to departure (CTD)
- advance booking window where required

These belong to date-specific Rate/Availability data associated with the RatePlan.

A restriction is explicit and is not encoded as a magic price or inventory value.

## Date Range

A business rule may apply across a date range, but the canonical evaluated state remains date-specific. Storage optimization is a persistence concern.

## Rate vs Inventory

- Rate = commercial amount/package for a RatePlan.
- Inventory = quantity available for the RoomType.
- Restriction = whether/how the RatePlan may be sold for a date/stay pattern.

They must not be collapsed into one entity.

## Closed vs Zero Inventory

closed_for_sale is a commercial restriction, not simply available_inventory = 0.

A RoomType may have physical availability while a RatePlan is closed.

Conversely, inventory may be zero while the RatePlan remains open.

## Advance Booking Restrictions

**PROPOSED:** Represent minimum/maximum advance-booking constraints separately from stay-date price.

Exact semantics and whether both limits are required are **TO VERIFY**.

## Reservation Snapshot

ReservationRoom preserves booked price/currency and applicable commercial policy facts at booking time. Current RatePlan values must never be required to reconstruct historical reservation economics.

## Open Questions

- **TO VERIFY:** exact occupancy dimensions required initially.
- **TO VERIFY:** child-age bands and child pricing rules.
- **TO VERIFY:** whether MinLOS/MaxLOS and CTA/CTD apply per RatePlan or need property-wide controls.
- **TO VERIFY:** exact cancellation-policy structure.
- **TO VERIFY:** whether advance-booking restrictions are required by the first OTA.
- **TO VERIFY:** exact date-specific restriction matrix required by each target channel.
