# 🏗️ ATS Platform - Architectural Documentation

A comprehensive multi-tenant Applicant Tracking System (ATS) built with microservices architecture, designed for B2B SaaS recruitment solutions.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture Diagrams](#architecture-diagrams)
- [Documentation](#documentation)
- [Tech Stack](#tech-stack)
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

## Tech Stack

| Layer | Technology |
|-------|------------|
| **Identity** | Azure AD External ID / AWS Cognito |
| **Gateway** | Spring Cloud Gateway |
| **Services** | Spring Boot 3.x |
| **Database** | MongoDB (schema-per-tenant) |
| **Cache** | Redis Cluster |
| **Events** | Apache Kafka / Confluent |
| **Storage** | Azure Blob Storage |
| **AI/ML** | Custom services (Resume Parser, Ranking) |
| **Observability** | ELK Stack, Prometheus, Grafana, Jaeger |

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
