# POC Preparation Checklist

Technical requirements and steps for integrating an existing Node.js application with Platformatic Watt.

---

## Prerequisites

### Development Environment

- [ ] **Node.js 22.19+** installed and tested with your application
- [ ] **Linux or macOS** preferred (Windows possible but may impact performance)
- [ ] **Docker** installed and running
- [ ] **Local code editor** for live modifications

### Application Requirements

- [ ] Working local copy of your application
- [ ] Application starts locally without complex rebuild processes
- [ ] If rebuilds are required, expect significant delays during integration

### Infrastructure Access

- [ ] Local instances of required databases and third-party APIs, OR
- [ ] Network access to remote databases/APIs from local machine
- [ ] **For load testing**: Local mocks of databases and third-party APIs (highly recommended)

---

## Quick Start: Make It Run with Watt

### Step 1: Install Dependencies

```bash
npm install wattpm @platformatic/node
```

### Step 2: Create watt.json

Create `watt.json` in your project root:

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.33.0.json",
  "application": {
    "main": "index.js"
  }
}
```

Replace `index.js` with your actual application entrypoint (e.g., `src/server.js`, `dist/main.js`).

### Step 3: Modify Your Entrypoint

Your entrypoint file should export one of:

#### Option A: Export a `create` function (Recommended)

```javascript
// index.js
const express = require('express')

async function create() {
  const app = express()

  app.get('/health', (req, res) => {
    res.json({ status: 'ok' })
  })

  // Add your routes...

  return app
}

module.exports = { create }
```

#### Option B: Export a `close` function

```javascript
// index.js
const express = require('express')

const app = express()
let server

// Your routes...
app.get('/health', (req, res) => {
  res.json({ status: 'ok' })
})

server = app.listen(process.env.PORT || 3000)

async function close() {
  return new Promise((resolve) => {
    server.close(resolve)
  })
}

module.exports = { close }
```

### Step 4: Run with Watt

```bash
# Development mode (with auto-reload)
npx wattpm dev

# Production mode
npx wattpm start
```

---

## Entrypoint Patterns by Framework

### Express

```javascript
// server.js
const express = require('express')

async function create() {
  const app = express()

  app.use(express.json())

  // Your middleware and routes
  app.get('/health', (req, res) => res.json({ status: 'ok' }))

  return app
}

module.exports = { create }
```

### Fastify

```javascript
// server.js
const fastify = require('fastify')

async function create() {
  const app = fastify({ logger: true })

  app.get('/health', async () => ({ status: 'ok' }))

  // Your routes and plugins

  return app
}

module.exports = { create }
```

### Koa

```javascript
// server.js
const Koa = require('koa')

async function create() {
  const app = new Koa()

  // Your middleware
  app.use(async (ctx) => {
    if (ctx.path === '/health') {
      ctx.body = { status: 'ok' }
    }
  })

  return app.callback()
}

module.exports = { create }
```

### NestJS

```javascript
// main.js
const { NestFactory } = require('@nestjs/core')
const { AppModule } = require('./app.module')

async function create() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log']
  })

  await app.init()
  return app.getHttpAdapter().getInstance()
}

module.exports = { create }
```

### Generic HTTP Server

```javascript
// server.js
const http = require('http')

async function create() {
  const server = http.createServer((req, res) => {
    if (req.url === '/health') {
      res.writeHead(200, { 'Content-Type': 'application/json' })
      res.end(JSON.stringify({ status: 'ok' }))
      return
    }
    // Your request handling
  })

  return server
}

module.exports = { create }
```

---

## Full watt.json Configuration

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/node/3.33.0.json",
  "application": {
    "main": "src/server.js",
    "commands": {
      "development": "node --watch src/server.js",
      "build": "npm run build",
      "production": "node dist/server.js"
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

---

## Environment Setup

Create `.env` file:

```bash
PLT_SERVER_HOSTNAME=0.0.0.0
PLT_SERVER_LOGGER_LEVEL=info
PORT=3000

# Your app-specific variables
DATABASE_URL=postgresql://localhost:5432/mydb
API_KEY=your-api-key
```

---

## Troubleshooting

### Application doesn't start

1. **Check Node.js version**: Must be 22.19.0+
   ```bash
   node --version
   ```

2. **Verify entrypoint exports**: Must export `create` or `close`
   ```bash
   node -e "console.log(require('./index.js'))"
   ```

3. **Check for port conflicts**: Watt will auto-select next available port

### Cannot connect to database

1. Ensure database is running locally
2. Check `DATABASE_URL` in `.env`
3. Verify network access from local machine

### TypeScript projects

For TypeScript, either:
- Build first: `npm run build && npx wattpm start`
- Use tsx: Update `watt.json` main to use compiled output

```json
{
  "application": {
    "main": "dist/index.js"
  }
}
```

---

## Validation Checklist

Before the POC session, verify:

- [ ] `npm install` completes without errors
- [ ] `npm install wattpm @platformatic/node` succeeds
- [ ] `watt.json` created with correct entrypoint
- [ ] Entrypoint exports `create` or `close` function
- [ ] `npx wattpm start` launches the application
- [ ] Health endpoint responds: `curl http://localhost:3000/health`
- [ ] Database connections work
- [ ] API integrations function

---

## Next Steps After Basic Integration

1. **Add workers for performance**: See [performance.md](performance.md)
2. **Set up multi-service architecture**: See [enterprise.md](enterprise.md)
3. **Configure deployment**: See [deployment/docker.md](deployment/docker.md)
4. **Add Kafka integration**: See [kafka.md](kafka.md)
