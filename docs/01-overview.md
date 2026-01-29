# System Overview

## Introduction

The ATS (Applicant Tracking System) Platform is a B2B SaaS solution designed to help organizations streamline their recruitment processes. Built with a modern microservices architecture, it provides AI-powered features for candidate matching, resume parsing, and automated interviews.

The platform is **cloud-agnostic** and can be deployed on either **Microsoft Azure** or **Amazon Web Services (AWS)**.

## Design Principles

### 1. Multi-Tenancy First
Every architectural decision considers tenant isolation, data security, and scalability across multiple client organizations.

### 2. Event-Driven Architecture
Services communicate asynchronously through Apache Kafka, enabling loose coupling, parallel processing, and reliable message delivery.

### 3. API Gateway Pattern
Spring Cloud Gateway serves as the single entry point, handling authentication, authorization, rate limiting, and request routing.

### 4. Cloud-Agnostic Design
All services are containerized and use cloud-managed equivalents, allowing deployment on Azure or AWS without code changes.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENT APPLICATIONS                          │
│              (Web App, Mobile App, Third-Party)                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    IDENTITY PROVIDER                             │
│         Azure: External ID (B2C)  │  AWS: Cognito               │
│         Social Login │ Enterprise SSO │ MFA                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SPRING CLOUD GATEWAY                           │
│    AuthN/AuthZ │ Routing │ Rate Limiting │ Caching │ OpenAPI    │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ BUSINESS DOMAIN │ │ TECHNICAL SVCS  │ │   AI SERVICES   │
│    SERVICES     │ │                 │ │                 │
│─────────────────│ │─────────────────│ │─────────────────│
│ • Candidate     │ │ • Tenant        │ │ • Resume Parser │
│ • Recruiter     │ │ • Document      │ │ • Ranking       │
│                 │ │ • Notification  │ │ • Interview Bot │
│                 │ │ • Config        │ │ • Voice Analysis│
└─────────────────┘ └─────────────────┘ └─────────────────┘
          │                   │                   │
          └─────────────┬─────┴───────────────────┘
                        │
          ┌─────────────┼─────────────┬───────────────┐
          ▼             ▼             ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  MongoDB    │ │    Redis    │ │   Kafka     │ │   Object    │
│  Database   │ │   Cache     │ │  (Async)    │ │   Storage   │
│─────────────│ │─────────────│ │─────────────│ │─────────────│
│Azure:Cosmos │ │Azure:Redis  │ │Azure:Event  │ │Azure:Blob   │
│AWS:DocDB    │ │AWS:ElastiC. │ │  Hubs       │ │AWS:S3       │
│             │ │             │ │AWS:MSK      │ │             │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

## Service Responsibilities

### Business Domain Services

| Service | Responsibilities |
|---------|-----------------|
| **Candidate Service** | Profile CRUD, application tracking, candidate search, talent pool management |
| **Recruiter Service** | Job posting management, hiring pipeline, interview scheduling, workflow automation |

### Technical Services

| Service | Responsibilities |
|---------|-----------------|
| **Tenant Service** | Tenant onboarding, schema provisioning, billing integration, isolation enforcement |
| **Document Service** | File upload/download via pre-signed URLs, version control, metadata management |
| **Notification Service** | Email, SMS, push notifications, in-app alerts |
| **Config Service** | Feature flags, tenant-specific settings, system configuration |

### AI Services

| Service | Responsibilities |
|---------|-----------------|
| **Resume Parser** | PDF/DOCX text extraction, NLP processing, structured data output |
| **Candidate Ranking** | Job-candidate matching, scoring algorithms, recommendation engine |
| **Interview Bot** | Conversational AI, question generation, response evaluation |
| **Voice Analysis** | Speech-to-text, sentiment analysis, interview transcription |

## Async Communication with Kafka

Services communicate asynchronously through Kafka for:

- **Decoupled Processing** - Services don't wait for each other
- **Parallel Execution** - Multiple AI services can process simultaneously
- **Reliability** - Events are persisted until successfully processed
- **Scalability** - Add more consumers to handle load spikes

**Example Flow:**
```
Candidate Applies → [candidate.applications topic] → Notification Service (sends email)
                                                   → Resume Parser (extracts data)
                                                   → Analytics Service (updates metrics)
```

## Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Availability** | 99.9% uptime SLA |
| **Latency** | P95 < 200ms for API responses |
| **Scalability** | Support 1000+ concurrent tenants |
| **Security** | SOC 2 Type II, GDPR, CCPA compliant |
| **Data Retention** | Configurable per tenant (default 7 years) |

## Technology Recommendations

| Layer | Recommended Technology |
|-------|------------------------|
| **Backend Services** | Java 21 + Spring Boot 3.x |
| **AI/ML Services** | Python (FastAPI) |
| **Frontend** | React / Angular |
| **Container Orchestration** | Kubernetes (AKS/EKS) |
| **Infrastructure as Code** | Terraform |
| **CI/CD** | GitHub Actions |

## Next Steps

- [Authentication & Authorization](02-authentication.md)
- [Multi-Tenancy Strategy](03-multi-tenancy.md)
- [Cloud Service Mapping](04-cloud-service-mapping.md)
