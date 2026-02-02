# Enterprise Patterns for Platformatic Watt

This guide covers enterprise-grade patterns for multi-service architectures, API composition, and multi-repository setups.

---

## Multi-Service Architecture

Watt runs multiple services within a single orchestrated runtime, with Platformatic Composer routing requests to each service.

### Architecture Overview

```
                    ┌─────────────────────────────────────┐
                    │           Watt Runtime              │
                    │                                     │
   Requests ───────▶│  ┌─────────────────────────────┐   │
                    │  │    Platformatic Composer    │   │
                    │  │    (API Gateway/Router)     │   │
                    │  └──────────┬──────────────────┘   │
                    │             │                       │
                    │    ┌────────┼────────┬─────────┐   │
                    │    ▼        ▼        ▼         ▼   │
                    │ ┌─────┐ ┌─────┐ ┌───────┐ ┌─────┐ │
                    │ │Next │ │Node │ │Fastify│ │ DB  │ │
                    │ │.js  │ │.js  │ │  API  │ │ API │ │
                    │ └─────┘ └─────┘ └───────┘ └─────┘ │
                    └─────────────────────────────────────┘
```

### Project Structure

```
my-enterprise-app/
├── watt.json                    # Root orchestration config
├── package.json
├── .env
└── web/
    ├── composer/                # API Gateway
    │   └── platformatic.json
    ├── frontend/                # Next.js/Remix frontend
    │   ├── platformatic.json
    │   └── src/
    ├── api/                     # Fastify API service
    │   ├── platformatic.json
    │   └── src/
    ├── db/                      # Platformatic DB service
    │   ├── platformatic.json
    │   └── migrations/
    └── worker/                  # Background worker service
        ├── platformatic.json
        └── src/
```

### Root watt.json Configuration

**Using autoload (recommended):**
```json
{
  "$schema": "https://schemas.platformatic.dev/watt/2.0.0.json",
  "server": {
    "hostname": "0.0.0.0",
    "port": "{PORT}"
  },
  "autoload": {
    "path": "web"
  },
  "entrypoint": "composer",
  "workers": {
    "frontend": "{PLT_FRONTEND_WORKERS}"
  }
}
```

**Using explicit service list:**
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/runtime/3.0.0.json",
  "entrypoint": "composer",
  "web": [
    { "id": "composer", "path": "web/composer" },
    { "id": "frontend", "path": "web/frontend" },
    { "id": "api", "path": "web/api" },
    { "id": "db", "path": "web/db" }
  ],
  "workers": {
    "frontend": "{PLT_FRONTEND_WORKERS}"
  },
  "runtime": {
    "logger": { "level": "{PLT_SERVER_LOGGER_LEVEL}" },
    "server": {
      "hostname": "{PLT_SERVER_HOSTNAME}",
      "port": "{PORT}"
    }
  }
}
```

### Worker Configuration

Configure multiple workers per service for SSR-heavy workloads:

```json
{
  "workers": {
    "frontend": "{PLT_FRONTEND_WORKERS}",
    "api": 2
  }
}
```

**Key points:**
- Use the **service directory name** as the key (e.g., `"frontend"`, not `"web/frontend"`)
- Worker count can be a number or environment variable
- Each worker handles requests independently
- Recommended: `PLT_FRONTEND_WORKERS=4` for production

### Environment Variable Injection

Use `{VAR_NAME}` syntax to inject environment variables in config files:

```json
{
  "port": "{PORT}",
  "workers": {
    "frontend": "{PLT_FRONTEND_WORKERS}"
  }
}
```

This works in any Watt/Platformatic JSON config file.

---

## Platformatic Composer (API Gateway)

Composer acts as the central orchestration service, routing requests to appropriate backend services.

### Composer Configuration

`web/composer/platformatic.json`:
```json
{
  "$schema": "https://schemas.platformatic.dev/composer/2.0.0.json",
  "composer": {
    "services": [
      {
        "id": "frontend",
        "origin": "internal://frontend",
        "proxy": { "prefix": "/" }
      },
      {
        "id": "api",
        "origin": "internal://api",
        "proxy": { "prefix": "/api" }
      },
      {
        "id": "content-worker",
        "origin": "internal://content-worker",
        "proxy": { "prefix": "/content" }
      },
      {
        "id": "db",
        "origin": "internal://db",
        "proxy": { "prefix": "/db" }
      }
    ],
    "refreshTimeout": 5000
  }
}
```

### Service Origins

Use `internal://` for in-memory service mesh communication:

