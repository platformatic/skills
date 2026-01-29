# Express Integration with Platformatic Watt

## Package
```
@platformatic/node
```

## Detection
- `package.json` contains `express` in dependencies
- No framework-specific config file

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

### Entry Point Detection
Look for the main file in order:
1. `package.json` `main` field
2. `src/index.js` or `src/index.ts`
3. `src/app.js` or `src/app.ts`
4. `index.js` or `index.ts`
5. `app.js` or `app.ts`
6. `server.js` or `server.ts`

### Port Binding
Express apps MUST read port from environment:
```javascript
const port = process.env.PORT || 3000;
app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
```

**Important**: Bind to `0.0.0.0`, not `localhost` or `127.0.0.1`.

### Process Manager Migration
Remove any process manager (PM2, forever, nodemon for production) - Watt handles process management.

Update package.json:
```json
{
  "scripts": {
    "dev": "wattpm dev",
    "build": "wattpm build",
    "start": "wattpm start"
  }
}
```

## Health Check Endpoint

Add a health check endpoint for Watt monitoring:
```javascript
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

## Common Patterns

### With Middleware Stack
```javascript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json());

// Routes
app.use('/api', apiRoutes);

// Health check
app.get('/health', (req, res) => res.json({ status: 'ok' }));

const port = process.env.PORT || 3000;
app.listen(port, '0.0.0.0');
```

### With TypeScript
```typescript
import express, { Application } from 'express';

const app: Application = express();
const port = process.env.PORT || 3000;

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
```

## Environment Variables

Standard Express environment:
- `NODE_ENV` - development/production
- `PORT` - Server port (provided by Watt)

Watt additions:
- `PLT_SERVER_HOSTNAME` - Bind address
- `PLT_SERVER_LOGGER_LEVEL` - Log level

## Common Issues

### Cannot bind to port
Ensure you're binding to `0.0.0.0` not `localhost`.

### App crashes on startup
Check that all environment variables are available. Use `.env` file for local development.

### Graceful shutdown
Express doesn't handle graceful shutdown by default. Watt manages this, but you can add your own handlers:
```javascript
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```
