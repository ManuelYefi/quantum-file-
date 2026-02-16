# ADR-004: Autoscaling Strategy with KEDA and HPA

## Status
Accepted

## Context
Processing load varies dramatically: from zero (no files) to high (1GB file = 100 chunks). Workers must scale dynamically to handle bursts while minimizing resource waste during idle periods.

## Decision
Use a dual autoscaling strategy:
1. **HPA (Horizontal Pod Autoscaler)**: CPU and memory-based scaling (baseline)
2. **KEDA (Kubernetes Event-Driven Autoscaling)**: Queue depth-based scaling (primary)

## Configuration
- **Min replicas**: 2 (high availability)
- **Max replicas**: 20 (resource ceiling)
- **KEDA trigger**: Scale when queue depth > 50 messages
- **HPA trigger**: Scale when CPU > 70% or memory > 80%
- **Scale-up**: Up to 4 pods per 60s window
- **Scale-down**: 25% reduction per 60s, with 120s stabilization

## Rationale
- **KEDA + AMQ**: Scales based on actual work pending (most accurate signal)
- **HPA as fallback**: Handles CPU-intensive processing scenarios
- **Conservative scale-down**: Prevents thrashing during bursty workloads
- **Min 2 replicas**: Ensures availability during rolling updates

## Consequences
- KEDA must be installed in the cluster
- AMQ metrics must be exposed for KEDA to query
- Dual scaling may occasionally conflict (mitigated by KEDA taking precedence)
