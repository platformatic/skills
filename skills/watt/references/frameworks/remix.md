# Remix Integration with Platformatic Watt

## Package
```
@platformatic/remix
```

## Detection Files
- `remix.config.js`
- `remix.config.ts`

## watt.json Configuration

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/remix/3.0.0.json",
  "application": {
    "commands": {
      "development": "remix vite:dev",
      "build": "remix vite:build",
      "production": "remix-serve ./build/server/index.js"
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

### Remix v1 (Classic Compiler)
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/remix/3.0.0.json",
  "application": {
    "commands": {
      "development": "remix dev",
      "build": "remix build",
      "production": "remix-serve build"
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
npm install wattpm @platformatic/remix
```

## Key Considerations

### Vite vs Classic Compiler
Remix v2 uses Vite by default. Detect by checking:
- `vite.config.ts` exists → Vite-based
- Only `remix.config.js` → Classic compiler

Adjust commands accordingly.

### Server Adapter
Watt works with the default `remix-serve`. For custom servers (Express adapter), adjust the production command.

### Build Output
- Vite: `./build/server/index.js`
- Classic: `./build`

## Environment Variables

Remix uses `.env` files by default:
```
PORT=3000
PLT_SERVER_HOSTNAME=0.0.0.0
PLT_SERVER_LOGGER_LEVEL=info
SESSION_SECRET=your-secret-here
```

Access in loaders/actions:
```typescript
export const loader = async () => {
  const apiUrl = process.env.API_URL;
  // ...
};
```

## Common Patterns

### With Vite (Remix v2+)
```typescript
// vite.config.ts
import { vitePlugin as remix } from "@remix-run/dev";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [remix()],
  server: {
    port: parseInt(process.env.PORT || "3000"),
    host: "0.0.0.0",
  },
});
```

### remix.config.js (Classic)
```javascript
/** @type {import('@remix-run/dev').AppConfig} */
module.exports = {
  ignoredRouteFiles: ["**/.*"],
  serverModuleFormat: "cjs",
};
```

### Health Check Route
Create `app/routes/health.tsx`:
```typescript
import { json } from "@remix-run/node";

export const loader = () => {
  return json({ status: "ok" });
};
```

## Deployment Considerations

### Session Storage
For production, use a database-backed session:
```typescript
import { createCookieSessionStorage } from "@remix-run/node";

export const sessionStorage = createCookieSessionStorage({
  cookie: {
    name: "__session",
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    secrets: [process.env.SESSION_SECRET!],
    sameSite: "lax",
  },
});
```

### Database Connections
Initialize database connections outside request handlers:
```typescript
// app/db.server.ts
import { PrismaClient } from "@prisma/client";

let prisma: PrismaClient;

if (process.env.NODE_ENV === "production") {
  prisma = new PrismaClient();
} else {
  if (!global.__db__) {
    global.__db__ = new PrismaClient();
  }
  prisma = global.__db__;
}

export { prisma };
```

## TypeScript

Remix projects are TypeScript by default. Watt handles this natively with Node.js type stripping.

## Common Issues

### Build fails with Vite
Ensure `vite.config.ts` is properly configured and all dependencies are installed:
```bash
npm install @remix-run/dev vite --save-dev
```

### Static assets not loading
Check that `public/` directory is properly served. Watt handles static files automatically.

### HMR not working
Ensure WebSocket connections are allowed. Vite HMR should work out of the box with Watt.

### Custom server not recognized
If using a custom Express server, switch to `@platformatic/node` instead:
```json
{
  "application": {
    "commands": {
      "production": "node server.js"
    }
  }
}
```
