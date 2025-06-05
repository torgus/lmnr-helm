# K8s Infrastructure Helm Chart

This Helm chart deploys a comprehensive Kubernetes infrastructure including:

- **Frontend**: Frontend application with ingress
- **App Server**: Backend application with load balancer
- **PostgreSQL**: Database StatefulSet with persistence
- **ClickHouse**: Analytics database StatefulSet with persistence
- **Redis**: In-memory data store
- **RabbitMQ**: Message broker StatefulSet with persistence
- **Standard Kubernetes Secrets**: Secure secret management
- **Storage Classes**: EBS storage configuration

## Prerequisites

- Kubernetes cluster (preferably EKS)
- Helm 3.x
- AWS Load Balancer Controller installed (for ingress/load balancers)
- EBS CSI driver installed (for persistent volumes)

## Installation

1. **Create the EKS cluster** (if not already created):
   ```bash
   eksctl create cluster --config-file=cluster-config.yaml
   ```

2. **Install the Helm chart**:
   ```bash
   helm install laminar ./helm-chart
   ```

3. **Install with custom secrets**:
   ```bash
   helm install laminar ./helm-chart \
     --set secrets.data.OPENAI_API_KEY="your-key" \
     --set secrets.data.NEXTAUTH_SECRET="your-secret" \
     --set postgres.password="secure-password" \
     --set clickhouse.password="secure-password" \
     --set rabbitmq.password="secure-password"
   ```

4. **Upgrade the installation**:
   ```bash
   helm upgrade laminar ./helm-chart
   ```

## Configuration

### Global Settings

```yaml
global:
  environment: "PRODUCTION"
  nodeGroupName: "t3a-xlarge"
```

### Component Configuration

Each component can be enabled/disabled and configured independently:

```yaml
frontend:
  enabled: true
  replicaCount: 1
  # ... other configuration

appServer:
  enabled: true
  replicaCount: 1
  # ... other configuration

postgres:
  enabled: true
  database: "lmnr"
  username: "lmnr"
  password: "lmnr123"  # Override in production
  persistence:
    enabled: true
    size: "20Gi"
  # ... other configuration

clickhouse:
  enabled: true
  database: "lmnr"
  username: "default"
  password: "clickhouse123"  # Override in production
  persistence:
    enabled: true
    size: "50Gi"
  # ... other configuration

redis:
  enabled: true
  # ... other configuration

rabbitmq:
  enabled: true
  username: "admin"
  password: "rabbitmq123"  # Override in production
  persistence:
    enabled: true
    size: "5Gi"
  # ... other configuration
```

### Image Configuration

The chart uses GitHub Container Registry (ghcr.io) for custom images:

```yaml
images:
  repository: "ghcr.io/lmnr-ai"
  frontend:
    name: "frontend"
    tag: "latest"
  appServer:
    name: "app-server"
    tag: "latest"
  postgres:
    repository: "postgres"
    tag: "15"
  clickhouse:
    repository: "clickhouse/clickhouse-server"
    tag: "latest"
```

### Secrets Configuration

The chart uses standard Kubernetes secrets:

```yaml
secrets:
  enabled: true
  data:
    OPENAI_API_KEY: ""
    DATABASE_URL: ""  # Auto-generated from postgres config
    CLICKHOUSE_URL: ""  # Auto-generated from clickhouse config
    RABBITMQ_URL: ""  # Auto-generated from rabbitmq config
    # ... many more secrets
```

**Important**: All secret values should be provided via values files or command-line arguments. Empty values in the secrets.data section will be ignored.

**Auto-generated URLs**:
- `DATABASE_URL`: `postgresql://username:password@postgres-service:5432/database`
- `CLICKHOUSE_URL`: `http://clickhouse-service:8123`
- `RABBITMQ_URL`: `amqp://username:password@rabbitmq-service:5672/`

### S3 Storage Configuration

If you want to store images and files in AWS S3, you need to configure the following secrets:

