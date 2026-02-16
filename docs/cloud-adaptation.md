# Cloud Adaptation Guide

## Azure

| On-Premises | Azure Equivalent |
|-------------|-----------------|
| AMQ (Artemis) | Azure Service Bus |
| Workers (K8s) | AKS (Azure Kubernetes Service) |
| Camel Ingester | Container App or AKS |
| Prometheus/Grafana | Azure Monitor + Log Analytics |
| Redis | Azure Cache for Redis |
| File Storage | Azure Blob Storage |

### Scaling
- **KEDA + Azure Service Bus scaler**: Scale pods based on Service Bus queue length
- **AKS Cluster Autoscaler**: Automatically add/remove nodes

### Key Changes
- Replace Artemis JMS with Azure Service Bus SDK (`azure-messaging-servicebus`)
- Use Azure Blob Storage trigger instead of file system watcher
- Configure Managed Identity for authentication (no credentials in config)

---

## AWS

| On-Premises | AWS Equivalent |
|-------------|---------------|
| AMQ (Artemis) | Amazon MQ (ActiveMQ) or SQS |
| Workers (K8s) | EKS (Elastic Kubernetes Service) |
| File Storage | S3 |
| Prometheus/Grafana | CloudWatch + Managed Grafana |
| Redis | ElastiCache for Redis |

### Scaling
- **KEDA + SQS scaler**: Scale pods based on SQS ApproximateNumberOfMessages
- **EKS Cluster Autoscaler** or **Karpenter**: Node-level scaling

### Key Changes
- Replace file watcher with S3 event notification (SNS/SQS)
- Option: Replace workers with **AWS Lambda** for small chunks (< 6 min processing)
- Use IAM roles for service accounts (IRSA) for authentication

---

## GCP

| On-Premises | GCP Equivalent |
|-------------|---------------|
| AMQ (Artemis) | Pub/Sub |
| Workers (K8s) | GKE (Google Kubernetes Engine) |
| File Storage | Cloud Storage |
| Prometheus/Grafana | Cloud Monitoring + Managed Prometheus |
| Redis | Memorystore for Redis |

### Scaling
- **KEDA + Pub/Sub scaler**: Scale based on subscription backlog
- **GKE Autopilot**: Fully managed node scaling

### Key Changes
- Replace JMS with Pub/Sub client library
- Use Cloud Storage event notifications for file arrival
- Use Workload Identity for authentication
