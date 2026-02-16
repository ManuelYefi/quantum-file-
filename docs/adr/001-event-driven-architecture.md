# ADR-001: Event-Driven Architecture for File Processing

## Status
Accepted

## Context
We need to process large files (up to 1GB, ~1M lines) in a distributed, parallel, scalable, and resilient manner. The system must handle failures gracefully and scale horizontally based on load.

## Decision
We adopt an event-driven, message-based distributed processing architecture with the following components:
- **Apache Camel** for file ingestion and chunk splitting (streaming mode)
- **RedHat AMQ 7.13 (ActiveMQ Artemis)** as the message broker
- **Spring Boot workers** as chunk consumers running in Kubernetes

## Rationale
- **Decoupling**: Producer (Camel) and consumers (Workers) are fully decoupled via AMQ
- **Scalability**: Workers scale horizontally based on queue depth (KEDA) or CPU (HPA)
- **Resilience**: AMQ provides persistence, DLQ, and automatic redelivery
- **Streaming**: Camel reads files in streaming mode, avoiding full-file memory loading
- **Proven stack**: All components are battle-tested in enterprise environments

## Consequences
- Requires AMQ broker infrastructure
- Adds operational complexity (broker management, monitoring)
- Enables independent scaling of ingestion and processing
- Provides natural backpressure through queue depth