```yaml
secrets:
  data:
    AWS_ACCESS_KEY_ID: "your-aws-access-key-id"
    AWS_SECRET_ACCESS_KEY: "your-aws-secret-access-key"
    S3_TRACE_PAYLOADS_BUCKET: "your-s3-bucket-name"
```

**Required AWS IAM Permissions**:
Your AWS access key should have the following permissions on the S3 bucket:
- `s3:GetObject`
- `s3:PutObject`
- `s3:DeleteObject`
- `s3:ListBucket`

**Example IAM Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-s3-bucket-name/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::your-s3-bucket-name"
    }
  ]
}
```

**Installation with S3 secrets**:
```bash
helm install laminar ./helm-chart \
  --set secrets.data.AWS_ACCESS_KEY_ID="your-aws-access-key-id" \
  --set secrets.data.AWS_SECRET_ACCESS_KEY="your-aws-secret-access-key" \
  --set secrets.data.S3_TRACE_PAYLOADS_BUCKET="your-s3-bucket-name"
```

## Architecture Overview

The following diagram illustrates the complete architecture including ingress, load balancers, and internal service communication:

```mermaid
graph TB
    %% External Traffic
    Users[👥 Users<br/>Web Browser]
    API_Clients[🔌 API Clients<br/>gRPC/HTTP]
    Internet((Internet))
    
    %% AWS Load Balancers
    ALB[🔷 AWS ALB<br/>Application Load Balancer<br/>Port 80/443]
    NLB[🔶 AWS NLB<br/>Network Load Balancer<br/>Port 443/8443]
    
    %% Kubernetes Services
    FE_SVC[Frontend Service<br/>ClusterIP:80]
    AS_SVC[App Server Service<br/>ClusterIP:8000]
    
    %% Application Pods
    Frontend[🖥️ Frontend Pod<br/>React/Next.js<br/>Port 3000]
    AppServer[⚙️ App Server Pod<br/>Python/FastAPI<br/>Port 8000/8001]
    Nginx[🌐 Nginx Sidecar<br/>Port 8080]
    
    %% Database Services
    PG_SVC[PostgreSQL Service<br/>ClusterIP:5432]
    CH_SVC[ClickHouse Service<br/>ClusterIP:8123]
    Redis_SVC[Redis Service<br/>ClusterIP:6379]
    RMQ_SVC[RabbitMQ Service<br/>ClusterIP:5672]
    
    %% Database Pods
    PostgreSQL[(🗄️ PostgreSQL<br/>StatefulSet<br/>Port 5432)]
    ClickHouse[(📊 ClickHouse<br/>StatefulSet<br/>Port 8123/9000)]
    Redis[(🔴 Redis<br/>Deployment<br/>Port 6379)]
    RabbitMQ[(🐰 RabbitMQ<br/>StatefulSet<br/>Port 5672/15672)]
    
    %% Storage
    EBS_SC[📦 EBS StorageClass<br/>gp3 Volumes]
    PG_PVC[📀 PostgreSQL PVC<br/>20Gi]
    CH_PVC[📀 ClickHouse PVC<br/>50Gi]
    RMQ_PVC[📀 RabbitMQ PVC<br/>5Gi]
    
    %% Secrets
    Secrets[🔐 Kubernetes Secret<br/>app-secrets<br/>Credentials & Config]
    
    %% Optional S3
    S3[☁️ AWS S3<br/>File Storage<br/>Images/Payloads]
    
    %% Traffic Flow
    Users --> Internet
    API_Clients --> Internet
    Internet --> ALB
    Internet --> NLB
    
    ALB --> FE_SVC
    NLB --> AS_SVC
    
    FE_SVC --> Frontend
    AS_SVC --> AppServer
    AppServer --> Nginx
    
    %% Internal Communication
    Frontend -.->|API Calls| AS_SVC
    AppServer --> PG_SVC
    AppServer --> CH_SVC
    AppServer --> Redis_SVC
    AppServer --> RMQ_SVC
    Frontend --> Redis_SVC
    Frontend --> PG_SVC
    Frontend --> CH_SVC
    
    %% Service to Pod Communication
    PG_SVC --> PostgreSQL
    CH_SVC --> ClickHouse
    Redis_SVC --> Redis
    RMQ_SVC --> RabbitMQ
    
    %% Storage Relationships
    EBS_SC --> PG_PVC
    EBS_SC --> CH_PVC
    EBS_SC --> RMQ_PVC
    
    PG_PVC --> PostgreSQL
    CH_PVC --> ClickHouse
    RMQ_PVC --> RabbitMQ
    
    %% Secrets Usage
    Secrets -.->|Credentials| Frontend
    Secrets -.->|Credentials| AppServer
    Secrets -.->|Credentials| PostgreSQL
    Secrets -.->|Credentials| ClickHouse
    Secrets -.->|Credentials| RabbitMQ
    
    %% Optional S3 Connection
    AppServer -.->|File Upload/Download| S3
    
    %% Styling
    classDef external fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef loadbalancer fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef service fill:#e8f5e8,stroke:#2e7d32,stroke-width:2px
    classDef app fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef database fill:#fff8e1,stroke:#f57f17,stroke-width:2px
    classDef storage fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    classDef secrets fill:#ffebee,stroke:#d32f2f,stroke-width:2px
    classDef cloud fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    
    class Users,API_Clients,Internet external
    class ALB,NLB loadbalancer
    class FE_SVC,AS_SVC,PG_SVC,CH_SVC,Redis_SVC,RMQ_SVC service
    class Frontend,AppServer,Nginx app
    class PostgreSQL,ClickHouse,Redis,RabbitMQ database
    class EBS_SC,PG_PVC,CH_PVC,RMQ_PVC storage
    class Secrets secrets
    class S3 cloud
