# Database Schema (Anonymized ERD)

This ERD shows the core tables and their relationships in the shared database. Tables with `tenant_id` are tenant-scoped (row-level isolation via TenantScope trait), tables with `account_id` are account-scoped (central administration isolation via AccountScope trait), and tables without either are global/shared across all tenants. The schema supports multi-tenant content management, role-based access control, subscription billing, ad campaign management, and modular feature extensions.

## Table Classification

- **Global Tables**: `permissions`, `countries`, `themes`, `component_types`, `stripe_products` -- shared reference data, no tenant/account scoping
- **Account-Scoped Tables**: `tenants`, `roles`, `users (central)` -- isolated by `account_id` for central administration
- **Tenant-Scoped Tables**: `content_items`, `categories`, `media_assets`, `pages`, `authors`, `seo_settings`, `general_settings`, `ad_campaigns`, `ad_banners`, `subscriptions`, `feed_sources`, `subscribers`, `interactive_content`, `queue_logs` -- isolated by `tenant_id`

## Diagram

```mermaid
erDiagram
    accounts {
        bigint id PK
        string name
        timestamps created_at
    }

    tenants {
        string id PK
        bigint user_id FK
        bigint account_id FK
        string name
        string email
        boolean is_active
        json data
        string stripe_customer_id
        string subscription_status
    }

    custom_domains {
        bigint id PK
        string tenant_id FK
        string domain
        string frontend_domain
    }

    users {
        bigint id PK
        bigint tenant_id FK
        bigint account_id FK
        string name
        string email
        string password
        timestamp email_verified_at
        softdeletes deleted_at
    }

    roles {
        bigint id PK
        bigint account_id FK
        string name
        string guard_name
        softdeletes deleted_at
    }

    permissions {
        bigint id PK
        string name
        string guard_name
        string group
        softdeletes deleted_at
    }

    model_has_roles {
        bigint role_id FK
        bigint model_id FK
        string model_type
    }

    role_has_permissions {
        bigint role_id FK
        bigint permission_id FK
    }

    content_items {
        bigint id PK
        bigint tenant_id FK
        bigint author_id FK
        bigint category_id FK
        string title
        string slug
        text body
        string status
        boolean featured
        timestamp published_at
        softdeletes deleted_at
    }

    categories {
        bigint id PK
        bigint tenant_id FK
        bigint parent_id FK
        string name
        string slug
        integer sort_order
    }

    media_assets {
        bigint id PK
        bigint tenant_id FK
        string filename
        string path
        string mime_type
        string credits
    }

    pages {
        bigint id PK
        bigint tenant_id FK
        string title
        string slug
        string type
        text body
        softdeletes deleted_at
    }

    authors {
        bigint id PK
        bigint tenant_id FK
        bigint user_id FK
        string name
        string bio
        string avatar
    }

    seo_settings {
        bigint id PK
        bigint tenant_id FK
        string page_type
        string meta_title
        string meta_description
        text robots_txt
    }

    general_settings {
        bigint id PK
        bigint tenant_id FK
        string key
        text value
    }

    ad_campaigns {
        bigint id PK
        bigint tenant_id FK
        string name
        string status
        timestamp start_date
        timestamp end_date
    }

    ad_banners {
        bigint id PK
        bigint tenant_id FK
        bigint campaign_id FK
        string type
        string placement
        integer priority
        integer weight
        boolean active
    }

    subscriptions {
        bigint id PK
        bigint tenant_id FK
        string stripe_id
        string plan
        string status
        timestamp trial_ends_at
    }

    feed_sources {
        bigint id PK
        bigint tenant_id FK
        string name
        string url
        string type
        boolean active
    }

    interactive_content {
        bigint id PK
        bigint tenant_id FK
        string title
        string type
        boolean active
    }

    queue_logs {
        bigint id PK
        bigint tenant_id FK
        string job_name
        string status
        text payload
    }

    accounts ||--o{ tenants : "owns"
    accounts ||--o{ users : "belongs to"
    accounts ||--o{ roles : "scoped to"
    tenants ||--o{ custom_domains : "has"
    tenants ||--o{ users : "scoped"
    tenants ||--o{ content_items : "scoped"
    tenants ||--o{ categories : "scoped"
    tenants ||--o{ media_assets : "scoped"
    tenants ||--o{ pages : "scoped"
    tenants ||--o{ authors : "scoped"
    tenants ||--o{ seo_settings : "scoped"
    tenants ||--o{ general_settings : "scoped"
    tenants ||--o{ ad_campaigns : "scoped"
    tenants ||--o{ subscriptions : "has"
    tenants ||--o{ feed_sources : "scoped"
    tenants ||--o{ interactive_content : "scoped"
    tenants ||--o{ queue_logs : "scoped"
    users ||--o{ model_has_roles : "assigned"
    roles ||--o{ model_has_roles : "assigned to"
    roles ||--o{ role_has_permissions : "granted"
    permissions ||--o{ role_has_permissions : "assigned"
    authors ||--|| users : "linked to"
    content_items }o--|| categories : "belongs to"
    content_items }o--|| authors : "written by"
    ad_banners }o--|| ad_campaigns : "belongs to"
    categories ||--o{ categories : "parent-child"
```

## Key Schema Patterns

| Pattern | Implementation |
|---------|---------------|
| **Row-level isolation** | `tenant_id` FK on all tenant-scoped tables, enforced by TenantScope trait |
| **Account isolation** | `account_id` FK on central tables, enforced by AccountScope trait |
| **Soft deletes** | Used on `users`, `content_items`, `pages`, `roles`, `permissions` |
| **Polymorphic roles** | `model_has_roles.model_type` allows roles on any model type |
| **Self-referencing** | `categories.parent_id` for nested category hierarchies |
| **Priority + weight** | `ad_banners` uses priority/weight columns for weighted random rotation |
| **JSON data** | `tenants.data` stores tenant-specific configuration as JSON |
