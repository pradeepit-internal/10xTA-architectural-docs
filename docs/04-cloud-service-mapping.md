# Cloud Service Mapping: Azure ↔ AWS

## Overview

The ATS platform is designed to be cloud-agnostic, deployable on either **Microsoft Azure** or **Amazon Web Services (AWS)**. This document provides the service mapping between both cloud providers.

## Service Mapping

| Component | Azure Service | AWS Service | Purpose |
|-----------|---------------|-------------|---------|
| **Identity Provider** | External ID (B2C) | Cognito | User authentication, SSO, social login |
| **API Management** | API Management | API Gateway | API throttling, policies, documentation |
| **Container Orchestration** | AKS (Kubernetes) | EKS (Kubernetes) | Microservices deployment |
| **Container Registry** | Container Registry | ECR | Docker image storage |
| **Database (MongoDB)** | Cosmos DB (MongoDB API) | DocumentDB / MongoDB Atlas | Multi-tenant data storage |
| **Cache** | Azure Cache for Redis | ElastiCache (Redis) | Session, caching, rate limiting |
| **Object Storage** | Blob Storage | S3 | Resumes, documents, media files |
| **Message Broker** | Event Hubs (Kafka API) | MSK (Managed Kafka) | Async event processing |
| **Email Service** | Communication Services | SES | Transactional emails |
| **SMS Service** | Communication Services | SNS | SMS notifications |
| **Push Notifications** | Notification Hubs | SNS + Pinpoint | Mobile push notifications |
| **Secrets Management** | Key Vault | Secrets Manager | API keys, credentials |
| **Logging** | Log Analytics | CloudWatch Logs | Centralized logging |
| **Monitoring** | Azure Monitor | CloudWatch | Metrics, dashboards |
| **Tracing** | Application Insights | X-Ray | Distributed tracing |
| **CDN** | Azure CDN | CloudFront | Static asset delivery |
| **DNS** | Azure DNS | Route 53 | Domain management |
| **Load Balancer** | Application Gateway | ALB | Layer 7 load balancing |
| **Virtual Network** | VNet | VPC | Network isolation |
| **CI/CD** | Azure DevOps / GitHub Actions | CodePipeline / GitHub Actions | Build & deployment |
| **Infrastructure as Code** | ARM Templates / Terraform | CloudFormation / Terraform | Infrastructure provisioning |

## Architecture by Cloud Provider

### Azure Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Cloud                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ External ID │    │    API      │    │    AKS      │     │
│  │   (B2C)     │───▶│ Management  │───▶│ (Kubernetes)│     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                               │              │
│         ┌─────────────────────────────────────┤              │
│         ▼                   ▼                 ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Cosmos DB  │    │ Event Hubs  │    │    Blob     │     │
│  │ (MongoDB)   │    │  (Kafka)    │    │   Storage   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ Redis Cache │    │  Key Vault  │    │App Insights │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### AWS Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                      AWS Cloud                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Cognito   │───▶│ API Gateway │───▶│    EKS      │     │
│  │             │    │             │    │ (Kubernetes)│     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                               │              │
│         ┌─────────────────────────────────────┤              │
│         ▼                   ▼                 ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ DocumentDB  │    │    MSK      │    │     S3      │     │
│  │ (MongoDB)   │    │  (Kafka)    │    │             │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ ElastiCache │    │  Secrets    │    │  X-Ray +    │     │
│  │  (Redis)    │    │  Manager    │    │ CloudWatch  │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Key Considerations

### Identity Provider

| Feature | Azure External ID | AWS Cognito |
|---------|-------------------|-------------|
| Social Login | ✅ Native support | ✅ Native support |
| Enterprise SSO (SAML) | ✅ Built-in | ✅ Via federation |
| Custom Policies | ✅ Advanced (IEF) | ⚠️ Limited |
| LinkedIn Login | ✅ Native | ✅ Custom OIDC |
| Pricing Model | Per MAU | Per MAU |

**Recommendation**: Azure External ID has better enterprise SSO capabilities; Cognito is simpler for basic use cases.

### Database (MongoDB)

| Feature | Azure Cosmos DB | AWS DocumentDB | MongoDB Atlas |
|---------|-----------------|----------------|---------------|
| MongoDB Compatibility | 4.2 | 4.0 | Latest |
| Global Distribution | ✅ Native | ⚠️ Manual | ✅ Native |
| Serverless Option | ✅ Yes | ❌ No | ✅ Yes |
| Multi-tenant Support | ✅ Yes | ✅ Yes | ✅ Yes |

**Recommendation**: MongoDB Atlas on either cloud provides best compatibility; Cosmos DB if staying Azure-native.

### Message Broker (Kafka)

| Feature | Azure Event Hubs | AWS MSK |
|---------|------------------|---------|
| Kafka API Compatible | ✅ Yes | ✅ Native Kafka |
| Managed Service | ✅ Fully managed | ✅ Managed |
| Schema Registry | ✅ Built-in | ⚠️ Glue Schema Registry |
| Pricing | Per throughput unit | Per broker hour |

**Recommendation**: Both work well; Event Hubs is simpler, MSK is more Kafka-native.

## Environment Configuration

### Azure Environment Variables

```bash
# Identity
AZURE_AD_B2C_TENANT=yourtenant
AZURE_AD_B2C_CLIENT_ID=xxx
AZURE_AD_B2C_POLICY=B2C_1_SignUpSignIn

# Database
COSMOS_DB_CONNECTION_STRING=mongodb://...

# Storage
AZURE_STORAGE_ACCOUNT=yourstorageaccount
AZURE_STORAGE_CONTAINER=documents

# Cache
AZURE_REDIS_HOST=yourredis.redis.cache.windows.net
AZURE_REDIS_KEY=xxx

# Messaging
AZURE_EVENTHUBS_CONNECTION_STRING=Endpoint=sb://...
```

### AWS Environment Variables

```bash
# Identity
AWS_COGNITO_USER_POOL_ID=us-east-1_xxx
AWS_COGNITO_CLIENT_ID=xxx
AWS_REGION=us-east-1

# Database
MONGODB_URI=mongodb+srv://...

# Storage
AWS_S3_BUCKET=your-documents-bucket
AWS_S3_REGION=us-east-1

# Cache
AWS_ELASTICACHE_ENDPOINT=yourredis.cache.amazonaws.com

# Messaging
AWS_MSK_BOOTSTRAP_SERVERS=broker1:9092,broker2:9092
```

## Cost Comparison (Estimated Monthly)

| Component | Azure (Est.) | AWS (Est.) |
|-----------|--------------|------------|
| Identity (10K MAU) | $50 | $55 |
| Kubernetes (3 nodes) | $300 | $280 |
| Database (100GB) | $200 | $180 |
| Redis Cache (6GB) | $150 | $140 |
| Object Storage (500GB) | $10 | $12 |
| Kafka (basic) | $200 | $250 |
| **Total (approx)** | **~$910** | **~$917** |

*Costs vary by region and usage patterns. Use pricing calculators for accurate estimates.*

## Next Steps

- [System Overview](01-overview.md)
- [Authentication & Authorization](02-authentication.md)
- [Multi-Tenancy Strategy](03-multi-tenancy.md)
