# High-Level System Architecture

The platform follows a layered architecture with clear separation between the central administration layer and tenant-scoped operations. Domain-based tenant resolution routes each request into the correct context, where all downstream services (database queries, cache tags, filesystem paths, and queue prefixes) operate within tenant boundaries. The modular design allows independent feature modules to be composed per-tenant.

## Key Architectural Decisions

- **Inertia.js + React** for the frontend provides SPA-like UX without a separate API for the admin panel
- **Repository Pattern** with interface bindings (singleton) ensures testability and consistent data access
- **Model Observers** handle cache invalidation automatically when data changes
- **11 Independent Feature Modules** (nWidart/Laravel-Modules) allow per-tenant feature composition
- **Dual API layers**: First-party API (CORS-restricted, for tenant frontends) and Third-party API (token-authenticated, rate-limited)

## Diagram

```mermaid
graph TB
    subgraph Clients["Client Layer"]
        Browser["Browser / SPA"]
        MobileApp["Mobile App"]
        ThirdParty["3rd-Party API Consumers"]
    end

    subgraph LoadBalancer["Infrastructure"]
        LB["Reverse Proxy / Load Balancer"]
    end

    subgraph AppLayer["Application Layer (Laravel)"]
        direction TB
        subgraph Middleware["Middleware Pipeline"]
            CORS["CORS"]
            TenantInit["Tenant Resolution\n(Domain/Subdomain)"]
            Auth["Authentication\n(Sanctum / Session)"]
            RateLimit["Rate Limiting"]
            PermCache["Permission Cache Key"]
        end

        subgraph CentralApp["Central Context"]
            AdminPanel["Admin Panel\n(Inertia.js / React)"]
            CentralAPI["Central API"]
            AccountScope["Account Scope\n(Row-Level Isolation)"]
            SuperAdmin["Super Admin\nBypass"]
        end

        subgraph TenantApp["Tenant Context"]
            TenantPanel["Tenant Admin Panel\n(Inertia.js / React)"]
            TenantAPI["Tenant API (v1)"]
            TenantScope["Tenant Scope\n(Row-Level Isolation)"]
        end

        subgraph Services["Service Layer"]
            RepoPattern["Repository Pattern\n(Interface -> Impl -> Singleton)"]
            BizServices["Business Services"]
            Observers["Model Observers\n(Cache Invalidation)"]
        end

        subgraph ContentModules["Feature Modules"]
            Mod1["Content Module A"]
            Mod2["Content Module B"]
            Mod3["Content Module C"]
            ModN["Content Module N\n(11 Independent Modules)"]
        end
    end

    subgraph DataLayer["Data & Infrastructure Layer"]
        MySQL[("MySQL\n(Shared Database)")]
        Redis[("Redis\n(Cache + Sessions + Queues)")]
        FileStore["File Storage\n(Tenant-Isolated Disks)"]
        Queue["Queue Workers\n(Horizon)"]
    end

    subgraph ExternalServices["External Services"]
        Stripe["Payment Gateway"]
        MailService["Email Service"]
        CDN["CDN / Asset Delivery"]
        ExternalAPIs["External Data APIs"]
    end

    Browser --> LB
    MobileApp --> LB
    ThirdParty --> LB
    LB --> Middleware

    CORS --> TenantInit
    TenantInit --> Auth
    Auth --> RateLimit
    RateLimit --> PermCache

    PermCache --> CentralApp
    PermCache --> TenantApp

    CentralApp --> Services
    TenantApp --> Services
    Services --> ContentModules

    Services --> MySQL
    Services --> Redis
    Services --> FileStore
    Services --> Queue

    Queue --> MySQL
    Queue --> ExternalAPIs
    Queue --> MailService

    CentralApp --> Stripe
    TenantApp --> CDN
```

## Component Responsibilities

| Component | Responsibility |
|-----------|---------------|
| **Reverse Proxy** | SSL termination, load balancing, domain routing |
| **Middleware Pipeline** | CORS, tenant resolution, auth, rate limiting, permission cache setup |
| **Central Context** | Super-admin operations, account management, tenant provisioning |
| **Tenant Context** | Tenant-scoped admin panel, content management, settings |
| **Repository Layer** | Data access abstraction via interface -> implementation -> singleton binding |
| **Feature Modules** | Self-contained modules with own models, controllers, migrations, routes |
| **Redis** | Cache (tag-based, tenant-prefixed), sessions, queue backend |
| **Horizon** | Queue monitoring and management with tenant-aware job dispatching |
