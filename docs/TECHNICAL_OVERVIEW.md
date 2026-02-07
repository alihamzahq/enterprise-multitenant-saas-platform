# Multi-Tenant Sports Content Management SaaS Platform

**Author:** Ali Hamza — Full Stack Developer  
**Tech Stack:** Laravel 10 · React 19 · Inertia.js · MySQL · Redis · REST APIs  
**Role:** Lead Developer — Architecture, Backend, Frontend, DevOps  

---

## Project Overview

This is a production-grade, multi-tenant SaaS platform built for managing sports content across multiple digital publications. Each tenant (publication) gets its own custom domain, theme, and subscription plan — with tenant data scoped via `tenant_id` in a shared database. The platform serves real-time sports data, AI-powered content generation, and rich editorial tools.

The platform powers **10 integrated sport modules** (Football, Tennis, Rugby, Basketball, Cricket, Golf, Ice Hockey, Horse Racing, Formula One, Rugby Union), each architecturally isolated as a standalone Laravel module with its own database, models, migrations, and API layer. Tenants share a common database and are scoped using `tenant_id` filtering throughout the application.

---

## Key Highlights

| Metric | Value |
|--------|-------|
| **Codebase Size** | 300+ PHP files, 100+ React components |
| **Database Architecture** | 12 databases (1 central + 1 shared tenant DB + 10 sport modules) |
| **Models** | 60+ core + 200+ module-specific |
| **Controllers** | 60+ |
| **Background Jobs** | 10+ queue jobs |
| **Console Commands** | 25+ artisan commands |
| **Permissions System** | 60+ permissions across 15 roles |
| **API Integrations** | 10+ third-party services |

---

## Architecture

### Multi-Tenancy (Shared Database, Tenant Scoping)

All tenants share a single database, with data isolation enforced via `tenant_id` scoping:

```
Central DB ──── Users, subscriptions, plans, tenant registry
Tenant Data ─── Shared tables scoped by tenant_id (articles, settings, themes)
Sport DBs  ──── 10 separate databases (one per sport module)
```

**Implementation:** Stancl Tenancy with automatic domain/subdomain resolution, tenant-scoped middleware, and `tenant_id` filtering across all tenant-specific queries. This single-database approach simplifies deployment and maintenance while still providing logical data separation between publications.

### Modular Sport Architecture

Each sport is a self-contained Laravel Module (`nwidart/laravel-modules`):

```
Modules/<Sport>/
├── App/
│   ├── Models/          # Sport-specific Eloquent models
│   ├── Controllers/     # HTTP controllers
│   ├── Repositories/    # Data access layer
│   ├── Services/        # Business logic
│   ├── Jobs/            # Background queue jobs
│   └── Providers/       # Service providers
├── Database/
│   ├── Migrations/      # Schema definitions
│   └── Seeders/         # Sample data
├── routes/
│   ├── api.php          # REST API endpoints
│   └── web.php          # Web routes
└── config/              # Module configuration
```

Modules can be independently enabled/disabled, migrated, and deployed — allowing selective feature rollout per tenant subscription tier.

### Design Patterns

- **Repository Pattern** — Interface-driven data access layer with 10+ repository implementations, enabling testable and swappable data sources
- **Service Layer** — Business logic encapsulated in dedicated service classes for content rewriting, image processing, widget resolution, etc.
- **Event-Driven Architecture** — Real-time broadcasting via Pusher/Laravel Echo WebSockets
- **Inertia.js SSR Bridge** — Server-side rendering with React, combining Laravel routing with React components without building a separate SPA

---

## Tech Stack

### Backend

| Technology | Purpose |
|-----------|---------|
| **Laravel 10** | Core framework — routing, ORM, middleware, queues |
| **PHP 8.2** | Language runtime |
| **MySQL** | Relational database (12 databases) |
| **Redis** | Caching, sessions, queue broker |
| **Laravel Sanctum** | API token authentication |
| **Laravel Horizon** | Queue monitoring dashboard |
| **Laravel Modules** | Modular architecture for sport modules |
| **Stancl Tenancy** | Multi-tenancy (shared database, tenant_id scoping) |
| **Spatie Permissions** | Role-based access control (60+ permissions, 15 roles) |
| **Spatie Activity Log** | Audit trail and activity tracking |

### Frontend

| Technology | Purpose |
|-----------|---------|
| **React 19** | UI library |
| **Inertia.js 2.0** | Server-driven SPA (no separate API needed) |
| **Vite 4** | Build tool with HMR |
| **SASS** | CSS preprocessing |
| **CKEditor 5** | Rich text editor for content creation |
| **React DnD** | Drag-and-drop for content reordering |
| **React Cropper** | Image editing and cropping |
| **SweetAlert2** | Modal dialogs and confirmations |
| **React Toastify** | Toast notifications |