```

### Key Architecture Components:

**🌐 External Access:**
- **ALB (Application Load Balancer)**: Routes HTTP/HTTPS traffic to Frontend
- **NLB (Network Load Balancer)**: Routes API/gRPC traffic to App Server

**🔄 Internal Communication:**
- Frontend communicates with App Server via internal Kubernetes services
- App Server connects to all databases (PostgreSQL, ClickHouse, Redis, RabbitMQ)
- All credentials managed through centralized Kubernetes secrets

**💾 Data Persistence:**
- PostgreSQL, ClickHouse, and RabbitMQ use StatefulSets with persistent EBS volumes
- Redis runs as a Deployment (ephemeral, in-memory only)

**🔐 Security:**
- All database credentials stored in `app-secrets` Kubernetes secret
- Optional S3 integration for file storage with IAM-based access

## Components

### Frontend
- **Port**: 3000
- **Ingress**: AWS ALB with SSL termination
- **Environment**: React/Next.js application
- **Dependencies**: App Server, Redis, PostgreSQL, ClickHouse
- **Image**: `ghcr.io/lmnr-ai/frontend:latest`

### App Server (Backend, Span ingestion)
- **Port**: 8000 (HTTP), 8001 (gRPC)
- **Load Balancer**: AWS NLB for external access
- **Dependencies**: PostgreSQL, ClickHouse, Redis, RabbitMQ
- **Image**: `ghcr.io/lmnr-ai/app-server:latest`

### PostgreSQL
- **Port**: 5432
- **Type**: StatefulSet with persistent storage
- **Version**: PostgreSQL 15
- **Storage**: 20Gi by default (configurable)
- **Database**: `lmnr` (configurable)

### ClickHouse
- **Port**: 8123 (HTTP), 9000 (Native)
- **Type**: StatefulSet with persistent storage
- **Storage**: 50Gi by default (configurable)
- **Database**: `lmnr` (configurable)

### Redis
- **Port**: 6379
- **Type**: In-memory cache and session store
- **Image**: Valkey (Redis-compatible)

### RabbitMQ
- **Port**: 5672 (AMQP), 15672 (Management)
- **Type**: StatefulSet with persistent storage
- **Storage**: 5Gi by default (configurable)
- **Clustering**: Kubernetes-based peer discovery

## Storage

The chart includes an EBS storage class for persistent volumes:

```yaml
storage:
  storageClass:
    name: "ebs-sc"
    type: "gp3"
    reclaimPolicy: "Retain"