| Origin Type | Example | Use Case |
|-------------|---------|----------|
| `internal://service-id` | `internal://api` | Services within same Watt runtime |
| `http://host:port` | `http://external:3000` | External services |

**Key point:** `internal://` enables zero-network-overhead communication between services.

### Path-Based Routing

| Path | Service | Description |
|------|---------|-------------|
| `/` | frontend | Next.js/Remix UI |
| `/api/*` | api | Fastify REST API |
| `/content/*` | content-worker | CMS webhooks, background tasks |
| `/db/*` | db | Platformatic DB auto-generated API |
| `/graphql` | db | GraphQL endpoint |
| `/documentation` | composer | Aggregated OpenAPI docs |

### Route Priority

**Important:** Routes are matched in order. Place more specific prefixes **after** less specific ones:

```json
{
  "services": [
    { "id": "frontend", "proxy": { "prefix": "/" } },
    { "id": "api", "proxy": { "prefix": "/api/v1" } },
    { "id": "content", "proxy": { "prefix": "/content" } }
  ]
}
```

The catch-all `/` route should typically be listed first for the frontend.

---

## Inter-Service Communication

Services communicate using internal hostnames without network overhead.

### Internal Service URLs

**From application code (fetch):**
```javascript
// From api service, call db service
const response = await fetch('http://db.plt.local/users');

// From frontend, call api service (server-side)
const data = await fetch('http://api.plt.local/products');

// Call content-worker to trigger cache invalidation
await fetch('http://content-worker.plt.local/revalidate', {
  method: 'POST',
  body: JSON.stringify({ tags: ['products'] })
});
```

**From Composer config (internal://):**
```json
{
  "origin": "internal://api"
}
```

### Two Communication Patterns

| Context | Pattern | Example |
|---------|---------|---------|
| Application code (fetch) | `http://{service-id}.plt.local` | `http://api.plt.local/users` |
| Composer config | `internal://{service-id}` | `internal://api` |

Both enable zero-network-overhead in-memory communication.

### Hostname Pattern

```
{service-id}.plt.local
```

Examples:
- `http://frontend.plt.local`
- `http://api.plt.local`
- `http://db.plt.local`
- `http://content-worker.plt.local`

---

## Platformatic DB Service

Auto-generate REST and GraphQL APIs from your database schema.

### DB Service Configuration

`web/db/platformatic.json`:
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/db/3.0.0.json",
  "db": {
    "connectionString": "{DATABASE_URL}",
    "graphql": true,
    "openapi": true
  },
  "migrations": {
    "dir": "./migrations",
    "autoApply": true
  },
  "authorization": {
    "adminSecret": "{PLT_ADMIN_SECRET}"
  }
}
```

### Features

- **Auto-generated REST**: CRUD endpoints from tables
- **Auto-generated GraphQL**: Full GraphQL schema
- **Migrations**: SQL migrations with auto-apply
- **Authorization**: Role-based access control

---

## Multi-Repository Setup

For organizations with microservices in separate Git repositories.

### watt.json with Remote Services

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/runtime/3.0.0.json",
  "entrypoint": "gateway",
  "web": [
    {
      "id": "gateway",
      "path": "web/gateway"
    },
    {
      "id": "user-service",
      "path": "{PLT_USER_SERVICE_PATH}",
      "url": "https://github.com/your-org/user-service.git"
    },
    {
      "id": "product-service",
      "path": "{PLT_PRODUCT_SERVICE_PATH}",
      "url": "https://github.com/your-org/product-service.git"
    },
    {
      "id": "order-service",
      "path": "{PLT_ORDER_SERVICE_PATH}",
      "url": "https://github.com/your-org/order-service.git"
    }
  ]
}
```

### Environment Configuration

