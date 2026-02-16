# ADR-003: Idempotency Strategy with Redis

## Status
Accepted

## Context
AMQ provides at-least-once delivery. During redelivery scenarios (broker restart, consumer crash, network issues), chunks may be delivered more than once. We must prevent duplicate processing.

## Decision
Implement idempotency using Redis as a distributed store, with a local in-memory fallback.

## Design
- Each chunk has a unique `chunkId` (fileName + index + UUID)
- Before processing, check if `chunkId` exists in Redis (`SETNX` with 24h TTL)
- If key already exists, skip processing (duplicate detected)
- Fallback to `ConcurrentHashMap` if Redis is unavailable

## Rationale
- **Redis**: Fast, distributed, survives pod restarts, supports TTL
- **SETNX**: Atomic check-and-set prevents race conditions
- **Fallback**: System continues working without Redis (degraded mode)
- **TTL**: Prevents unbounded memory growth in Redis

## Consequences
- Requires Redis infrastructure (already used for other purposes)
- 24h TTL means reprocessing is possible after TTL expiry (acceptable for batch files)
- Local fallback loses idempotency guarantees across pod restarts
