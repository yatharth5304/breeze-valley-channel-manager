# ADR 0001 — Initial Architecture Boundaries

## Status
Accepted as the initial project constraint.

## Context
The product needs to synchronize a single property's inventory, rates, and reservations across multiple OTAs while remaining maintainable and reliable.

## Decision
Start with a simple modular application architecture with:
- a core hotel domain
- application services for business operations
- a persistence boundary
- an explicit synchronization/event boundary
- isolated OTA adapters

Keep the internal inventory representation authoritative locally and make external reservation handling idempotent.

## Consequences
- OTA-specific behavior stays isolated.
- Core reservation and inventory rules can be tested without OTA access.
- Adding another channel should primarily require a new adapter and mappings rather than changes throughout the core.
- The architecture does not prescribe microservices, Kubernetes, Kafka, or a particular programming language/framework.

## Related Documentation
See ARCHITECTURE.md, AGENTS.md, and docs/domain/domain-model.md.
