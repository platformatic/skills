# Fastify Integration with Platformatic Watt

## Package
```
@platformatic/node
```

## Detection
- `package.json` contains `fastify` in dependencies

## watt.json Configuration

### Standard Fastify
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

### With @fastify/cli
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.0.0.json",
  "application": {
    "commands": {
      "development": "fastify start -w -l info -P src/app.js",
      "build": "echo 'No build step required'",
      "production": "fastify start -l info src/app.js"
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

### TypeScript
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
npm install wattpm @platformatic/node
```

## Key Considerations

### Host Binding
Ensure Fastify binds to `0.0.0.0`:
```javascript
const fastify = require('fastify')({ logger: true });

fastify.listen({
  port: process.env.PORT || 3000,
  host: '0.0.0.0'
});
```

### Pino Logger Integration
Fastify uses Pino by default, which aligns with Watt's logging. You can share configuration:
```javascript
const fastify = require('fastify')({
  logger: {
    level: process.env.PLT_SERVER_LOGGER_LEVEL || 'info'
  }
});
```

### Plugin System
Fastify plugins work normally within Watt. No special configuration needed.

## Health Check

Register a health check route:
```javascript
fastify.get('/health', async (request, reply) => {
  return { status: 'ok' };
});
```

## Common Patterns

### Standard Setup
```javascript
const fastify = require('fastify')({
  logger: {
    level: process.env.PLT_SERVER_LOGGER_LEVEL || 'info'
  }
});

// Register plugins
fastify.register(require('@fastify/cors'));
fastify.register(require('@fastify/helmet'));

// Routes
fastify.get('/health', async () => ({ status: 'ok' }));

// Start server
const start = async () => {
  try {
    await fastify.listen({
      port: process.env.PORT || 3000,
      host: '0.0.0.0'
    });
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
```

### With TypeScript
```typescript
import Fastify from 'fastify';

const fastify = Fastify({
  logger: {
    level: process.env.PLT_SERVER_LOGGER_LEVEL || 'info'
  }
});

fastify.get('/health', async () => {
  return { status: 'ok' };
});

const start = async () => {
  try {
    await fastify.listen({
      port: parseInt(process.env.PORT || '3000'),
      host: '0.0.0.0'
    });
  } catch (err) {
    fastify.log.error(err);
    process.exit(1);
  }
};

start();
```

### With @fastify/autoload
```javascript
const path = require('path');
const AutoLoad = require('@fastify/autoload');

module.exports = async function (fastify, opts) {
  fastify.register(AutoLoad, {
    dir: path.join(__dirname, 'plugins'),
    options: Object.assign({}, opts)
  });

  fastify.register(AutoLoad, {
    dir: path.join(__dirname, 'routes'),
    options: Object.assign({}, opts)
  });
};
```

## Environment Variables

- `PORT` - Server port
- `PLT_SERVER_HOSTNAME` - Bind address
- `PLT_SERVER_LOGGER_LEVEL` - Pino log level

## Graceful Shutdown

Fastify handles graceful shutdown automatically. Watt coordinates with this:
```javascript
// Optional: custom close handlers
fastify.addHook('onClose', async (instance) => {
  // Cleanup connections
});
```

## Common Issues

### Address already in use
Watt automatically tries the next available port. Ensure your code reads from `process.env.PORT`.

### Validation errors
If using @fastify/env, ensure schema allows environment variables from Watt.

### Slow startup
For large plugin trees, consider lazy loading or using `--inspect` to debug startup time.
