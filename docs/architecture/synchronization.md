# Synchronization Model

## Direction A — OTA to Core
1. Receive OTA API response, webhook, or polling result.
2. Adapter authenticates/parses and converts the external payload to an internal representation.
3. Apply idempotency using channel and external identifiers/event identifiers.
4. Pass normalized data to the reservation/inventory application service.
5. Commit the local business change transactionally.
6. Record the resulting event/audit information.

## Direction B — Core to OTA
1. A local inventory/rate/reservation change occurs.
2. Record an internal event/outbox entry as part of the local transaction where appropriate.
3. A sync process claims work.
4. The relevant OTA adapter translates the internal change to the channel contract.
5. Send the request to the OTA.
6. Record success, failure, retry state, and relevant external identifiers.

## Failure Handling
The system should distinguish between:
- known business rejection
- authentication/authorization failure
- validation or mapping failure
- rate limiting
- transient network/API failure
- timeout with unknown external outcome

Retries should be deliberate and bounded. Unknown outcomes require reconciliation rather than blind duplication.

## Reconciliation
A later reconciliation process should be able to compare important local and external state where the channel API permits it. Reconciliation is a reliability mechanism, not a substitute for correct transactional processing.