### Third-Party Integrations

| Service | Purpose |
|---------|---------|
| **Stripe** (Laravel Cashier) | Subscription billing, invoices, payment methods |
| **OpenAI** | AI-powered article rewriting and content generation |
| **Pusher / Laravel Echo** | Real-time WebSocket notifications |
| **Mailgun** | Transactional email delivery |
| **Google OAuth** | Social authentication |
| **reCAPTCHA** | Bot protection |
| **Cloudflare** | DNS management and CDN |
| **Firebase** | Push notifications and topic sync |
| **New Relic** | Application performance monitoring |

### DevOps & Quality

| Tool | Purpose |
|------|---------|
| **Bitbucket Pipelines** | CI/CD pipeline |
| **Laravel Telescope** | Request and query monitoring |
| **Laravel Debugbar** | Development debugging |

---

## Core Features

### Content Management System
- Full CRUD for articles with rich text editing (CKEditor 5)
- Article categorization, tagging, and taxonomy management
- Multi-author support with author profiles
- Article reordering via drag-and-drop
- RSS feed aggregation from external sources
- AI-powered article rewriting via OpenAI integration
- Image gallery with cropping, WebP conversion, and optimization
- SEO management with meta tags, sitemaps, and structured data

### Multi-Tenant Platform
- Tenant creation via admin panel with automated provisioning
- Custom domain and subdomain support per tenant
- Per-tenant theme customization with multiple templates and color palettes
- Shared database with `tenant_id` scoping for logical data separation
- Tenant-scoped migrations and seeders
- SSL certificate generation for custom domains

### Subscription & Billing
- Stripe integration for recurring subscriptions
- Multiple subscription tiers with feature gating
- Invoice generation and payment history
- Payment method management

### Real-Time Features
- WebSocket-based live notifications via Pusher/Laravel Echo
- Live sports centre with real-time match data
- Firebase push notification integration
- Match notifications to tenant subscribers

### Theme & Widget System
- Customizable theme templates per tenant
- Widget-based page composition
- Sport-specific widgets (standings, fixtures, scores, player stats)
- Color palette management
- Static page builder with custom meta data

### Administration
- 60+ granular permissions across 15 roles
- Activity logging and audit trail
- Queue monitoring via Laravel Horizon dashboard
- User management with profile types
- Passwordless login support
- Frontend deployment management from admin panel

---

## Background Processing

The platform relies heavily on queued jobs for scalability:

- **Tenant Provisioning** — Automated tenant setup, database seeding, and migration execution
- **AI Content Processing** — Article rewriting via OpenAI with queue-based async processing
- **Image Processing** — Optimization, resizing, and WebP conversion on upload
- **RSS Feed Aggregation** — Scheduled fetching from multiple external sports news sources
- **Notification Delivery** — Firebase topic sync and real-time match alerts to tenant subscribers
- **Permission Management** — Bulk permission updates propagated across all tenants
- **Data Synchronization** — Sports teams and widget data extraction

Queue infrastructure: **Redis** with **Laravel Horizon** monitoring.

---

## Database Design

### 12-Database Architecture

```
┌──────────────────────────────────────────────────────────┐
│                        MAIN DB                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Global configurations, default settings, plans,   │  │
│  │  system-wide content shared across all tenants     │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│                       TENANT DB                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Users, Roles, Subscriptions, Payments, Domains    │  │
│  │  Articles, Themes, Widgets, Settings               │  │
│  │  ─── all scoped by tenant_id ───                   │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│                     SPORT MODULES                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│  │ sport_1  │ │ sport_2  │ │ sport_3  │ │ sport_4  │     │
│  ├──────────┤ ├──────────┤ ├──────────┤ ├──────────┤     │
│  │ sport_5  │ │ sport_6  │ │ sport_7  │ │ sport_8  │     │
│  ├──────────┤ ├──────────┤                               │
│  │ sport_9  │ │ sport_10 │                               │
│  └──────────┘ └──────────┘                               │
└──────────────────────────────────────────────────────────┘
```

### Key Model Domains (60+ Core + 200+ Module-Specific)

- **Content** — Articles, authors, categories, sections, pages, topics, subjects
- **Tenancy** — Tenants, leagues, sports, subscriptions, custom domains
- **Theming** — Themes, configurations, color palettes, components, widgets
- **Billing** — Products, pricing, payment credentials
- **Settings** — General, AI, SEO, and notification configurations
- **System** — Activity logs, queue logs, API clients, API tokens

