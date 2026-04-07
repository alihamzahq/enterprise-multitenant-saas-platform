# Modular Architecture (Feature Modules)

The platform uses nWidart/Laravel-Modules to organize domain-specific features into 11 fully self-contained modules. Each module is a mini-Laravel application with its own Models, Controllers, Repositories, Services, Console Commands, Jobs, Migrations, Routes, and even its own dedicated database connection. Modules are hot-pluggable via a JSON activation file -- tenants can be configured to access specific modules, enforced by middleware. The core app provides shared infrastructure (tenancy, auth, caching) while modules handle domain-specific logic independently.

## Why Modular Architecture?

- **Separation of Concerns**: Each domain feature is fully isolated with its own MVC stack
- **Independent Databases**: Each module connects to its own dedicated database, preventing data coupling
- **Hot-Pluggable**: Modules can be enabled/disabled at runtime via `modules_statuses.json`
- **Per-Tenant Access Control**: Middleware enforces which tenants can access which modules
- **Independent Scaling**: Modules can be developed, tested, and deployed independently
- **Scheduled Autonomy**: Each module registers its own cron-based tasks (data fetching, sync, etc.)

## Architecture Diagram

```mermaid
graph TB
    subgraph CoreApp["Core Application (Laravel)"]
        direction TB
        Kernel["HTTP Kernel"]
        TenantRes["Tenant Resolution"]
        AuthLayer["Auth & Permissions"]
        ServiceContainer["Service Container"]
        Scheduler["Task Scheduler"]
        QueueManager["Queue Manager (Horizon)"]
    end

    subgraph ModuleSystem["Module System (nWidart/Laravel-Modules)"]
        direction TB
        ModuleLoader["Module Loader"]
        ActivationFile["modules_statuses.json\n(Enable/Disable Modules)"]
        ModuleConfig["config/modules.php"]

        ActivationFile --> ModuleLoader
        ModuleConfig --> ModuleLoader
    end

    subgraph ModuleStructure["Module Internal Structure (per module)"]
        direction TB

        subgraph Providers["Providers"]
            MainSP["ModuleServiceProvider\n(Boot + Register)"]
            RouteSP["RouteServiceProvider\n(Web + API Routes)"]
        end

        subgraph HTTPLayer["HTTP Layer"]
            Controllers["Controllers\n(REST API Endpoints)"]
            ModMiddleware["Custom Middleware\n(Token Auth, etc.)"]
            Resources["API Resources\n(Response Transformers)"]
        end

        subgraph DataLayer["Data Layer"]
            Models["Models\n(Own DB Connection)"]
            Repos["Repositories\n(Data Access)"]
            Migrations["Migrations\n(Module-Specific Schema)"]
            Seeders["Seeders"]
        end

        subgraph BusinessLogic["Business Logic"]
            Services["Services"]
            Traits["Traits\n(Reusable Logic)"]
            Constants["Constants"]
        end

        subgraph AsyncLayer["Async Processing"]
            Commands["Console Commands\n(Data Fetching, Sync)"]
            Jobs["Queued Jobs\n(Background Processing)"]
            Schedule["Scheduled Tasks\n(Cron-Based)"]
        end

        subgraph Assets["Frontend"]
            Views["Blade Views"]
            JS["JavaScript"]
            Config["Module Config"]
        end
    end

    subgraph Databases["Database Connections"]
        CoreDB[("Core MySQL\n(Shared Tenant DB)")]
        ModDB_A[("Module A DB")]
        ModDB_B[("Module B DB")]
        ModDB_C[("Module C DB")]
        ModDB_N[("Module N DB\n(11 Dedicated DBs)")]
    end

    subgraph ActiveModules["Registered Modules (11)"]
        ModA["Module A"]
        ModB["Module B"]
        ModC["Module C"]
        ModD["Module D"]
        ModE["Module E"]
        ModF["Module F"]
        ModG["Module G"]
        ModH["Module H"]
        ModI["Module I"]
        ModJ["Module J"]
        ModK["Module K"]
    end

    subgraph TenantAccess["Per-Tenant Module Access"]
        CheckSport["CheckTenantModule\nMiddleware"]
        TenantModules["tenant_modules\n(Pivot Table)"]
        APIScopes["API Scope Guard\n(module:read, module:write)"]
    end

    CoreApp --> ModuleSystem
    ModuleLoader --> ActiveModules

    ActiveModules --> ModuleStructure

    MainSP --> ServiceContainer
    RouteSP --> Kernel
    Commands --> Scheduler
    Jobs --> QueueManager
    Schedule --> Scheduler

    Models --> ModDB_A
    Models --> ModDB_B
    Models --> ModDB_C
    Models --> ModDB_N

    Controllers --> TenantAccess
    CheckSport --> TenantModules
    TenantAccess --> AuthLayer

    CoreApp --> CoreDB
```