**Development (.env.development)**:
```bash
# Point to local sibling directories for active development
PLT_USER_SERVICE_PATH=../user-service
PLT_PRODUCT_SERVICE_PATH=../product-service
PLT_ORDER_SERVICE_PATH=../order-service
```

**Production (.env.production)**:
```bash
# Point to resolved directories after Git clone
PLT_USER_SERVICE_PATH=web/user-service
PLT_PRODUCT_SERVICE_PATH=web/product-service
PLT_ORDER_SERVICE_PATH=web/order-service
```

### CLI Commands

| Command | Purpose |
|---------|---------|
| `npm run resolve` | Clone services from Git, install dependencies |
| `npm run build` | Build all services |
| `npm run dev` | Start development with file watching |
| `npm run start` | Launch production |

### package.json Scripts

```json
{
  "scripts": {
    "resolve": "wattpm resolve",
    "dev": "wattpm dev",
    "build": "wattpm build",
    "start": "wattpm start"
  }
}
```

---

## Enterprise Features with Watt-Extra

For production-grade enterprise deployments, Watt-Extra provides:

- **Monitoring**: Prometheus metrics, distributed tracing
- **Compliance**: Audit logging, security policies
- **Caching**: Distributed cache integration
- **Authentication**: SSO, JWT validation
- **ICC Integration**: Infrastructure Control Center

Wrap existing services without code changes:

```json
{
  "extends": "@platformatic/watt-extra",
  "monitoring": {
    "prometheus": true,
    "tracing": true
  },
  "authentication": {
    "jwt": {
      "secret": "{JWT_SECRET}"
    }
  }
}
```

---

## Example: Full Enterprise Stack

### Directory Structure

```
enterprise-app/
├── watt.json
├── package.json
├── .env
├── docker-compose.yml
└── web/
    ├── gateway/
    │   └── platformatic.json      # Composer config
    ├── dashboard/
    │   ├── platformatic.json      # Next.js config
    │   ├── next.config.mjs
    │   └── app/
    ├── users-api/
    │   ├── platformatic.json      # Fastify config
    │   └── routes/
    ├── users-db/
    │   ├── platformatic.json      # DB config
    │   └── migrations/
    ├── products-api/
    │   ├── platformatic.json
    │   └── routes/
    └── products-db/
        ├── platformatic.json
        └── migrations/
```

### Root watt.json

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/runtime/3.0.0.json",
  "entrypoint": "gateway",
  "web": [
    { "id": "gateway", "path": "web/gateway" },
    { "id": "dashboard", "path": "web/dashboard" },
    { "id": "users-api", "path": "web/users-api" },
    { "id": "users-db", "path": "web/users-db" },
    { "id": "products-api", "path": "web/products-api" },
    { "id": "products-db", "path": "web/products-db" }
  ],
  "runtime": {
    "logger": { "level": "{PLT_SERVER_LOGGER_LEVEL}" },
    "server": {
      "hostname": "{PLT_SERVER_HOSTNAME}",
      "port": "{PORT}"
    }
  }
}
```

### Gateway Configuration

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/composer/3.0.0.json",
  "composer": {
    "services": [
      { "id": "dashboard", "proxy": { "prefix": "/" } },
      { "id": "users-api", "proxy": { "prefix": "/api/users" } },
      { "id": "users-db", "proxy": { "prefix": "/db/users" } },
      { "id": "products-api", "proxy": { "prefix": "/api/products" } },
      { "id": "products-db", "proxy": { "prefix": "/db/products" } }
    ]
  }
}
```

---

## Docker Compose for Development

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3042:3042"
    environment:
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3042
      - DATABASE_URL=postgresql://user:pass@db:5432/app
      - PLT_SERVER_LOGGER_LEVEL=info
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=app
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

---

## Resources

- [Composer Next Node Fastify Demo](https://github.com/platformatic/composer-next-node-fastify)
- [Multi-Repository Guide](https://docs.platformatic.dev/docs/guides/use-watt-multiple-repository)
- [wattpm-resolve Demo](https://github.com/platformatic/wattpm-resolve)
- [Platformatic DB Documentation](https://docs.platformatic.dev/docs/db/overview)
- [Platformatic Composer Documentation](https://docs.platformatic.dev/docs/composer/overview)
