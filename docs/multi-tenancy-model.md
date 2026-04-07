# Shared Database Multi-Tenancy Model

The platform uses a single shared MySQL database with row-level tenant isolation. Two complementary Eloquent scope traits enforce data boundaries: **TenantScope** (for tenant-context operations using `tenant_id`) and **AccountScope** (for central-context operations using `account_id`). Global scopes are automatically applied on all CRUD operations, with security logging for unauthorized cross-tenant access attempts.

## How It Works

1. **TenantScope Trait** - Applied to models that need tenant isolation
   - `SELECT`: Adds `WHERE tenant_id = ?` via a global scope
   - `CREATE`: Auto-sets `tenant_id` to the current tenant
   - `UPDATE/DELETE`: Verifies ownership, aborts with 403 + logs if mismatched

2. **AccountScope Trait** - Applied to models in the central context
   - Same CRUD protections but scoped by `account_id`
   - Bypassed when in tenant context (TenantScope takes priority)
   - Super-admin role bypasses all account filtering

3. **Column Check Caching** - Both traits cache `Schema::hasColumn()` results per-request to avoid repeated information_schema queries

## Diagram

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

## Isolation Guarantees

| Operation | TenantScope Behavior | AccountScope Behavior |
|-----------|---------------------|----------------------|
| **SELECT** | Adds `WHERE tenant_id = ?` | Adds `WHERE account_id = ?` |
| **CREATE** | Auto-assigns `tenant_id` | Auto-assigns `account_id` |
| **UPDATE** | Verifies `tenant_id` matches, 403 if not | Verifies `account_id` matches, 403 if not |
| **DELETE** | Verifies `tenant_id` matches, logs + 403 | Verifies `account_id` matches, logs + 403 |
| **Super Admin** | N/A (always in tenant context) | Bypasses all filtering |
| **No Auth** | Skips (public routes) | Skips (public routes) |

## Security Measures

- **Unauthorized access logging**: All cross-tenant access attempts are logged with model class, model ID, and both the attacker's and owner's tenant IDs
- **Immediate abort**: Unauthorized access results in an immediate 403 response
- **No lazy loading escape**: Global scopes apply to all query methods including relationships
- **Column cache**: `Schema::hasColumn()` results are cached per-table per-request to prevent performance degradation
