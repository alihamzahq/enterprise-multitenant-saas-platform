# Request Lifecycle / Tenant Resolution Flow

Every incoming request passes through a middleware pipeline that resolves the tenant from the domain/subdomain, initializes the tenant context (scoping database queries, cache tags, filesystem paths, and queue prefixes), verifies the tenant is active, and then hands off to the authentication and authorization layer. The tenant context propagates automatically to all downstream services via Laravel's service container.

## Key Design Decisions

- **Domain-based resolution** with 30-day Redis caching eliminates repeated DB lookups
- **Tenancy bootstrappers** scope 4 subsystems simultaneously: database, cache, filesystem, queues
- **Permission cache keys** are tenant-specific to prevent cross-tenant permission leakage
- **Tenant resolution middleware** is given highest priority in the middleware stack

## Diagram

```mermaid
sequenceDiagram
    participant Client as Client (Browser/API)
    participant Proxy as Reverse Proxy
    participant MW as Middleware Pipeline
    participant TR as Tenant Resolver
    participant Cache as Resolver Cache (Redis)
    participant Boot as Tenancy Bootstrappers
    participant Auth as Auth Middleware
    participant Perm as Permission System
    participant Ctrl as Controller
    participant Repo as Repository Layer
    participant Scope as TenantScope Trait
    participant DB as MySQL (Shared DB)
    participant RCache as Redis Cache

    Client->>Proxy: HTTP Request (tenant.example.com)
    Proxy->>MW: Forward Request

    Note over MW: Global Middleware:<br/>CORS, TrimStrings,<br/>ValidatePostSize

    MW->>TR: InitializeTenancyByDomainOrSubdomain
    TR->>Cache: Lookup domain -> tenant mapping
    alt Cache Hit
        Cache-->>TR: Return cached tenant (TTL: 30 days)
    else Cache Miss
        TR->>DB: SELECT * FROM domains WHERE domain = ?
        DB-->>TR: Tenant record
        TR->>Cache: Store mapping
    end

    TR->>Boot: Initialize Tenancy
    Note over Boot: Bootstrappers execute:<br/>1. Database connection scoping<br/>2. Cache tag prefixing<br/>3. Filesystem path isolation<br/>4. Queue tenant context

    Boot->>MW: CheckTenantForMaintenanceMode
    MW->>MW: CheckTenantActive

    alt Tenant Inactive
        MW-->>Client: 503 Tenant Inactive View
    end

    MW->>Auth: Session / Sanctum Auth
    Auth->>Perm: SetPermissionCacheKey
    Note over Perm: Cache key:<br/>spatie.permission.cache<br/>.tenant.{tenant_id}

    Perm->>Ctrl: Route to Controller
    Ctrl->>Repo: Call Repository Method
    Repo->>Scope: Eloquent Query (auto-scoped)
    Note over Scope: Global Scope adds:<br/>WHERE tenant_id = ?
    Scope->>DB: Scoped SQL Query
    DB-->>Repo: Tenant-filtered Results
    Repo->>RCache: Cache with tenant tag
    Repo-->>Ctrl: Return Data
    Ctrl-->>Client: JSON / Inertia Response
```

## Middleware Pipeline Order

| Order | Middleware | Purpose |
|-------|-----------|---------|
| 1 | `TrustProxies` | Handle proxy headers |
| 2 | `HandleCors` | CORS headers |
| 3 | `PreventRequestsDuringMaintenance` | Maintenance mode check |
| 4 | `ValidatePostSize` | Request size limits |
| 5 | `TrimStrings` | Input sanitization |
| 6 | **`InitializeTenancyByDomainOrSubdomain`** | **Tenant resolution (highest priority)** |
| 7 | `PreventAccessFromCentralDomains` | Block central domain in tenant routes |
| 8 | `CheckTenantForMaintenanceMode` | Per-tenant maintenance mode |
| 9 | `CheckTenantActive` | Verify tenant is enabled |
| 10 | `EncryptCookies` / `StartSession` | Session management |
| 11 | `VerifyCsrfToken` | CSRF protection |
| 12 | `HandleInertiaRequests` | Inertia.js shared data |
| 13 | `SetPermissionCacheKey` | Tenant-aware permission caching |
| 14 | `auth` | Authentication check |

## Bootstrapped Subsystems

When tenancy is initialized, these bootstrappers configure the application for the current tenant:

1. **DatabaseTenancyBootstrapper** - Switches the database connection to include tenant context
2. **CacheTenancyBootstrapper** - Prefixes all cache keys with `tenant_{id}` tag
3. **FilesystemTenancyBootstrapper** - Redirects storage paths to tenant-specific directories
4. **QueueTenancyBootstrapper** - Ensures queued jobs retain tenant context when processed
