# Enterprise Multi-Tenant SaaS Platform

A large-scale **enterprise multi-tenant SaaS platform** built for content management and publishing workflows.  
This repository documents the **architecture, system design, and technical decisions** of a real-world production system.

> ⚠️ **Confidentiality Notice**  
> Source code, internal documentation, and screenshots are intentionally excluded to respect company confidentiality.  
> This repository focuses on architecture, scalability strategies, and engineering practices.

---

## Project Context

- This project was **already in development** when I joined.
- I entered during the **mid-phase of development**.
- Over time, I took ownership of core modules and later **led the project as a Team Lead**.
- My role involved **stabilizing the system, scaling it, and guiding architectural decisions**.

---

## My Role & Responsibilities

**Team Lead / Full-Stack Developer**

- Led backend and frontend development efforts
- Designed and improved **multi-tenant architecture**
- Implemented **tenant isolation strategies**
- Optimized performance for high-traffic scenarios
- Reviewed code, mentored developers, and guided best practices
- Collaborated with product, QA, and DevOps teams
- Ensured security, scalability, and maintainability

---

## System Overview

The platform supports multiple organizations (tenants) on a shared infrastructure while ensuring strict data isolation and scalability.

**Key Characteristics:**
- Enterprise-grade SaaS
- Multi-tenant architecture
- Role-based access control
- Subscription-based business model
- High availability and performance focus

---

## Architecture Highlights

- **Tenant Identification**
  - Domain-based and/or request-based tenant resolution
- **Data Isolation**
  - Tenant-aware queries and scoped access
- **Authentication & Authorization**
  - Centralized auth with tenant context
- **Scalability**
  - Designed to support thousands of tenants
- **Security**
  - Strong access boundaries between tenants
 
  > Diagrams reflect the real production system. Source code is confidential.

---

### System Architecture
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

### Shared Database Multi-Tenancy Model
```mermaid
graph TB
    subgraph SharedDB["Single Shared MySQL Database"]
        direction TB

        subgraph GlobalTables["Global Tables (No tenant_id)"]
            Countries["countries"]
            Themes["themes"]
            Components["component_types"]
            StripeProducts["stripe_products"]
            Permissions["permissions"]
        end

        subgraph TenantScoped["Tenant-Scoped Tables (tenant_id FK)"]
            Users["users"]
            Content["content_items"]
            Categories["categories"]
            MediaAssets["media_assets"]
            Pages["pages"]
            SEO["seo_settings"]
            Settings["general_settings"]
            AdCampaigns["ad_campaigns"]
            Subscribers["subscribers"]
        end

        subgraph AccountScoped["Account-Scoped Tables (account_id FK)"]
            Tenants["tenants"]
            Roles["roles"]
            AccountUsers["users (central)"]
        end

        subgraph PivotTables["Pivot / Junction Tables"]
            ModelRoles["model_has_roles"]
            ModelPerms["model_has_permissions"]
            RolePerms["role_has_permissions"]
            TenantConfig["tenant_configurations"]
        end
    end

    subgraph IsolationLayer["Row-Level Isolation Mechanism"]
        direction LR
        TenantTrait["TenantScope Trait"]
        AccountTrait["AccountScope Trait"]

        TenantTrait -->|"Auto-applies WHERE tenant_id = ?"| TenantScoped
        AccountTrait -->|"Auto-applies WHERE account_id = ?"| AccountScoped
    end

    subgraph QueryLifecycle["Automatic Query Scoping"]
        SELECT["SELECT: Global scope filter"]
        CREATE["CREATE: Auto-set tenant_id"]
        UPDATE["UPDATE: Ownership verification"]
        DELETE["DELETE: Ownership verification + logging"]
    end

    TenantTrait --> QueryLifecycle
    AccountTrait --> QueryLifecycle

    subgraph Bypass["Bypass Conditions"]
        SABypass["Super Admin: bypasses AccountScope"]
        TenantCtx["Tenant Context: AccountScope defers to TenantScope"]
        NoAuth["Unauthenticated: scopes skip (public routes)"]
    end

    AccountTrait -.-> Bypass
```

📁 **More diagrams:** [Request Lifecycle](./docs/request-lifecycle.md) · [Auth Flow](./docs/auth-flow.md) · [Database Schema](./docs/database-schema.md) · [Modular Architecture](./docs/modular-architecture.md)

## Tech Stack

### Backend
- Laravel
- PHP
- MySQL
- REST APIs
- Queue & background job processing

### Frontend
- React / Next.js
- Component-driven architecture
- API-driven UI

### Infrastructure & DevOps
- Docker
- Docker Compose
- CI/CD pipelines
- Cloud-based deployment (AWS / similar)

---

## Key Engineering Contributions

- Improved multi-tenant request lifecycle
- Refactored legacy modules for maintainability
- Introduced performance optimizations
- Defined coding standards and review processes
- Assisted in production debugging and incident resolution

---

## Why This Repository Exists

Many impactful engineering projects cannot be open-sourced.  
This repository exists to demonstrate:

- Real-world **SaaS architecture experience**
- Leadership in an **enterprise environment**
- Ability to work on **large, long-running systems**
- Strong understanding of **scalability, security, and system design**

📁 Detailed technical documentation is available here: [/docs](./docs)
---

## Related Public Links

- 🌐 Product Website (Public Frontend): https://www.ipublisher.app
- 👤 Author: Ali Hamza  
- 💼 Role: Team Lead / Full-Stack Developer

---

## License

No license is provided, as this repository does not contain distributable source code.

---

## Contact

If you’re a recruiter or hiring manager and would like to discuss the architecture, challenges, or decisions in more detail, feel free to reach out via:

- GitHub: https://github.com/alihamzahq
- Website: https://alihamza.dev
