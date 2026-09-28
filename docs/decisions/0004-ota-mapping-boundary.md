# ADR 0004 — OTA Mapping Boundary

## Status
Proposed

## Context
Internal RoomType and RatePlan identifiers must remain independent of identifiers assigned by OTAs.

## Decision
Store explicit ChannelRoomMapping and ChannelRateMapping records under ChannelProperty.

Room mappings target the internal RoomType because that is the sellable inventory unit. Rate mappings target the internal RatePlan.

Adapters use mappings to translate between internal and external identifiers. Core entities do not store OTA-specific IDs.

## Consequences
- Adding a channel does not require contaminating core entities with external identifiers.
- Mapping failures can be detected before synchronization.
- Different channels can use different external identifiers for the same internal entity.
