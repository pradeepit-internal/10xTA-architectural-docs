# Multi-Tenancy Strategy

## Overview

The ATS platform implements a **schema-per-tenant** (database-per-tenant in MongoDB terms) isolation model, providing strong data separation while maintaining operational efficiency.

## Isolation Models Comparison

| Model | Isolation | Cost | Complexity | Use Case |
|-------|-----------|------|------------|----------|
| **Shared Schema** | Low | Low | Low | Early-stage, small tenants |
| **Schema-per-Tenant** ✓ | High | Medium | Medium | Enterprise B2B SaaS |
| **Database-per-Tenant** | Highest | High | High | Regulated industries |

**Our Choice: Schema-per-Tenant** provides the right balance of isolation for enterprise clients while keeping infrastructure costs manageable.

## Database Options

| Provider | Azure | AWS |
|----------|-------|-----|
| **Managed MongoDB** | Cosmos DB (MongoDB API) | DocumentDB |
| **MongoDB Atlas** | Available on Azure | Available on AWS |

**Recommendation**: MongoDB Atlas provides best compatibility and can run on either cloud provider.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              MongoDB Cluster (Atlas / Cosmos / DocumentDB)       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐ │
│  │  ats_tenant_001  │  │  ats_tenant_002  │  │ ats_tenant_003 │ │
│  │  ──────────────  │  │  ──────────────  │  │ ────────────── │ │
│  │  • candidates    │  │  • candidates    │  │ • candidates   │ │
│  │  • jobs          │  │  • jobs          │  │ • jobs         │ │
│  │  • applications  │  │  • applications  │  │ • applications │ │
│  │  • interviews    │  │  • interviews    │  │ • interviews   │ │
│  │  • users         │  │  • users         │  │ • users        │ │
│  └──────────────────┘  └──────────────────┘  └────────────────┘ │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    ats_system (Shared)                    │   │
│  │  ────────────────────────────────────────────────────────│   │
│  │  • tenants (registry)    • system_config                 │   │
│  │  • master_data           • audit_logs                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Tenant Resolution Flow

```
┌──────────┐     ┌─────────────┐     ┌───────────────┐     ┌──────────────┐
│  Request │────▶│   Gateway   │────▶│ Tenant Filter │────▶│   Service    │
│  + JWT   │     │             │     │               │     │              │
└──────────┘     └─────────────┘     └───────────────┘     └──────────────┘
                       │                     │                     │
                       ▼                     ▼                     ▼
                 ┌───────────┐        ┌───────────┐         ┌───────────┐
                 │ Extract   │        │ Validate  │         │ Connect   │
                 │ tenantId  │        │ tenant    │         │ to tenant │
                 │ from JWT  │        │ exists    │         │ database  │
                 └───────────┘        └───────────┘         └───────────┘
```

## Implementation

### 1. Tenant Context Holder (Thread-Safe)

```java
public class TenantContext {
    
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();
    
    public static void setTenantId(String tenantId) {
        currentTenant.set(tenantId);
    }
    
    public static String getTenantId() {
        return currentTenant.get();
    }
    
    public static void clear() {
        currentTenant.remove();
    }
}
```

### 2. Dynamic Database Routing

```java
@Configuration
public class MultiTenantMongoConfig {

    @Bean
    public MongoDatabaseFactory mongoDatabaseFactory(MongoClient mongoClient) {
        return new SimpleMongoClientDatabaseFactory(mongoClient, "ats_system") {
            @Override
            public MongoDatabase getMongoDatabase() throws DataAccessException {
                String tenantId = TenantContext.getTenantId();
                if (tenantId != null) {
                    String databaseName = "ats_tenant_" + tenantId;
                    return mongoClient.getDatabase(databaseName);
                }
                return super.getMongoDatabase(); // System database
            }
        };
    }
}
```

### 3. Tenant Interceptor for Services

```java
@Component
public class TenantInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        String tenantId = request.getHeader("X-Tenant-ID");
        
        if (tenantId == null || tenantId.isBlank()) {
            response.setStatus(HttpStatus.BAD_REQUEST.value());
            return false;
        }
        
        // Validate tenant exists and is active
        if (!tenantService.isValidTenant(tenantId)) {
            response.setStatus(HttpStatus.FORBIDDEN.value());
            return false;
        }
        
        TenantContext.setTenantId(tenantId);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler, Exception ex) {
        TenantContext.clear(); // Prevent tenant leakage
    }
}
```