---

## API Layer

### Route Structure

- **Authentication Routes** — Register, login, password reset
- **Tenant Web Routes** — 200+ routes covering articles, users, themes, widgets, and settings
- **Tenant API Routes** — Public REST API for articles, sitemaps, categories, newsletters
- **Central API Routes** — Contact forms, status updates, system endpoints
- **Module Routes** — Sport-specific API endpoints per module

**Authentication:** Laravel Sanctum (token-based) + session-based for web
**Response Format:** Inertia.js responses (web) + JSON resources (API)

---

## My Role & Contributions

As the **lead developer**, I was responsible for:

- **System Architecture** — Designed the multi-tenant, modular architecture from scratch, using Stancl Tenancy with shared-database tenant scoping and nwidart/laravel-modules for sport module separation
- **Backend Development** — Built the entire Laravel backend including 60+ controllers, repository pattern implementation, service layer, queue jobs, and 25+ artisan commands
- **Frontend Development** — Developed the React-based admin panel with Inertia.js, including article management, theme customization, widget builder, and real-time notification interfaces
- **Database Design** — Architected the shared-database multi-tenant system with `tenant_id` scoping, plus 10 dedicated sport module databases; designed migration strategies for both central and sport-specific schemas
- **API Integrations** — Integrated 10+ third-party services including Stripe payments, OpenAI content generation, Pusher real-time, Firebase notifications, and RSS feed aggregation
- **Payment System** — Implemented subscription management with Stripe, including plan management, invoice generation, and payment method handling
- **AI Features** — Built the article rewriting pipeline using OpenAI's API with queue-based processing for scalability
- **Permission System** — Designed and implemented 60+ granular permissions across 15 roles with tenant-scoped access control
- **DevOps** — Set up CI/CD with Bitbucket Pipelines, configured Laravel Horizon for queue monitoring
- **Performance Optimization** — Database indexing strategies, query optimization, cron timing adjustments, and redundant index cleanup

---

## Project Structure

```
SportsCMS/
├── app/
│   ├── Http/Controllers/        # 60+ controllers
│   ├── Models/                  # 60+ Eloquent models
│   ├── Services/                # Business logic layer
│   ├── Repositories/            # Data access layer
│   ├── Jobs/                    # 10+ background jobs
│   ├── Events/                  # Event-driven architecture
│   ├── Console/Commands/        # 25+ artisan commands
│   └── Interfaces/              # Repository contracts
├── Modules/                     # 10 sport modules
│   ├── Football/
│   ├── Tennis/
│   ├── Rugby/
│   ├── BasketBall/
│   ├── Cricket/
│   ├── Golf/
│   ├── Icehockey/
│   ├── Horseracing/
│   ├── FormulaOne/
│   └── RugbyUnion/
├── resources/js/
│   ├── Pages/                   # React page components
│   ├── components/ui/           # Reusable UI components
│   ├── contexts/                # React Context state
│   ├── hooks/                   # Custom React hooks
│   └── Layouts/                 # Layout components
├── database/
│   ├── migrations/              # Central migrations
│   └── tenant_migrations/       # Tenant-scoped migrations
├── routes/
│   ├── web.php                  # 200+ web routes
│   ├── api.php                  # REST API routes
│   └── auth.php                 # Authentication routes
├── config/                      # 30+ configuration files
└── tests/                       # Pest PHP test suite
```

---

## Screenshots

> *This is a proprietary project and cannot be shared publicly.*

---

## Summary

This project demonstrates expertise in:

- **Enterprise SaaS Architecture** — Multi-tenant design with shared-database tenant scoping, modular sport engines, and subscription-based access control
- **Full Stack Development** — Laravel 10 backend with React 19 frontend, connected via Inertia.js for server-driven SPA behavior
- **API Design & Integration** — RESTful APIs, 10+ third-party service integrations, real-time WebSocket communication
- **Scalable Infrastructure** — Queue-based processing (Redis), background job architecture, and Laravel Horizon monitoring
- **AI Integration** — OpenAI-powered content generation pipeline with queue-based processing
- **Payment Processing** — Stripe subscription management with invoice handling
- **Security** — Role-based access control (60+ permissions), Sanctum authentication, reCAPTCHA, tenant-scoped data access
- **Code Quality** — CI/CD pipeline with automated testing and deployment

---

*This documentation represents a proprietary project. Source code is not publicly available. This portfolio document is for demonstration of technical skills and architecture decisions.*
