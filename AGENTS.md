# Project Constitution

## Purpose
Build a reliable hotel channel manager for a single property first, while keeping the domain model reusable for additional properties later.

## Architecture Rules
- Keep core business/domain logic independent from OTA-specific code.
- Use isolated OTA adapters/connectors behind explicit boundaries.
- Do not place OTA-specific behavior inside core reservation or inventory logic.
- Keep database access behind application/domain services rather than scattering persistence logic through the codebase.
- Prefer a simple architecture and avoid unnecessary microservices or infrastructure.

## Data Integrity
- The internal system is the source of truth for local inventory representation.
- Inventory changes and reservation effects must be transactionally consistent.
- External reservation identifiers must be idempotent.
- Sync operations must be retryable and auditable.

## Synchronization
- Support both inbound OTA events/reservations and outbound inventory/rate/reservation changes as the integration model evolves.
- Expect API failures, timeouts, duplicate events, rate limits, authentication failures, invalid mappings, partial sync, and network failures.
- Design retries and reconciliation deliberately rather than hiding failures.

## Change Discipline
Before changing architecture or implementation:
1. Read this file.
2. Read PROJECT_STATE.md.
3. Inspect the existing code and relevant architecture/domain documentation.
4. Do not silently change established architectural decisions.
5. Document significant decisions.
6. Update PROJECT_STATE.md after meaningful milestones.

## Security
- Never commit credentials, tokens, API keys, secrets, or production connection strings.
- Treat external payloads and credentials as untrusted input.
- Avoid sensitive data in logs.

## Dependencies and Infrastructure
- Add dependencies only when they solve a concrete need.
- Do not introduce Kubernetes, Kafka, service meshes, microservices, or other operational complexity without a demonstrated requirement.

## Testing
Prioritize tests around inventory correctness, reservation state changes, idempotency, mapping behavior, synchronization, retries, and failure handling.
