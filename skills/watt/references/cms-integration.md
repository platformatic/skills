# CMS Integration with Platformatic Watt

Guide for integrating headless CMS platforms (Contentful, Sanity, Strapi, etc.) with Watt.

---

## Architecture Overview

```
                    ┌─────────────────────────────────────┐
                    │           Watt Runtime              │
                    │                                     │
   Browser ────────▶│  ┌─────────────────────────────┐   │
                    │  │         Composer            │   │
                    │  └──────────┬──────────────────┘   │
                    │             │                       │
                    │    ┌────────┼────────┐             │
                    │    ▼        ▼        ▼             │
                    │ ┌─────┐ ┌─────┐ ┌─────────────┐   │
                    │ │Next │ │ API │ │Content      │   │
                    │ │.js  │ │     │ │Worker       │   │
                    │ └─────┘ └─────┘ └──────┬──────┘   │
                    └────────────────────────│───────────┘
                                             │
   CMS Webhook ──────────────────────────────┘
```

**Key components:**
- **Next.js**: SSR frontend with cached content
- **API**: Business logic service
- **Content Worker**: Handles CMS webhooks and cache invalidation

---

## Why a Separate Content Worker?

1. **Isolation**: Webhook processing doesn't block business-critical API calls
2. **Independent scaling**: Scale content operations separately
3. **Rate limiting**: Easier to add queuing for burst webhook traffic
4. **Separation of concerns**: CMS logic isolated from core application

---

## Project Structure

```
my-app/
├── watt.json
├── web/
│   ├── composer/
│   │   └── platformatic.json
│   ├── storefront/          # Next.js frontend
│   │   ├── platformatic.json
│   │   └── app/
│   ├── api/                 # Business API
│   │   ├── platformatic.json
│   │   └── src/
│   └── content-worker/      # CMS webhooks
│       ├── platformatic.json
│       └── src/
```

---

## Content Worker Service

### platformatic.json

```json
{
  "$schema": "https://schemas.platformatic.dev/service/2.0.0.json",
  "service": {
    "openapi": true
  },
  "plugins": {
    "paths": ["./src/app.ts"],
    "typescript": true
  }
}
```

### Webhook Handler

```typescript
// src/app.ts
import { FastifyInstance } from 'fastify';

export default async function (fastify: FastifyInstance) {
  // Health check
  fastify.get('/health', async () => ({ status: 'ok' }));

  // CMS webhook endpoint
  fastify.post('/webhook/cms', async (request, reply) => {
    const secret = request.headers['x-webhook-secret'];

    // Verify webhook secret
    if (secret !== process.env.CMS_WEBHOOK_SECRET) {
      return reply.status(401).send({ error: 'Unauthorized' });
    }

    const payload = request.body as CMSWebhookPayload;
    const contentType = payload.sys?.contentType?.sys?.id;

    // Map content types to cache tags
    const tagMap: Record<string, string> = {
      'product': 'products',
      'article': 'articles',
      'page': 'pages',
      'category': 'categories',
    };

    const tag = tagMap[contentType];
    if (tag) {
      // Call Next.js revalidate API
      await fetch('http://storefront.plt.local/api/revalidate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ tags: [tag] }),
      });

      fastify.log.info({ contentType, tag }, 'Cache invalidated');
    }

    return { success: true };
  });
}

interface CMSWebhookPayload {
  sys: {
    contentType?: {
      sys: {
        id: string;
      };
    };
  };
}
```

---

## Next.js Revalidation Endpoint

```typescript
// app/api/revalidate/route.ts
import { revalidateTag } from 'next/cache';
import { NextRequest } from 'next/server';

export async function POST(request: NextRequest) {
  // Optional: verify internal request
  const origin = request.headers.get('origin');

  const { tags } = await request.json();

  for (const tag of tags) {
    revalidateTag(tag);
  }

  return Response.json({
    revalidated: true,
    tags,
    timestamp: Date.now(),
  });
}
```

---

## Composer Routing

```json
{
  "$schema": "https://schemas.platformatic.dev/composer/2.0.0.json",
  "composer": {
    "services": [
      {
        "id": "storefront",
        "origin": "internal://storefront",
        "proxy": { "prefix": "/" }
      },
      {
        "id": "api",
        "origin": "internal://api",
        "proxy": { "prefix": "/api/v1" }
      },
      {
        "id": "content-worker",
        "origin": "internal://content-worker",
        "proxy": { "prefix": "/content" }
      }
    ]
  }
}
```

