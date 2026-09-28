# Architecture Documentation

This directory contains architectural models, persistence design, concrete database schema design, application contracts, synchronization design, and technology proposals.

Start with:
- persistence-design.md — logical persistence mapping and transaction/inventory invariants.
- database-schema-design.md — concrete PostgreSQL tables, constraints, indexes, concurrency strategy, idempotency, outbox persistence, and migration strategy.
- application-interfaces.md — application-layer contracts and infrastructure boundaries.
- technology-stack.md — evaluated technology options and approved implementation direction.
- MODULE_BOUNDARIES.md — module responsibilities and dependency rules.
- synchronization.md — inbound/outbound synchronization model.

The technology direction is approved in principle. The concrete schema and migration strategy are proposed for review. No production schema, migration files, or application implementation are defined by these documents.
