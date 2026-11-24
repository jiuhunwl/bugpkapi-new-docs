# Architecture Overview

[简体中文](architecture.md) · **English**

BugPk-Api consists of three repositories: the Go backend, the Vue frontend, and this documentation repository. The frontend and backend communicate through `/system-api` (admin console and customer site APIs) and `/api/{apiCode}` (the open API gateway).

## Backend Layering

The backend follows DDD layering, organized by responsibility under `internal/`:

```text
cmd/bugpk-api        main service entry: load config, initialize infrastructure, register routes, graceful shutdown
internal/app         application assembly layer: route registration, startup bootstrap, lifecycle, contract tests
internal/domain      domain layer: auth / user / interface / billing / payment / risk / ticket / content ...
internal/infra       infrastructure: MySQL, Redis, migrations, encryption, object storage adapters
internal/pkg         shared utilities: logging, error codes, pagination, time, password policy, etc.
migrations           database migrations, the single source of truth
sql/full             full SQL initialization generated from migrations
```

The dependency direction is fixed as `app → domain → infra/pkg`. The domain layer is unaware of HTTP details, and routing and parameter parsing are centralized in the app layer.

## Startup Flow

```text
Read config.yaml and environment variables
  → validate config (secrets, TLS, timezone confirmation, database/Redis connectivity)
  → optionally run migrations (MIGRATION_ENABLED)
  → bootstrap the first administrator (INITIAL_ADMIN_PASSWORD)
  → initialize cache, rate limiter, captcha providers
  → register system routes and open API routes
  → start the HTTP server, exposing /healthz and /readyz
  → on SIGINT/SIGTERM, stop accepting new requests, wait for in-flight requests, release resources
```

If any critical step fails, startup is blocked so the service never runs in a "half-initialized" state.

## Request Paths

### Admin Console and Customer Site

The frontend accesses `/system-api/*` on the same origin, and the backend handles JWT verification, RBAC authorization, parameter validation, and business processing. All write operations are recorded in the operation log; errors return unified error codes so the frontend can map them for i18n.

### Open API Gateway

```text
Caller requests /api/{apiCode}
  → validate API Key and ownership
  → validate API status, QPS, and plan quota
  → parameter validation and request body rules
  → forward to upstream (connection pool, timeout, retry policy per API configuration)
  → result caching (optional) and response normalization
  → record call log (asynchronously aggregated by minute/hour/day)
  → billing deduction and transaction persistence
```

The gateway depends only on API configuration and never hardcodes upstream addresses in code; a new API can go live simply by configuring it in the admin console.

## Data and Billing

- **API configuration**: APIs, groups, upstream addresses, timeouts, caching, rate limits, and billing methods are all configured in the database.
- **Plans**: three types — site-wide plans, single-API plans, and credits packages; plan changes are driven by orders.
- **Orders and payment**: the order state machine and idempotent payment callback handling are separated; callbacks must be signature-verified, deduplicated, and amount-checked.
- **Call log**: raw logs and aggregate tables are separated; dashboards read only the aggregate tables, so large-table aggregation never drags down the home page.
- **Audit**: admin console write operations, logins, and key changes go into the operation log and can be searched by user and time.

## Deployment Topology

Same-origin deployment is recommended to reduce cross-origin and cookie issues:

```text
Browser
  │  HTTPS
  ▼
Nginx ── /system-api, /api ──> Go service (127.0.0.1:8080)
  └──── other paths ─────────> frontend static assets (SPA fallback to index.html)

Go service ──> MySQL 8
           └─> Redis 6+
```

The backend can also serve the frontend assets directly (`FRONTEND_SERVE_ENABLED`), which suits small single-machine deployments; when scale grows, split it into a separate static site and a separate backend.

## Extension Points

- **Payment channels**: adding a channel only requires implementing the unified APIs for order creation, callback signature verification, and reconciliation; it can be enabled once the keys are configured in the admin console.
- **Captcha**: multiple providers are supported and configured per scenario, with fallback on failure.
- **Content and branding**: site name, logo, SEO, footer, and agreement templates are all configurable in the backend, so the code is not tied to a specific brand.
- **Internationalization**: all frontend copy goes through language packs; adding a language only requires adding a language file with a symmetric key structure.