CMS webhook URL: `https://your-domain.com/content/webhook/cms`

---

## Contentful-Specific Setup

### Environment Variables

```bash
CONTENTFUL_SPACE_ID=abc123xyz
CONTENTFUL_ACCESS_TOKEN=xxx...
CONTENTFUL_PREVIEW_TOKEN=xxx...
CONTENTFUL_WEBHOOK_SECRET=my-secret-123
```

### Content Fetching with Cache Tags

```typescript
// lib/contentful.ts
import { unstable_cache } from 'next/cache';

const client = createClient({
  space: process.env.CONTENTFUL_SPACE_ID!,
  accessToken: process.env.CONTENTFUL_ACCESS_TOKEN!,
});

export const getProducts = unstable_cache(
  async () => {
    const entries = await client.getEntries({
      content_type: 'product',
    });
    return entries.items;
  },
  ['contentful-products'],
  { tags: ['products'], revalidate: 3600 }
);

export const getArticles = unstable_cache(
  async () => {
    const entries = await client.getEntries({
      content_type: 'article',
    });
    return entries.items;
  },
  ['contentful-articles'],
  { tags: ['articles'], revalidate: 3600 }
);
```

### Webhook Configuration in Contentful

1. Go to **Settings** → **Webhooks**
2. Create new webhook:
   - **URL**: `https://your-domain.com/content/webhook/cms`
   - **Triggers**: Entry publish, Entry unpublish
   - **Headers**: `x-webhook-secret: your-secret`
   - **Content type filter**: Select relevant types

---

## Mock Data for Development

Enable development without CMS credentials:

```typescript
// lib/contentful.ts
const USE_MOCK = !process.env.CONTENTFUL_SPACE_ID;

export const getProducts = unstable_cache(
  async () => {
    if (USE_MOCK) {
      return mockProducts;
    }
    const entries = await client.getEntries({ content_type: 'product' });
    return entries.items;
  },
  ['products'],
  { tags: ['products'] }
);

// Mock data matching Contentful structure
const mockProducts = [
  {
    sys: { id: '1' },
    fields: {
      title: 'Mock Product',
      slug: 'mock-product',
      price: 99.99,
    },
  },
];
```

---

## Draft Preview Mode

### Enable Draft Mode

```typescript
// app/api/draft/route.ts
import { draftMode } from 'next/headers';
import { redirect } from 'next/navigation';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const secret = searchParams.get('secret');
  const slug = searchParams.get('slug');

  if (secret !== process.env.CONTENTFUL_PREVIEW_SECRET) {
    return new Response('Invalid token', { status: 401 });
  }

  draftMode().enable();
  redirect(slug || '/');
}
```

### Use Preview Client

```typescript
export const getProduct = async (slug: string, preview = false) => {
  const client = preview ? previewClient : deliveryClient;
  const entries = await client.getEntries({
    content_type: 'product',
    'fields.slug': slug,
  });
  return entries.items[0];
};
```

---

## Other CMS Platforms

### Sanity

```typescript
// Webhook payload structure
interface SanityWebhook {
  _type: string;
  _id: string;
}

const tagMap: Record<string, string> = {
  'product': 'products',
  'post': 'articles',
};
```

### Strapi

```typescript
// Webhook payload structure
interface StrapiWebhook {
  model: string;
  entry: { id: number };
}

const tagMap: Record<string, string> = {
  'product': 'products',
  'article': 'articles',
};
```

---

## Best Practices

1. **Verify webhooks**: Always check webhook secrets
2. **Log invalidations**: Track what's being invalidated for debugging
3. **Batch invalidations**: Group multiple changes when possible
4. **Use mock data**: Enable development without CMS credentials
5. **Isolate content operations**: Use a dedicated worker service
6. **Monitor webhook latency**: Slow webhooks can delay content updates

---

## Resources

- [Contentful Webhooks](https://www.contentful.com/developers/docs/webhooks/)
- [Next.js Cache Revalidation](https://nextjs.org/docs/app/building-your-application/caching#revalidating)
- [Sanity Webhooks](https://www.sanity.io/docs/webhooks)
- [Strapi Webhooks](https://docs.strapi.io/dev-docs/backend-customization/webhooks)