## Module Lifecycle

This sequence diagram shows how modules are discovered, loaded, and registered during application boot.

```mermaid
sequenceDiagram
    participant Boot as Application Boot
    participant Loader as Module Loader
    participant JSON as modules_statuses.json
    participant SP as Module ServiceProvider
    participant Container as Service Container
    participant Router as Router
    participant Scheduler as Task Scheduler
    participant DB as Module Database

    Boot->>Loader: Load Module System
    Loader->>JSON: Read activation statuses
    JSON-->>Loader: {Module A: true, Module B: true, ...}

    loop For Each Active Module
        Loader->>SP: Register ServiceProvider
        SP->>Container: Bind services (singleton)
        SP->>Router: Register API + Web routes
        SP->>Scheduler: Register scheduled commands
        SP->>DB: Merge database connection config
    end

    Note over Container: All modules now registered.<br/>Each has own DB connection,<br/>routes, commands, and services.

    Container-->>Boot: Application Ready
```

## Module Internal Composition (Typical Module)

Each module follows a consistent structure that mirrors a full Laravel application:

| Component | Count (Typical) | Purpose |
|-----------|-----------------|---------|
| **Models** | 30-60 | Domain entities with dedicated DB connection |
| **Controllers** | 10-15 | REST API endpoints |
| **Repositories** | 5-10 | Data access abstraction |
| **Services** | 2-5 | Business logic |
| **Console Commands** | 10-20 | Data fetching, synchronization, maintenance |
| **Queued Jobs** | 5-10 | Async background processing |
| **Traits** | 5-15 | Reusable logic (caching, data transformation) |
| **API Resources** | 3-8 | Response transformation |
| **Migrations** | 30-50 | Module-specific database schema |
| **Scheduled Tasks** | 10-15 | Cron-based data sync (every few seconds to daily) |

## Database Isolation

Each module connects to its own dedicated MySQL database:

```
Core App  -->  shared_database     (tenants, users, content, settings)
Module A  -->  module_a_database   (module A domain data)
Module B  -->  module_b_database   (module B domain data)
Module C  -->  module_c_database   (module C domain data)
...
Module K  -->  module_k_database   (module K domain data)
```

Models within each module extend a `BaseModel` that sets the database connection:

```
BaseModel.__construct() --> $this->connection = config('app.DB_CONNECTION_MODULE_X')
```

This ensures all queries from a module automatically use the correct database without explicit connection specification in each query.

## Access Control Flow

```
HTTP Request
  --> Tenant Resolution (domain-based)
  --> CheckTenantModule Middleware (does tenant have access to this module?)
  --> API Scope Guard (does the API token have module:read or module:write scope?)
  --> Module Controller
  --> Module Repository (queries module's dedicated database)
  --> Response
```

## Key Design Patterns

1. **Module Registration via module.json**: Each module declares its service providers, allowing the framework to auto-discover and register them
2. **Dedicated Database Connections**: Prevents data coupling between modules and the core app
3. **Self-Contained Scheduling**: Each module's ServiceProvider registers its own scheduled tasks in the `schedule()` method
4. **API Scope Protection**: Module API routes are protected by granular OAuth-style scopes (e.g., `module:read`, `module:write`)
5. **Singleton Services**: Business services are registered as singletons for performance
6. **BaseModel Pattern**: All module models extend a BaseModel that auto-configures the database connection
