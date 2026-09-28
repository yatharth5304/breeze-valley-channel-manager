# Development

## Current Phase
Phase 0 — architecture and foundation.

## Working Rules
- Read AGENTS.md and PROJECT_STATE.md before substantive changes.
- Inspect existing code before introducing new modules or dependencies.
- Keep domain logic testable without requiring an OTA connection.
- Keep integration code isolated from core domain rules.
- Prefer small, reviewable changes.
- Update documentation when architecture or significant domain decisions change.

## Technology
The concrete implementation stack is intentionally not finalized in Phase 0.

## Testing Priorities
Tests should eventually cover:
- inventory calculations and availability
- reservation creation, modification, and cancellation
- external reservation idempotency
- room/rate/channel mappings
- outbound synchronization
- inbound synchronization
- retry behavior and failure states
- transactional consistency

## Git Practices
- Use focused commits with clear messages.
- Do not commit secrets or local environment files.
- Do not modify unrelated repositories or project areas.
