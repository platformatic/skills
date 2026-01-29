# Astro Integration with Platformatic Watt

## Package
```
@platformatic/astro
```

## Detection Files
- `astro.config.mjs`
- `astro.config.ts`
- `astro.config.js`

## watt.json Configuration

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/astro/3.0.0.json",
  "application": {
    "commands": {
      "development": "astro dev",
      "build": "astro build",
      "production": "node ./dist/server/entry.mjs"
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
npm install wattpm @platformatic/astro
```

## Key Considerations

### Output Mode
Watt requires server-side rendering. Configure Astro for SSR:

```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  output: 'server', // or 'hybrid' for partial SSR
  adapter: node({
    mode: 'standalone'
  }),
  server: {
    port: parseInt(process.env.PORT || '3000'),
    host: '0.0.0.0'
  }
});
```

### Required Adapter
Install the Node.js adapter:
```bash
npm install @astrojs/node
```

### Static Output Not Supported
`output: 'static'` (default) won't work with Watt. Use `server` or `hybrid`.

## Environment Variables

Create `.env`:
```
PORT=3000
PLT_SERVER_HOSTNAME=0.0.0.0
PLT_SERVER_LOGGER_LEVEL=info
PUBLIC_API_URL=https://api.example.com
```

Access in Astro:
```astro
---
// Server-side only
const secret = import.meta.env.API_SECRET;

// Available on client (PUBLIC_ prefix)
const apiUrl = import.meta.env.PUBLIC_API_URL;
---
```

## Common Patterns

### Full SSR Configuration
```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';
import react from '@astrojs/react';

export default defineConfig({
  output: 'server',
  adapter: node({
    mode: 'standalone'
  }),
  integrations: [react()],
  server: {
    port: parseInt(process.env.PORT || '3000'),
    host: process.env.PLT_SERVER_HOSTNAME || '0.0.0.0'
  },
  vite: {
    server: {
      watch: {
        usePolling: true // For Docker/containers
      }
    }
  }
});
```

### Hybrid Rendering
```javascript
// astro.config.mjs
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  output: 'hybrid', // Static by default, opt-in to SSR
  adapter: node({
    mode: 'standalone'
  })
});
```

Then in pages:
```astro
---
// This page will be server-rendered
export const prerender = false;
---
```

### Health Check Endpoint
Create `src/pages/health.json.ts`:
```typescript
import type { APIRoute } from 'astro';

export const GET: APIRoute = () => {
  return new Response(
    JSON.stringify({ status: 'ok' }),
    {
      status: 200,
      headers: {
        'Content-Type': 'application/json'
      }
    }
  );
};
```

## Build Output

Astro with Node adapter outputs to:
- `./dist/server/entry.mjs` - Server entry point
- `./dist/client/` - Static assets

Watt serves both automatically.

## Framework Integrations

Astro works with multiple UI frameworks:
```bash
# React
npx astro add react

# Vue
npx astro add vue

# Svelte
npx astro add svelte
```

All work seamlessly with Watt.

## TypeScript

Astro has built-in TypeScript support. No additional configuration needed for Watt.

## Common Issues

### "output: static" error
Change to `output: 'server'` or `output: 'hybrid'` in `astro.config.mjs`.

### Missing adapter
Install and configure `@astrojs/node`:
```bash
npm install @astrojs/node
```

### Port binding issues
Ensure `server.host` is set to `'0.0.0.0'` in astro.config.

### Build fails
Check that all dependencies are installed and integrations are properly configured.

### HMR not working in Docker
Add polling to vite config:
```javascript
vite: {
  server: {
    watch: {
      usePolling: true
    }
  }
}
```

## Content Collections

Astro's content collections work normally with Watt. No special configuration needed.
