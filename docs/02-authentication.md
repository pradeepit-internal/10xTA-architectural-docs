# Authentication & Authorization

## Overview

The ATS platform implements a dual authentication strategy to support both individual recruiters (B2C) and enterprise clients (B2B). The platform supports either **Azure External ID (B2C)** or **AWS Cognito** as the identity provider.

## Identity Provider Comparison

| Feature | Azure External ID | AWS Cognito |
|---------|-------------------|-------------|
| Social Login | ✅ Native (LinkedIn, Google, Microsoft) | ✅ Native (Google, Facebook, Amazon) |
| LinkedIn Login | ✅ Built-in | ⚠️ Custom OIDC setup required |
| Enterprise SSO (SAML) | ✅ Built-in federation | ✅ Via identity pools |
| Custom Policies | ✅ Advanced (Identity Experience Framework) | ⚠️ Limited (Lambda triggers) |
| MFA | ✅ Built-in | ✅ Built-in |
| User Management UI | ✅ Azure Portal | ✅ AWS Console |
| Pricing | Per MAU | Per MAU |

**Recommendation**: Azure External ID is preferred for enterprise SSO scenarios; Cognito works well for simpler authentication needs.

## Authentication Flows

### 1. Social Login (Individual Users)

```
┌─────────┐     ┌──────────────┐     ┌─────────────────┐     ┌─────────────┐
│  User   │────▶│   Identity   │────▶│ Social Provider │────▶│   Gateway   │
│         │◀────│   Provider   │◀────│ (LinkedIn/Google)│◀────│             │
└─────────┘     └──────────────┘     └─────────────────┘     └─────────────┘
                      │
                      ▼
               ┌──────────────┐
               │  JWT Token   │
               │  (id_token)  │
               └──────────────┘
```

**Supported Social Providers:**
- LinkedIn (primary for recruiters)
- Google
- Microsoft Personal
- Email/Password (fallback)

### 2. Enterprise SSO (Organizational Clients)

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Employee   │────▶│   Identity   │────▶│ Corporate IdP   │
│             │◀────│   Provider   │◀────│ (SAML/OIDC)     │
└─────────────┘     │  Federation  │     └─────────────────┘
                    └──────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Federated    │
                    │ JWT Token    │
                    └──────────────┘
```

**Supported Protocols:**
- SAML 2.0
- OpenID Connect (OIDC)
- WS-Federation

## JWT Token Structure

```json
{
  "iss": "https://[identity-provider-url]",
  "sub": "user-uuid-here",
  "aud": "ats-platform-client-id",
  "exp": 1704067200,
  "iat": 1704063600,
  "auth_time": 1704063600,
  "idp": "linkedin.com",
  "tenantId": "tenant-uuid",
  "roles": ["RECRUITER", "HIRING_MANAGER"],
  "permissions": ["candidate:read", "candidate:write", "job:manage"],
  "name": "John Doe",
  "email": "john.doe@company.com"
}
```

## Gateway Authentication Flow

### Spring Cloud Gateway Security Configuration

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthConverter())
                )
            )
            .authorizeExchange(exchanges -> exchanges
                .pathMatchers("/public/**", "/health/**").permitAll()
                .pathMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .pathMatchers("/api/v1/**").authenticated()
            )
            .build();
    }
}
```

### Tenant Context Injection

```java
@Component
public class TenantContextFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication())
            .cast(JwtAuthenticationToken.class)
            .flatMap(auth -> {
                String tenantId = extractTenantId(auth.getToken());
                String userId = auth.getToken().getSubject();
                List<String> roles = extractRoles(auth.getToken());
                
                ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
                    .header("X-Tenant-ID", tenantId)
                    .header("X-User-ID", userId)
                    .header("X-User-Roles", String.join(",", roles))
                    .build();
                    
                return chain.filter(exchange.mutate().request(mutatedRequest).build());
            });
    }
}
```

## Authorization Model

### Role-Based Access Control (RBAC)

| Role | Description | Permissions |
|------|-------------|-------------|
| `SUPER_ADMIN` | Platform administrator | Full system access |
| `TENANT_ADMIN` | Tenant administrator | Tenant configuration, user management |
| `HIRING_MANAGER` | Hiring team lead | Job management, offer approvals |
| `RECRUITER` | Recruitment specialist | Candidate management, interviews |
| `INTERVIEWER` | Interview participant | View candidates, submit feedback |
| `VIEWER` | Read-only access | View jobs and candidates |

### Permission Matrix

```
┌─────────────────┬───────┬───────┬───────┬───────┬───────┬───────┐
│ Permission      │ SUPER │ ADMIN │ HM    │ RECR  │ INTV  │ VIEW  │
├─────────────────┼───────┼───────┼───────┼───────┼───────┼───────┤
│ candidate:read  │   ✓   │   ✓   │   ✓   │   ✓   │   ✓   │   ✓   │
│ candidate:write │   ✓   │   ✓   │   ✓   │   ✓   │   ✗   │   ✗   │
│ job:read        │   ✓   │   ✓   │   ✓   │   ✓   │   ✓   │   ✓   │
│ job:manage      │   ✓   │   ✓   │   ✓   │   ✗   │   ✗   │   ✗   │
│ offer:approve   │   ✓   │   ✓   │   ✓   │   ✗   │   ✗   │   ✗   │
│ tenant:manage   │   ✓   │   ✓   │   ✗   │   ✗   │   ✗   │   ✗   │
│ system:admin    │   ✓   │   ✗   │   ✗   │   ✗   │   ✗   │   ✗   │
└─────────────────┴───────┴───────┴───────┴───────┴───────┴───────┘
```

## Configuration Examples

### Azure External ID Configuration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://${AZURE_B2C_TENANT}.b2clogin.com/${AZURE_B2C_TENANT}.onmicrosoft.com/${AZURE_B2C_POLICY}/v2.0/
          jwk-set-uri: https://${AZURE_B2C_TENANT}.b2clogin.com/${AZURE_B2C_TENANT}.onmicrosoft.com/${AZURE_B2C_POLICY}/discovery/v2.0/keys
          audiences: ${AZURE_B2C_CLIENT_ID}
```

### AWS Cognito Configuration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://cognito-idp.${AWS_REGION}.amazonaws.com/${AWS_COGNITO_USER_POOL_ID}
          jwk-set-uri: https://cognito-idp.${AWS_REGION}.amazonaws.com/${AWS_COGNITO_USER_POOL_ID}/.well-known/jwks.json
          audiences: ${AWS_COGNITO_CLIENT_ID}
```

## Security Best Practices

### Token Validation
- Validate issuer (`iss`) against known identity provider
- Verify audience (`aud`) matches application client ID
- Check token expiration (`exp`)
- Validate signature using JWKS endpoint

### Tenant Isolation
- Tenant ID extracted from JWT claims
- All downstream requests include `X-Tenant-ID` header
- Services validate tenant context before data access

### Rate Limiting
- Per-user: 100 requests/minute
- Per-tenant: 10,000 requests/minute
- Burst allowance: 2x normal rate for 10 seconds

## Next Steps

- [Multi-Tenancy Strategy](03-multi-tenancy.md)
- [Cloud Service Mapping](04-cloud-service-mapping.md)
