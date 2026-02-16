# Quantum File Processor

Distributed file processing system for large files (1GB / 1M+ lines) using event-driven architecture with Apache Camel, RedHat AMQ 7.13, and horizontally scalable Kubernetes workers.

## Architecture

```
+-------------------+
|   Archivo 1GB     |
|  input/archivo    |
+-------------------+
          |
          v
+-------------------+
| Apache Camel      |
| - Streaming read  |
| - Split 10k lines |
| - Add metadata    |
+-------------------+
          |
          v
+-----------------------------+
| RedHat AMQ (Artemis)        |
| Queue: quantum.file.chunks  |
| - Persistence               |
| - DLQ + Redelivery          |
| - Round-robin distribution  |
+-----------------------------+
          |
     +---------+---------+
     |         |         |
     v         v         v
+--------+ +--------+ +--------+
|Worker 1| |Worker 2| |Worker N|
|  Pod   | |  Pod   | |  Pod   |
+--------+ +--------+ +--------+
     |         |         |
     v         v         v
+-------------------+
| Result / Storage  |
+-------------------+
```

## Project Structure

```
quantum-file-processor/
├── common/                     # Shared models, DTOs, utilities
│   └── src/main/java/com/quantum/common/
│       ├── model/              # Chunk, ChunkMetadata, FileProcessingRecord
│       ├── dto/                # ChunkResult
│       ├── config/             # JacksonConfig, QueueConstants
│       └── util/               # ChunkIdGenerator
├── file-ingester/              # Apache Camel file ingestion service
│   └── src/main/java/com/quantum/ingester/
│       ├── route/              # FileIngestionRoute (Camel)
│       ├── processor/          # ChunkMetadata, FileValidation processors
│       └── config/             # JMS, Metrics configuration
├── chunk-processor/            # Spring Boot worker service
│   └── src/main/java/com/quantum/processor/
│       ├── consumer/           # ChunkConsumer (JMS listener)
│       ├── service/            # Processing, Idempotency, Tracking services
│       ├── config/             # JMS, Redis, Metrics configuration
│       └── metrics/            # QueueMetrics
├── k8s/                        # Kubernetes manifests
│   ├── base/                   # Deployments, Services, HPA, ConfigMaps
│   ├── amq/                    # AMQ broker StatefulSet + config
│   ├── keda/                   # KEDA ScaledObject for queue-based scaling
│   └── monitoring/             # Prometheus, Grafana deployments
├── monitoring/                 # Monitoring configurations
│   ├── prometheus/             # Prometheus scrape config
│   └── grafana/                # Dashboard JSON
├── docs/                       # Documentation
│   ├── adr/                    # Architecture Decision Records
│   ├── architecture-recommendations.md
│   ├── cloud-adaptation.md
│   └── deployment-guide.md
├── docker-compose.yml          # Local development stack
└── input/                      # File input directory
```

## Components

| Component | Technology | Responsibility |
|-----------|-----------|----------------|
| File Reader | Apache Camel 4.4 | File ingestion, streaming split into 10K-line chunks |
| Message Broker | RedHat AMQ 7.13 (Artemis) | Persistent message distribution with DLQ |
| Worker Pods | Spring Boot 3.2 | Parallel chunk processing with idempotency |
| Idempotency Store | Redis 7 | Duplicate chunk detection (SETNX + TTL) |
| Orchestrator | Kubernetes | Horizontal scaling (HPA + KEDA) |
| Monitoring | Prometheus + Grafana | Metrics, dashboards, alerting |

## Key Features

- **Streaming ingestion**: Files read without loading into memory
- **Configurable chunk size**: Default 10,000 lines (env: `CHUNK_SIZE`)
- **Idempotent processing**: Redis-based deduplication with local fallback
- **Global file tracking**: Know when all chunks of a file are processed
- **Dead Letter Queue**: Failed chunks routed to DLQ after 5 retries
- **Queue-based autoscaling**: KEDA scales workers based on pending messages
- **Full observability**: Prometheus metrics, Grafana dashboards, alerting rules

## Quick Start

### Prerequisites
- Java 21+
- Maven 3.9+
- Docker & Docker Compose

### 1. Start infrastructure
```bash
docker-compose up -d amq-broker redis prometheus grafana
```

### 2. Build
```bash
mvn clean package -DskipTests
```

### 3. Run services
```bash
# Terminal 1 - File Ingester
cd file-ingester && mvn spring-boot:run

# Terminal 2 - Chunk Processor
cd chunk-processor && mvn spring-boot:run
```

### 4. Drop a test file
```bash
seq 1 100000 | awk '{print "LINE_"$1",field2,field3,field4"}' > input/test-file.csv
```

### 5. Monitor
- AMQ Console: http://localhost:8161 (admin/admin)
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (admin/admin)

## Message Flow

1. **File arrives** in `input/` directory
2. **Camel route** reads file in streaming mode
3. **Splitter** divides into 10K-line chunks with metadata (chunkId, totalChunks, fileName, timestamp)
4. **Each chunk** published to `quantum.file.chunks` queue (persistent)
5. **Worker pods** consume chunks via JMS listener (round-robin)
6. **Idempotency check** prevents duplicate processing
7. **Business logic** processes each line
8. **Results** sent to `quantum.file.results` queue
9. **Tracking service** monitors per-file completion progress
10. **Metrics** exposed via Prometheus for Grafana dashboards and alerts

## Queues

| Queue | Type | Purpose |
|-------|------|---------|
| `quantum.file.chunks` | Anycast | Chunk distribution to workers |
| `quantum.file.results` | Anycast | Processing results |
| `quantum.file.notifications` | Multicast | Event notifications (topic) |
| `DLQ.quantum.file.chunks` | Anycast | Failed chunks after max retries |

## Documentation

- [Deployment Guide](docs/deployment-guide.md)
- [Cloud Adaptation Guide](docs/cloud-adaptation.md)
- [Architecture Recommendations](docs/architecture-recommendations.md)
- [ADR-001: Event-Driven Architecture](docs/adr/001-event-driven-architecture.md)
- [ADR-002: Chunk-Based Splitting](docs/adr/002-chunk-based-splitting.md)
- [ADR-003: Idempotency Strategy](docs/adr/003-idempotency-strategy.md)
- [ADR-004: Autoscaling Strategy](docs/adr/004-autoscaling-strategy.md)
- [ADR-005: Observability Stack](docs/adr/005-observability-stack.md)

## Cloud Adaptation

| On-Prem | Azure | AWS | GCP |
|---------|-------|-----|-----|
| AMQ | Azure Service Bus | Amazon MQ / SQS | Pub/Sub |
| K8s Workers | AKS | EKS | GKE |
| Redis | Azure Cache | ElastiCache | Memorystore |
| Monitoring | Azure Monitor | CloudWatch | Cloud Monitoring |

See [Cloud Adaptation Guide](docs/cloud-adaptation.md) for details.

## License

MIT
