# Next.js Integration with Platformatic Watt

## Package
```
@platformatic/next
```

## Detection Files
- `next.config.js`
- `next.config.ts`
- `next.config.mjs`

## watt.json Configuration

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/next/3.0.0.json",
  "application": {
    "basePath": "/",
    "commands": {
      "development": "next dev",
      "build": "next build",
      "production": "next start"
    }
  },
  "runtime": {
    "logger": {
      "level": "{PLT_SERVER_LOGGER_LEVEL}"
    },
    "server": {
      "hostname": "{PLT_SERVER_HOSTNAME}",
      "port": "{PORT}"
    }
  }
}
```

## Installation

```bash
npm install wattpm
npm install @platformatic/next
```

## Key Considerations

### App Router vs Pages Router
Watt supports both routing paradigms. No special configuration needed.

### Base Path
Set `application.basePath` if your app is not served from the root:
```json
{
  "application": {
    "basePath": "/my-app"
  }
}
```

### Workers (Production Scaling)
For production, configure workers via environment variable:
```
PLT_NEXT_WORKERS=4
```

**Standalone Next.js app** - add to watt.json:
```json
{
  "runtime": {
    "workers": "{PLT_NEXT_WORKERS}"
  }
}
```

**Multi-service setup** - use the service directory name as key in root watt.json:
```json
{
  "workers": {
    "storefront": "{PLT_STOREFRONT_WORKERS}",
    "admin": "{PLT_ADMIN_WORKERS}"
  }
}
```

**Key point:** Use the service directory name (e.g., `"storefront"`), not the full path.

### Output Mode (Important)

**Recommended:** Use `output: 'standalone'` for optimal Watt deployment:
```javascript
// next.config.mjs
export default {
  output: 'standalone',
};
```

**Not supported:** `output: 'export'` (static export) - Watt requires server-side rendering.

### Turbopack
Currently, disable Turbopack and use standard webpack build for compatibility.

## Quick Import

For existing Next.js projects, use the import utility:
```bash
npx wattpm-utils import
```

This auto-generates `watt.json` and installs dependencies.

## Environment Variables

Next.js environment variables work as expected:
- `.env.local` for local development
- `.env.production` for production
- `NEXT_PUBLIC_*` for client-side variables

Watt adds these:
- `PLT_SERVER_HOSTNAME` - Server bind address
- `PLT_SERVER_LOGGER_LEVEL` - Log level (info, debug, warn, error)
- `PORT` - Server port

## Common Issues

### Port Conflicts
Ensure `PORT` env var doesn't conflict with Next.js default 3000. Watt will try the next available port if occupied.

### API Routes
Next.js API routes (`/api/*`) work normally within Watt.

### Middleware
Next.js middleware is fully supported.

### Image Optimization
The built-in Next.js image optimizer works with Watt.

For external image sources (CDN, CMS like Contentful):
```javascript
// next.config.mjs
export default {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'images.ctfassets.net',  // Contentful
      },
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
      },
    ],
  },
};
```

## Multi-Zone Setup

For micro-frontends, configure multiple Next.js apps in a monorepo:
```json
{
  "services": [
    { "path": "./apps/main", "id": "main" },
    { "path": "./apps/admin", "id": "admin" }
  ]
}
```

## TypeScript

Next.js TypeScript projects work out of the box. No additional configuration needed - Watt uses Node.js native type stripping.

---

## ISR and Cache Invalidation

### Cache Tags with `unstable_cache`

Use Next.js 14+ `unstable_cache` with tags for fine-grained cache invalidation:

```typescript
import { unstable_cache } from 'next/cache';

const getProducts = unstable_cache(
  async () => {
    const res = await fetch('http://api.plt.local/products');
    return res.json();
  },
  ['products'],  // cache key
  { tags: ['products'], revalidate: 3600 }
);
```

### On-Demand Revalidation API

Create a revalidation endpoint for webhooks:

```typescript
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  const { tags } = await request.json();

  for (const tag of tags) {
    revalidateTag(tag);
  }

  return Response.json({ revalidated: true, tags });
}
```

### Multi-Worker Cache Behavior

**Important:** Cache tags work across Watt workers. When you call `revalidateTag()`, it invalidates the cache in all worker instances.

### CMS Webhook Pattern

For headless CMS integration (Contentful, Sanity, etc.):

1. CMS webhook triggers your content-worker service
2. Content-worker parses the content type
3. Content-worker calls Next.js revalidate API

```javascript
// In content-worker service
app.post('/webhook/cms', async (req) => {
  const contentType = req.body.sys.contentType.sys.id;
  const tagMap = {
    'product': 'products',
    'article': 'articles',
    'page': 'pages'
  };

  const tag = tagMap[contentType];
  if (tag) {
    await fetch('http://frontend.plt.local/api/revalidate', {
      method: 'POST',
      body: JSON.stringify({ tags: [tag] })
    });
  }
});
```

---

## Performance Optimization

### Standalone Output Mode

For production, use standalone output for smaller images:

```javascript
// next.config.mjs
export default {
  output: 'standalone',
};
```

### Multithreaded SSR with Workers

Configure workers for parallel SSR processing:

```json
{
  "runtime": {
    "workers": {
      "static": "{PLT_NEXT_WORKERS}"
    }
  }
}
```

**CPU Scaling Rule**: Set CPU limit = `PLT_NEXT_WORKERS × 1000m`

Example for 4 workers:
```bash
PLT_NEXT_WORKERS=4
# Requires: cpu limit = 4000m (4 cores)
```

### Distributed Caching

Share ISR cache across replicas with Valkey/Redis:

```json
{
  "cache": {
    "adapter": "valkey",
    "url": "{PLT_VALKEY_HOST}"
  }
}
```

### Optimized Production watt.json

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/next/3.0.0.json",
  "application": {
    "basePath": "/",
    "commands": {
      "development": "next dev",
      "build": "next build",
      "production": "next start"
    }
  },
  "runtime": {
    "logger": {
      "level": "{PLT_SERVER_LOGGER_LEVEL}"
    },
    "server": {
      "hostname": "0.0.0.0",
      "port": "{PORT}"
    },
    "workers": {
      "static": "{PLT_NEXT_WORKERS}"
    }
  },
  "cache": {
    "adapter": "valkey",
    "url": "{PLT_VALKEY_HOST}"
  }
}
```

### Why Watt is Faster

Watt uses **SO_REUSEPORT** (Linux kernel feature) to distribute connections across workers with zero coordination overhead—eliminating the ~30% performance tax from PM2/cluster module IPC-based load balancing.

Benchmarks show **93% latency improvement** vs traditional approaches.

For full details, see [../performance.md](../performance.md)
