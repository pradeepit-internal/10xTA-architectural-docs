# 🏗️ ATS Platform - Architectural Documentation

A comprehensive multi-tenant Applicant Tracking System (ATS) built with microservices architecture, designed for B2B SaaS recruitment solutions.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture Diagrams](#architecture-diagrams)
- [Documentation](#documentation)
- [Tech Stack](#tech-stack)
- [Why Java over Node.js?](#why-java-over-nodejs)
  - [Node.js Multi-Tenancy Challenge](#nodejs-multi-tenancy-challenge)
- [AWS vs Azure Cost Comparison](#aws-vs-azure-cost-comparison)
- [Service Catalog](#service-catalog)
- [Getting Started](#getting-started)

## Overview

This repository contains architectural documentation for a modern, cloud-native ATS platform featuring:

- **Multi-Tenant Architecture** - One schema per tenant (MongoDB)
- **AI-Powered Features** - Resume parsing, candidate ranking, interview bots
- **Enterprise-Grade Security** - Azure AD B2C, SSO/SAML, RBAC
- **Event-Driven Design** - Apache Kafka for reliable event streaming
- **Microservices** - Spring Boot ecosystem with Spring Cloud Gateway

## Architecture Diagrams

### System Architecture
![System Architecture](diagrams/ats-architecture-v2.mermaid)

```
📁 diagrams/
├── ats-architecture-v2.mermaid    # High-level system architecture
└── kafka-event-flow.mermaid       # Event streaming topology
```

> **Tip**: View `.mermaid` files directly on GitHub or use the [Mermaid Live Editor](https://mermaid.live)

## Documentation

| Document | Description |
|----------|-------------|
| [01-overview.md](docs/01-overview.md) | System overview and design principles |
| [02-authentication.md](docs/02-authentication.md) | Identity management and SSO integration |
| [03-multi-tenancy.md](docs/03-multi-tenancy.md) | Multi-tenant data isolation strategy |
| [04-event-driven-architecture.md](docs/04-event-driven-architecture.md) | Kafka event streaming patterns |
| [05-technology-stack.md](docs/05-technology-stack.md) | Technology choices and recommendations |

## Tech Stack

### Core Services (Java 21 + Spring Boot 3.2)
| Component | Technology |
|-----------|------------|
| **API Gateway** | Spring Cloud Gateway |
| **Business Services** | Spring Boot 3.2 |
| **Security** | Spring Security + OAuth2 |
| **Database** | Spring Data MongoDB |
| **Events** | Spring Kafka |

### AI/ML Services (Python + FastAPI)
| Component | Technology |
|-----------|------------|
| **Resume Parser** | spaCy, PyPDF2, python-docx |
| **Ranking Service** | scikit-learn, transformers |
| **Voice Analysis** | Whisper, sentiment analysis |

### Infrastructure
| Component | Technology |
|-----------|------------|
| **Identity** | Azure AD External ID |
| **Database** | MongoDB Atlas |
| **Cache** | Redis Cluster |
| **Events** | Apache Kafka / Confluent |
| **Storage** | Azure Blob Storage |
| **Observability** | ELK, Prometheus, Grafana, Jaeger |

## Why Java over Node.js?

For an enterprise B2B ATS platform, we chose **Java 21 + Spring Boot** over Node.js. Here's why:

### Comparison Matrix

| Factor | Java/Spring Boot | Node.js | Winner |
|--------|------------------|---------|--------|
| **Enterprise SSO/Auth** | Spring Security (15+ years battle-tested, native SAML/OIDC) | Passport.js (works but less mature) | ☕ Java |
| **Kafka Integration** | Spring Kafka (native, production-ready) | kafkajs (good but more boilerplate) | ☕ Java |
| **Multi-tenancy** | Well-documented patterns, Hibernate filters | Manual implementation required | ☕ Java |
| **Type Safety** | Compile-time checks, strong typing | TypeScript helps but runtime errors possible | ☕ Java |
| **Concurrency** | Virtual threads (Java 21), mature threading | Single-threaded, worker threads limited | ☕ Java |
| **Long-running Jobs** | Excellent thread management | Event loop blocking issues | ☕ Java |
| **Startup Time** | Slower (~5-10s) | Fast (~1-2s) | 🟢 Node |
| **Memory Usage** | Higher (~200-500MB) | Lower (~50-100MB) | 🟢 Node |
| **Dev Speed** | Slower initial setup | Faster prototyping | 🟢 Node |

### Key Decision Factors for ATS

```
✅ Enterprise Authentication (Azure AD B2C, SAML, SSO)
   → Spring Security has superior enterprise auth support

✅ Event-Driven Architecture (Kafka)
   → Spring Kafka offers first-class, production-grade integration

✅ Multi-Tenant Data Isolation
   → Spring ecosystem has proven patterns for tenant context propagation

✅ Compliance Requirements (GDPR, SOC 2)
   → Java's mature tooling for audit trails and security

✅ AI Service Integration
   → Better handling of CPU-intensive resume parsing with virtual threads
```

### Our Hybrid Approach

| Service Type | Technology | Reason |
|--------------|------------|--------|
| **Core Business Services** | Java 21 + Spring Boot | Enterprise patterns, auth, Kafka |
| **API Gateway** | Spring Cloud Gateway | Native ecosystem integration |
| **AI/ML Services** | Python + FastAPI | ML libraries, model inference |
| **Real-time (optional)** | Node.js | WebSockets if needed |

> 📖 For detailed analysis, see [05-technology-stack.md](docs/05-technology-stack.md)

### Node.js Multi-Tenancy Challenge

In Java, `ThreadLocal` provides tenant context per request. Node.js is single-threaded, so we need alternatives:

#### Solution: AsyncLocalStorage (Node.js 16+)

```javascript
// tenantContext.js
const { AsyncLocalStorage } = require('async_hooks');
const tenantStorage = new AsyncLocalStorage();

// Middleware to set tenant context
const tenantMiddleware = (req, res, next) => {
  const tenantId = req.headers['x-tenant-id'];
  tenantStorage.run({ tenantId }, () => next());
};

// Get tenant anywhere in async call chain
const getTenantId = () => {
  const store = tenantStorage.getStore();
  return store?.tenantId;
};

// Usage in repository
const getCandidates = async () => {
  const tenantId = getTenantId();
  const db = mongoose.connection.useDb(`ats_tenant_${tenantId}`);
  return db.collection('candidates').find({}).toArray();
};
```

#### NestJS Approach (Recommended for Node.js)

```typescript
// tenant.middleware.ts
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  constructor(private readonly cls: ClsService) {}
  
  use(req: Request, res: Response, next: NextFunction) {
    const tenantId = req.headers['x-tenant-id'] as string;
    this.cls.set('tenantId', tenantId);
    next();
  }
}

// Any service can access tenant
@Injectable()
export class CandidateService {
  constructor(private readonly cls: ClsService) {}
  
  getTenantId(): string {
    return this.cls.get('tenantId');
  }
}
```

| Approach | Java | Node.js |
|----------|------|---------|
| **Mechanism** | ThreadLocal | AsyncLocalStorage |
| **Library** | Native | Native (16+) or `cls-hooked` |
| **Framework** | Spring Context | NestJS + `nestjs-cls` |
| **Reliability** | Battle-tested | Works but less mature |

> ⚠️ **Caveat**: AsyncLocalStorage has edge cases with some libraries. Java's ThreadLocal is more predictable for enterprise multi-tenancy.

---

## AWS vs Azure Cost Comparison

Monthly cost estimate for production ATS platform (India regions):

### Compute & Orchestration

| Service | AWS | Azure | Notes |
|---------|-----|-------|-------|
| **Kubernetes** | EKS + 3x m5.xlarge | AKS + 3x D4s_v3 | |
| | ₹42,000/mo | ₹38,000/mo | AKS control plane free |
| **Container Registry** | ECR (50GB) | ACR Basic | |
| | ₹400/mo | ₹350/mo | |

### Database & Cache

| Service | AWS | Azure | Notes |
|---------|-----|-------|-------|
| **MongoDB** | DocumentDB (or Atlas) | Cosmos DB (MongoDB API) | |
| | ₹45,000/mo | ₹55,000/mo | Atlas on either: ₹35,000 |
| **Redis Cache** | ElastiCache r6g.large | Azure Cache P1 | |
| | ₹22,000/mo | ₹25,000/mo | |

### Event Streaming

| Service | AWS | Azure | Notes |
|---------|-----|-------|-------|
| **Kafka** | MSK (kafka.m5.large x3) | Event Hubs Premium | |
| | ₹35,000/mo | ₹28,000/mo | Or Confluent: ₹20,000 |

### Storage & CDN

| Service | AWS | Azure | Notes |
|---------|-----|-------|-------|
| **Object Storage** | S3 (500GB + requests) | Blob Storage Hot | |
| | ₹1,200/mo | ₹1,000/mo | |
| **CDN** | CloudFront | Azure CDN | |
| | ₹2,500/mo | ₹2,000/mo | |

### Identity & Security

| Service | AWS | Azure | Notes |
|---------|-----|-------|-------|
| **Identity Provider** | Cognito (50k MAU) | Azure AD B2C (50k MAU) | |
| | ₹12,000/mo | ₹15,000/mo | B2C has better enterprise SSO |
| **Secrets Manager** | Secrets Manager | Key Vault | |
| | ₹800/mo | ₹600/mo | |

### Observability

| Service | AWS | Azure | Notes |
|---------|-----|-------|-------|
| **Logging** | CloudWatch Logs | Azure Monitor | |
| | ₹8,000/mo | ₹7,000/mo | |
| **APM** | X-Ray | App Insights | |
| | ₹5,000/mo | ₹4,500/mo | |

### Total Comparison

| Category | AWS (₹/mo) | Azure (₹/mo) |
|----------|------------|--------------|
| Compute | 42,400 | 38,350 |
| Database | 67,000 | 80,000 |
| Events | 35,000 | 28,000 |
| Storage | 3,700 | 3,000 |
| Identity | 12,800 | 15,600 |
| Observability | 13,000 | 11,500 |
| **TOTAL** | **₹1,73,900** | **₹1,76,450** |

### Recommendation

| Criteria | Winner | Reason |
|----------|--------|--------|
| **Overall Cost** | 🟡 Tie | ~2% difference |
| **Enterprise SSO** | 🔵 Azure | AD B2C superior for SAML/enterprise federation |
| **Kafka Native** | 🟠 AWS | MSK is true Kafka; Event Hubs is "Kafka-compatible" |
| **MongoDB** | 🟢 Atlas | Use MongoDB Atlas on either cloud (best compatibility) |
| **Kubernetes** | 🔵 Azure | AKS control plane is free |
| **India Region** | 🔵 Azure | Better availability zones in Central India |

### 💡 Our Recommendation: **Azure** for this ATS platform

```
Reasons:
1. Azure AD B2C is best-in-class for enterprise SSO (critical for B2B)
2. AKS control plane is free (saves ₹6,000+/mo)
3. Better integration with Office 365 (enterprise clients use it)
4. Use MongoDB Atlas instead of Cosmos DB (true MongoDB compatibility)
5. Use Confluent Cloud for Kafka (works on both clouds)
```

### Optimized Azure Stack (Recommended)

| Service | Provider | Monthly Cost (₹) |
|---------|----------|------------------|
| AKS (3x D4s_v3) | Azure | 38,000 |
| MongoDB Atlas M30 | MongoDB | 35,000 |
| Azure Cache Redis P1 | Azure | 25,000 |
| Confluent Basic | Confluent | 20,000 |
| Blob Storage | Azure | 1,000 |
| Azure AD B2C | Azure | 15,000 |
| Observability | Azure | 11,500 |
| **TOTAL** | | **₹1,45,500/mo** |

---

## Service Catalog

### Business Domain Services
- **Candidate Service** - Profile management, application tracking, talent pools
- **Recruiter Service** - Job management, pipelines, interview scheduling

### Technical Services
- **Tenant Service** - Provisioning, schema management, billing
- **Document Service** - File storage with Azure Blob SAS URLs
- **Notification Service** - Email, SMS, push notifications
- **Config Service** - Feature flags, tenant settings

### AI Services
- **Resume Parser** - Structured extraction from PDF/DOCX
- **Candidate Ranking** - ML-based job-candidate matching
- **Interview Bot** - Conversational AI for screening
- **Voice Analysis** - Transcription and sentiment analysis

## Getting Started

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/architectural-docs.git
cd architectural-docs

# View diagrams (requires Mermaid CLI or use GitHub preview)
npx @mermaid-js/mermaid-cli -i diagrams/ats-architecture-v2.mermaid -o output.png
```

## License

MIT License - See [LICENSE](LICENSE) for details.

---

**Maintained by**: Abhijeet  
**Last Updated**: January 2025