```

All StatefulSets use this storage class by default.

## Load Balancers

- **App Server**: NLB with SSL termination (ports 443, 8443)
- **Frontend**: ALB with SSL termination (port 443)

## Monitoring and Health Checks

All services include:
- Liveness probes
- Readiness probes  
- Resource limits and requests
- Proper logging configuration

## Security

- All secrets managed via Kubernetes secrets
- Database passwords should be overridden in production
- Pod security contexts where applicable

## Production Deployment

For production deployments, create a values file with secure passwords:

```yaml
# production-values.yaml
postgres:
  password: "your-secure-postgres-password"

clickhouse:
  password: "your-secure-clickhouse-password"

rabbitmq:
  password: "your-secure-rabbitmq-password"
  erlangCookie: "your-secure-erlang-cookie"

secrets:
  data:
    OPENAI_API_KEY: "your-openai-api-key"
    NEXTAUTH_SECRET: "your-nextauth-secret"
    # S3 Configuration (if storing images/files in S3)
    AWS_ACCESS_KEY_ID: "your-aws-access-key-id"
    AWS_SECRET_ACCESS_KEY: "your-aws-secret-access-key"
    S3_TRACE_PAYLOADS_BUCKET: "your-s3-bucket-name"
    # ... other production secrets
```

Then deploy with:
```bash
helm install laminar ./helm-chart -f production-values.yaml
```

## Customization

You can customize the installation by:

1. **Creating a custom values file**:
   ```bash
   helm install laminar ./helm-chart -f custom-values.yaml
   ```

2. **Setting individual values**:
   ```bash
   helm install laminar ./helm-chart --set frontend.replicaCount=3
   ```

3. **Disabling components**:
   ```bash
   helm install laminar ./helm-chart --set clickhouse.enabled=false
   ```

## Troubleshooting

### Common Issues

1. **Persistent Volume Claims pending**:
   - Ensure EBS CSI driver is installed
   - Check storage class configuration
   - Verify node availability in specified zones

2. **Database connection failures**:
   - Check if PostgreSQL/ClickHouse pods are running
   - Verify service endpoints: `kubectl get svc`
   - Check database passwords in secrets

3. **Load Balancer not created**:
   - Verify AWS Load Balancer Controller is installed
   - Review AWS subnet tags

### Useful Commands

```bash
# Check pod status
kubectl get pods -l app=frontend
kubectl get pods -l app=postgres

# View logs
kubectl logs -l app=app-server -f
kubectl logs -l app=postgres -f

# Check secrets
kubectl get secrets
kubectl describe secret app-secrets

# Check services
kubectl get svc
kubectl get ingress

# Check persistent volumes
kubectl get pvc
kubectl get pv

# Access databases (for debugging)
kubectl exec -it postgres-0 -- psql -U lmnr -d lmnr
kubectl exec -it clickhouse-0 -- clickhouse-client
```

## Backup and Recovery

### PostgreSQL
```bash
# Create backup
kubectl exec postgres-0 -- pg_dump -U lmnr lmnr > backup.sql

# Restore backup  
kubectl exec -i postgres-0 -- psql -U lmnr -d lmnr < backup.sql
```

### ClickHouse
```bash
# Create backup
kubectl exec clickhouse-0 -- clickhouse-client --query "BACKUP DATABASE lmnr TO 'backup.zip'"
```

## Uninstallation

```bash
helm uninstall laminar
```

**Note**: This will not delete persistent volumes. To delete everything including data:

```bash
helm uninstall laminar
kubectl delete pvc -l app=postgres
kubectl delete pvc -l app=clickhouse  
kubectl delete pvc -l app=rabbitmq
```

