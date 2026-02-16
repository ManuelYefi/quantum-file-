# Architecture Review & Recommendations

## Current Architecture Assessment

The architecture is well-designed and follows enterprise-grade patterns. It is event-driven, horizontally scalable, resilient, and production-ready.

---

## Implemented Improvements

### 1. Idempotency (Redis-based)
**Problem**: AMQ provides at-least-once delivery, meaning chunks could be reprocessed during redelivery.

**Solution implemented**:
- Redis `SETNX` with 24h TTL for each `chunkId`
- Automatic fallback to in-memory `ConcurrentHashMap` when Redis is unavailable
- Located in `IdempotencyService`

### 2. Backpressure Configuration
**Problem**: Workers overwhelmed by too many messages.

**Solution implemented**:
- Configurable JMS `prefetch` size (default: 10)
- Configurable consumer concurrency window (`1-5`)
- Session-transacted consumers (message acknowledged only after successful processing)
- AMQ broker configured with `PAGE` policy for address-full scenarios

### 3. Global File Tracking
**Problem**: No way to know when an entire file has been fully processed.

**Solution implemented**:
- `TrackingService` maintains per-file processing state
- Tracks `processedChunks`, `failedChunks`, `totalChunks`
- Automatically transitions to `COMPLETED` or `PARTIALLY_COMPLETED`
- Exposes progress as Prometheus gauge (`file.processing.progress`)

### 4. Separate Queues
**Problem**: Single queue mixes concerns.

**Solution implemented**:
- `quantum.file.chunks` (anycast) - chunk processing
- `quantum.file.results` (anycast) - processing results
- `quantum.file.notifications` (multicast/topic) - notifications
- `DLQ.quantum.file.chunks` - dead letter queue for failed chunks

### 5. KEDA Autoscaling
**Problem**: CPU-only scaling doesn't reflect actual workload.

**Solution implemented**:
- KEDA `ScaledObject` triggers on Artemis queue depth
- Scales from 2 to 20 replicas based on pending messages
- Fallback to 4 replicas if KEDA metrics collection fails
- HPA retained as secondary CPU/memory-based scaling

---

## Additional Recommendations

### Near-Term

1. **Circuit Breaker for External Calls**: If `LineProcessorService` calls external APIs, add Resilience4j circuit breaker to prevent cascading failures.

2. **Structured Logging**: Adopt JSON logging format with correlation IDs (`chunkId`, `fileName`) for easier log aggregation in ELK/Splunk.

3. **Health Check Enhancement**: Add custom health indicators that verify AMQ connectivity and Redis availability in the readiness probe.

4. **Graceful Shutdown**: Configure `spring.lifecycle.timeout-per-shutdown-phase=30s` to allow in-flight chunk processing to complete during pod termination.

### Mid-Term

5. **Schema Registry**: If chunk format evolves, consider Apache Avro + Schema Registry for backward-compatible serialization.

6. **Distributed Tracing**: Add OpenTelemetry for end-to-end tracing from file ingestion through chunk processing.

7. **File Re-processing API**: REST endpoint to re-trigger processing for a specific file (re-submit all chunks).

8. **Multi-format Support**: Abstract the line parser to support CSV, fixed-width, and JSON formats via strategy pattern.

### Long-Term

9. **Event Sourcing**: Store all processing events for audit trail and replay capability.

10. **Multi-Cluster**: For geo-distributed processing, consider AMQ federation or cross-cluster replication.
