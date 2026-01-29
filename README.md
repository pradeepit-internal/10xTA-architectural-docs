# 🏗️ 10xTA- Architectural Documentation

A comprehensive multi-tenant system  built with microservices architecture, designed for B2B SaaS recruitment solutions.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Documentation](#documentation)
- [Tech Stack](#tech-stack)
- [Cloud Deployment](#cloud-deployment)

## Overview

This repository contains architectural documentation for a modern, cloud-native ATS platform featuring:

- **Multi-Tenant Architecture** - One schema per tenant (MongoDB)
- **AI-Powered Features** - Resume parsing, candidate ranking, interview bots
- **Enterprise-Grade Security** - Azure AD B2C / AWS Cognito, SSO/SAML, RBAC
- **Event-Driven Design** - Apache Kafka for async communication
- **Cloud Agnostic** - Deployable on Azure or AWS

## Architecture Diagram

```
📁 diagrams/
└── ats-architecture.mermaid    # Complete system architecture with Azure/AWS mappings
```

> **Tip**: View `.mermaid` files directly on GitHub or use the [Mermaid Live Editor](https://mermaid.live)

## Documentation

| Document | Description |
|----------|-------------|
| [01-overview.md](docs/01-overview.md) | System overview and design principles |
| [02-authentication.md](docs/02-authentication.md) | Identity management and SSO integration |
| [03-multi-tenancy.md](docs/03-multi-tenancy.md) | Multi-tenant data isolation strategy |
| [04-cloud-service-mapping.md](docs/04-cloud-service-mapping.md) | Azure ↔ AWS service mapping |

## Tech Stack

| Layer | Azure | AWS | Purpose |
|-------|-------|-----|---------|
| **Identity** | External ID (B2C) | Cognito | Authentication, SSO |
| **Gateway** | API Management + Spring Cloud Gateway | API Gateway + Spring Cloud Gateway | Routing, rate limiting |
| **Services** | AKS | EKS | Microservices (Spring Boot) |
| **Database** | Cosmos DB (MongoDB API) | DocumentDB / MongoDB Atlas | Multi-tenant data |
| **Cache** | Azure Cache for Redis | ElastiCache | Sessions, caching |
| **Storage** | Blob Storage | S3 | Documents, media |
| **Messaging** | Event Hubs (Kafka API) | MSK | Async events |
| **Observability** | Monitor + App Insights | CloudWatch + X-Ray | Logging, tracing |

## Service Catalog

### Business Domain Services
- **Candidate Service** - Profile management, application tracking, talent pools
- **Recruiter Service** - Job management, pipelines, interview scheduling

### Technical Services
- **Tenant Service** - Provisioning, schema management, billing
- **Document Service** - File storage with pre-signed URLs
- **Notification Service** - Email, SMS, push notifications
- **Config Service** - Feature flags, tenant settings

### AI Services
- **Resume Parser** - Structured extraction from PDF/DOCX
- **Candidate Ranking** - ML-based job-candidate matching
- **Interview Bot** - Conversational AI for screening
- **Voice Analysis** - Transcription and sentiment analysis

## Cloud Deployment

The platform is designed to run on either **Azure** or **AWS**:

```
Choose ONE cloud provider:

┌─────────────────────────────┐    ┌─────────────────────────────┐
│         AZURE               │ OR │          AWS                │
├─────────────────────────────┤    ├─────────────────────────────┤
│ • External ID (B2C)         │    │ • Cognito                   │
│ • Cosmos DB                 │    │ • DocumentDB                │
│ • Event Hubs                │    │ • MSK                       │
│ • Blob Storage              │    │ • S3                        │
│ • AKS                       │    │ • EKS                       │
└─────────────────────────────┘    └─────────────────────────────┘
```

See [Cloud Service Mapping](docs/04-cloud-service-mapping.md) for detailed comparison.

## Async Communication (Kafka)

The platform uses **Apache Kafka** (via Azure Event Hubs or AWS MSK) for:

- **Decoupled Services** - Services communicate via events, not direct calls
- **Parallel Processing** - AI services process resumes/ranking concurrently
- **Reliability** - Messages persist until processed successfully
- **Scalability** - Add consumers to handle increased load

**Key Topics:**
| Topic | Purpose |
|-------|---------|
| `candidate.applications` | Application submissions and status changes |
| `resume.processing` | Queue for AI resume parsing |
| `notifications` | Email/SMS/Push notification triggers |
| `audit.events` | Compliance and audit trail |

## Getting Started

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/architectural-docs.git
cd architectural-docs

# View diagrams (requires Mermaid CLI or use GitHub preview)
npx @mermaid-js/mermaid-cli -i diagrams/ats-architecture.mermaid -o architecture.png
```

## License

MIT License - See [LICENSE](LICENSE) for details.

---

**Maintained by**: Abhijeet  
**Last Updated**: January 2025
