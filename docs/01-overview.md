# System Overview

## Introduction

The ATS (Applicant Tracking System) Platform is a B2B SaaS solution designed to help organizations streamline their recruitment processes. Built with a modern microservices architecture, it provides AI-powered features for candidate matching, resume parsing, and automated interviews.

## Design Principles

### 1. Multi-Tenancy First
Every architectural decision considers tenant isolation, data security, and scalability across multiple client organizations.

### 2. Event-Driven Architecture
Services communicate through Apache Kafka, enabling loose coupling, reliable message delivery, and audit trails for compliance.

### 3. API Gateway Pattern
Spring Cloud Gateway serves as the single entry point, handling authentication, authorization, rate limiting, and request routing.

### 4. Cloud-Native
Designed for deployment on Azure/AWS with containerized services, managed databases, and auto-scaling capabilities.

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
│              (Azure AD External ID / Cognito)                    │
│         Social Login │ Enterprise SSO │ MFA │ Custom Policies    │
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
          └───────────────────┼───────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│    MongoDB      │ │  Apache Kafka   │ │  Azure Blob     │
│ (Multi-Tenant)  │ │ (Event Stream)  │ │   Storage       │
└─────────────────┘ └─────────────────┘ └─────────────────┘
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
| **Document Service** | File upload/download via SAS URLs, version control, metadata management |
| **Notification Service** | Email (SendGrid/SES), SMS, push notifications, in-app alerts |
| **Config Service** | Feature flags, tenant-specific settings, system configuration |

### AI Services

| Service | Responsibilities |
|---------|-----------------|
| **Resume Parser** | PDF/DOCX text extraction, NLP processing, structured data output |
| **Candidate Ranking** | Job-candidate matching, scoring algorithms, recommendation engine |
| **Interview Bot** | Conversational AI, question generation, response evaluation |
| **Voice Analysis** | Speech-to-text, sentiment analysis, interview transcription |

## Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Availability** | 99.9% uptime SLA |
| **Latency** | P95 < 200ms for API responses |
| **Scalability** | Support 1000+ concurrent tenants |
| **Security** | SOC 2 Type II, GDPR, CCPA compliant |
| **Data Retention** | Configurable per tenant (default 7 years) |

## Next Steps

- [Authentication & Authorization](02-authentication.md)
- [Multi-Tenancy Strategy](03-multi-tenancy.md)
- [Event-Driven Architecture](04-event-driven-architecture.md)
- [Technology Stack](05-technology-stack.md)
