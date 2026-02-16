# ADR-005: Observability Stack

## Status
Accepted

## Context
A distributed file processing system requires comprehensive observability to detect failures, monitor performance, and enable capacity planning.

## Decision
Implement a full observability stack using:
- **Prometheus** for metrics collection and alerting
- **Grafana** for dashboards and visualization
- **Micrometer** for application-level metrics instrumentation
- **Spring Boot Actuator** for health checks and metrics exposure

## Key Metrics
| Metric | Source | Purpose |
|--------|--------|---------|
| `chunks_processed_total` | Worker | Throughput tracking |
| `lines_processed_total` | Worker | Detailed processing count |
| `lines_failed_total` | Worker | Error tracking |
| `chunk_processing_time` | Worker | Latency monitoring |
| `amq_queue_depth` | Broker | Backlog monitoring |
| `amq_dlq_depth` | Broker | Failure detection |
| `jvm_heap_usage_ratio` | JVM | Resource monitoring |

## Alerts
- **DLQ not empty** (critical): Immediate investigation required
- **High queue depth** (warning): Scaling may be needed
- **High CPU** (warning): Resource limits may need adjustment
- **High error rate** (critical): Business logic issues
- **Low throughput** (warning): Potential consumer issues

## Consequences
- Adds Prometheus and Grafana to infrastructure requirements
- Metrics endpoint adds minimal overhead (~1-2ms per scrape)
- Dashboard JSON is version-controlled and provisioned automatically