## Tenant Provisioning

### Provisioning Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ New Tenant  │────▶│   Create    │────▶│   Create    │────▶│  Initialize │
│  Request    │     │   Entry     │     │  Database   │     │    Data     │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                          │                   │                    │
                          ▼                   ▼                    ▼
                    ┌───────────┐       ┌───────────┐        ┌───────────┐
                    │ tenants   │       │ ats_      │        │ seed      │
                    │ collection│       │ tenant_X  │        │ master    │
                    │ in system │       │ database  │        │ data      │
                    └───────────┘       └───────────┘        └───────────┘
```

### Tenant Registry Schema

```javascript
// ats_system.tenants collection
{
  "_id": ObjectId("..."),
  "tenantId": "tenant-uuid-001",
  "name": "Acme Corporation",
  "subdomain": "acme",
  "status": "ACTIVE", // ACTIVE, SUSPENDED, PROVISIONING, DEACTIVATED
  "plan": "ENTERPRISE", // FREE, STARTER, PROFESSIONAL, ENTERPRISE
  "settings": {
    "maxUsers": 100,
    "maxJobs": 500,
    "features": ["ai_matching", "video_interviews", "sso"],
    "customDomain": "careers.acme.com"
  },
  "billing": {
    "customerId": "stripe_cus_xxx",
    "subscriptionId": "stripe_sub_xxx"
  },
  "metadata": {
    "createdAt": ISODate("2025-01-15T10:00:00Z"),
    "createdBy": "admin-user-id",
    "lastActiveAt": ISODate("2025-01-27T08:30:00Z")
  }
}
```

## Data Isolation Guarantees

### Query-Level Enforcement

```java
@Repository
public class CandidateRepository {

    @Autowired
    private MongoTemplate mongoTemplate;

    public List<Candidate> findAll() {
        // Database is already scoped to tenant via TenantContext
        // Additional safety: verify tenant in document matches context
        Query query = new Query();
        
        // Defense in depth - filter even within tenant database
        String tenantId = TenantContext.getTenantId();
        query.addCriteria(Criteria.where("tenantId").is(tenantId));
        
        return mongoTemplate.find(query, Candidate.class);
    }
}
```

### Index Strategy

```javascript
// Each tenant database has identical indexes
db.candidates.createIndex({ "email": 1 }, { unique: true });
db.candidates.createIndex({ "skills": 1 });
db.candidates.createIndex({ "createdAt": -1 });
db.candidates.createIndex({ 
  "firstName": "text", 
  "lastName": "text", 
  "skills": "text" 
});

// Jobs collection
db.jobs.createIndex({ "status": 1, "createdAt": -1 });
db.jobs.createIndex({ "department": 1 });
```

## Tenant Lifecycle

| State | Description | Data Access |
|-------|-------------|-------------|
| `PROVISIONING` | Database being created | None |
| `ACTIVE` | Normal operation | Full |
| `SUSPENDED` | Payment issue / Policy violation | Read-only |
| `DEACTIVATED` | Tenant requested deletion | None (archived) |
| `DELETED` | After retention period | Purged |

## File Storage Strategy

Tenant files (resumes, documents) are isolated using prefixes:

### Azure Blob Storage
```
Container: ats-documents
├── tenant-001/
│   ├── resumes/
│   ├── offer-letters/
│   └── profile-images/
├── tenant-002/
│   ├── resumes/
│   └── ...
```

### AWS S3
```
Bucket: ats-documents
├── tenant-001/
│   ├── resumes/
│   ├── offer-letters/
│   └── profile-images/
├── tenant-002/
│   ├── resumes/
│   └── ...
```

Pre-signed URLs (Azure SAS / AWS Presigned) ensure secure, time-limited access to files.

## Compliance & Audit

- All tenant data operations logged to `ats_system.audit_logs`
- Kafka events include `tenantId` for cross-service tracing
- Data export available per tenant for GDPR requests
- Configurable retention policies per tenant

## Next Steps

- [Cloud Service Mapping](04-cloud-service-mapping.md)
