# ADR 0005 — Outbox and Synchronization Boundary

## Status
Proposed

## Context
A committed local inventory/rate/reservation change must not silently lose its outbound synchronization intent. OTA calls can fail independently of the local transaction.

## Decision
Represent outbound intent as an OutboxEvent created within the same local transaction as the business change. Synchronization processing turns that durable intent into SyncJob work and records each execution as a SyncAttempt.

Inbound external deliveries first cross a durable WebhookEvent/idempotency boundary before core application services apply business effects.

OTA adapters do not mutate core persistence directly.

## Consequences
- Local business success and outbound intent remain transactionally coupled.
- Retries are observable without creating a new business event for every attempt.
- Unknown external outcomes can be reconciled rather than blindly duplicated.
- Exact worker/queue mechanism remains TO VERIFY and is intentionally not selected here.
