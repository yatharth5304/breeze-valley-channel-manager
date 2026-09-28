# ADR 0009 — Date-Specific Rates and Restrictions

## Status
Proposed

## Decision
Keep RatePlan as the stable commercial definition and represent date-specific prices and restrictions separately as RateValue and RateRestriction concepts.

RateValue holds date/occupancy-dependent commercial amounts. RateRestriction holds date-specific selling controls such as MinLOS, MaxLOS, closed-for-sale, CTA, CTD, and applicable advance-booking limits.

Meal/package inclusion and cancellation policy belong to RatePlan; reservations snapshot the applicable commercial terms.

## Consequences
- Current rate-plan definitions can change without rewriting historical reservations.
- Rates and inventory remain separate concepts.
- Restrictions are explicit and diagnosable rather than encoded as prices or inventory.
- Exact occupancy, child pricing, cancellation, and OTA restriction capabilities remain TO VERIFY.
