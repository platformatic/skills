# Koa Integration with Platformatic Watt

## Package
```
@platformatic/node
```

## Detection
- `package.json` contains `koa` in dependencies

## watt.json Configuration

### JavaScript Project
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.0.0.json",
  "application": {
    "commands": {
      "development": "node --watch src/index.js",
      "build": "echo 'No build step required'",
      "production": "node src/index.js"
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

### TypeScript Project
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.0.0.json",
  "application": {
    "commands": {
      "development": "tsx watch src/index.ts",
      "build": "tsc",
      "production": "node dist/index.js"
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
```

For TypeScript, also add:
```bash
npm install tsx typescript --save-dev
```

## Key Considerations

### Host Binding
Koa must bind to `0.0.0.0` for Watt:
```javascript
const Koa = require('koa');
const app = new Koa();

const port = process.env.PORT || 3000;
app.listen(port, '0.0.0.0');
```

### Middleware Stack
Koa middleware works normally. Ensure async/await patterns are used.

## Health Check

Add a health check route:
```javascript
const Router = require('@koa/router');
const router = new Router();

router.get('/health', (ctx) => {
  ctx.body = { status: 'ok' };
});

app.use(router.routes());
```

## Common Patterns

### Standard Setup
```javascript
const Koa = require('koa');
const Router = require('@koa/router');
const bodyParser = require('koa-bodyparser');
const cors = require('@koa/cors');

const app = new Koa();
const router = new Router();

// Middleware
app.use(cors());
app.use(bodyParser());

// Health check
router.get('/health', (ctx) => {
  ctx.body = { status: 'ok' };
});

// API routes
router.get('/api/users', async (ctx) => {
  ctx.body = { users: [] };
});

app.use(router.routes());
app.use(router.allowedMethods());

// Start server
const port = process.env.PORT || 3000;
app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
```

### With TypeScript
```typescript
import Koa from 'koa';
import Router from '@koa/router';
import bodyParser from 'koa-bodyparser';

const app = new Koa();
const router = new Router();

router.get('/health', (ctx) => {
  ctx.body = { status: 'ok' };
});

app.use(bodyParser());
app.use(router.routes());
app.use(router.allowedMethods());

const port = parseInt(process.env.PORT || '3000');
app.listen(port, '0.0.0.0');
```

### Error Handling
```javascript
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (err) {
    ctx.status = err.status || 500;
    ctx.body = {
      error: err.message
    };
    ctx.app.emit('error', err, ctx);
  }
});

app.on('error', (err, ctx) => {
  console.error('Server error:', err);
});
```

## Environment Variables

- `PORT` - Server port
- `PLT_SERVER_HOSTNAME` - Bind address
- `PLT_SERVER_LOGGER_LEVEL` - Log level
- `NODE_ENV` - Environment (development/production)

## Logging Integration

For structured logging compatible with Watt:
```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.PLT_SERVER_LOGGER_LEVEL || 'info'
});

app.use(async (ctx, next) => {
  const start = Date.now();
  await next();
  const ms = Date.now() - start;
  logger.info({ method: ctx.method, url: ctx.url, ms }, 'request');
});
```

## Graceful Shutdown

Implement graceful shutdown:
```javascript
const server = app.listen(port, '0.0.0.0');

process.on('SIGTERM', () => {
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

## Common Issues

### Context type errors (TypeScript)
Install `@types/koa` and `@types/koa__router`:
```bash
npm install @types/koa @types/koa__router --save-dev
```

### Middleware order
Koa middleware executes in order added. Ensure error handlers are first.

### Response already sent
Always use `return` after sending response in middleware to prevent further processing.
