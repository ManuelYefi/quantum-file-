# Deployment Guide

## Prerequisites

- Java 21+
- Maven 3.9+
- Docker & Docker Compose
- Kubernetes cluster (for production)
- KEDA installed in cluster (for queue-based autoscaling)

## Local Development

### 1. Start Infrastructure

```bash
docker-compose up -d amq-broker redis prometheus grafana
```

### 2. Build the Project

```bash
mvn clean package -DskipTests
```

### 3. Run File Ingester

```bash
cd file-ingester
mvn spring-boot:run
```

### 4. Run Chunk Processor

```bash
cd chunk-processor
mvn spring-boot:run
```

### 5. Test with a File

Place a test file in the `input/` directory:

```bash
# Generate a test file with 100,000 lines
seq 1 100000 | awk '{print "LINE_"$1",field2,field3,field4"}' > input/test-file.csv
```

### 6. Monitor

- **AMQ Console**: http://localhost:8161 (admin/admin)
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 (admin/admin)
- **Ingester Metrics**: http://localhost:8081/actuator/prometheus
- **Processor Metrics**: http://localhost:8082/actuator/prometheus

## Kubernetes Deployment

### 1. Create Namespace

```bash
kubectl apply -f k8s/base/namespace.yml
```

### 2. Deploy Configuration

```bash
kubectl apply -f k8s/base/configmap.yml
kubectl apply -f k8s/base/secret.yml
```

### 3. Deploy AMQ Broker

```bash
kubectl apply -f k8s/amq/broker-config.yml
kubectl apply -f k8s/amq/amq-broker.yml
```

### 4. Deploy Redis

```bash
kubectl apply -f k8s/base/redis.yml
```

### 5. Build and Push Docker Images

```bash
# Build
docker build -t quantum/file-ingester:latest ./file-ingester
docker build -t quantum/chunk-processor:latest ./chunk-processor

# Tag and push to your registry
docker tag quantum/file-ingester:latest <registry>/quantum/file-ingester:latest
docker tag quantum/chunk-processor:latest <registry>/quantum/chunk-processor:latest
docker push <registry>/quantum/file-ingester:latest
docker push <registry>/quantum/chunk-processor:latest
```

### 6. Deploy Applications

```bash
kubectl apply -f k8s/base/file-ingester-deployment.yml
kubectl apply -f k8s/base/chunk-processor-deployment.yml
```

### 7. Configure Autoscaling

```bash
# HPA (CPU/Memory based)
kubectl apply -f k8s/base/hpa.yml

# KEDA (Queue depth based) - requires KEDA installed
kubectl apply -f k8s/keda/keda-scaledobject.yml
```

### 8. Deploy Monitoring

```bash
kubectl apply -f k8s/monitoring/prometheus.yml
kubectl apply -f k8s/monitoring/prometheus-rules.yml
kubectl apply -f k8s/monitoring/grafana.yml
kubectl apply -f k8s/monitoring/grafana-dashboards.yml
```

## Environment Variables

### File Ingester
| Variable | Default | Description |
|----------|---------|-------------|
| `AMQ_HOST` | localhost | AMQ broker hostname |
| `AMQ_PORT` | 61616 | AMQ broker port |
| `AMQ_USER` | admin | AMQ username |
| `AMQ_PASSWORD` | admin | AMQ password |
| `INPUT_DIR` | input | Input file directory |
| `CHUNK_SIZE` | 10000 | Lines per chunk |
| `SERVER_PORT` | 8081 | HTTP server port |

### Chunk Processor
| Variable | Default | Description |
|----------|---------|-------------|
| `AMQ_HOST` | localhost | AMQ broker hostname |
| `AMQ_PORT` | 61616 | AMQ broker port |
| `AMQ_USER` | admin | AMQ username |
| `AMQ_PASSWORD` | admin | AMQ password |
| `REDIS_HOST` | localhost | Redis hostname |
| `REDIS_PORT` | 6379 | Redis port |
| `CONSUMER_CONCURRENCY` | 1-5 | JMS listener concurrency |
| `CONSUMER_PREFETCH` | 10 | Message prefetch count |
| `SERVER_PORT` | 8082 | HTTP server port |
