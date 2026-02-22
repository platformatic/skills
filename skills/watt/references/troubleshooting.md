# Troubleshooting Platformatic Watt

## Common Issues and Solutions

### Node.js Version Issues

**Error**: `Watt requires Node.js v22.19.0 or higher`

**Solution**:
```bash
# Check current version
node --version

# Install Node.js 22 using nvm
nvm install 22
nvm use 22

# Or using fnm
fnm install 22
fnm use 22

# Verify
node --version  # Should show v22.x.x
```

---

### Port Already in Use

**Error**: `Error: listen EADDRINUSE: address already in use :::3000`

**Solutions**:

1. **Find and kill the process**:
```bash
# Find process using port 3000
lsof -i :3000

# Kill it
kill -9 <PID>
```

2. **Use a different port**:
```bash
PORT=3001 npm run dev
```

3. **Let Watt auto-select**:
Watt automatically tries the next available port if the default is occupied.

---

### watt.json Not Found

**Error**: `Cannot find watt.json`

**Solution**:
```bash
# Initialize Watt in project
npm create wattpm

# Or create manually with appropriate framework config
```

---

### Invalid watt.json

**Error**: `Error parsing watt.json: Unexpected token`

**Solution**:
```bash
# Validate JSON syntax
node -e "JSON.parse(require('fs').readFileSync('watt.json'))"

# Common issues:
# - Trailing commas
# - Missing quotes on keys
# - Comments (JSON doesn't support comments)
```

---

### wattpm Command Not Found

**Error**: `wattpm: command not found`

**Solutions**:

1. **Install wattpm**:
```bash
npm install wattpm
```

2. **Use npx**:
```bash
npx wattpm dev
npx wattpm build
npx wattpm start
```

3. **Check package.json scripts**:
```json
{
  "scripts": {
    "dev": "wattpm dev",
    "build": "wattpm build",
    "start": "wattpm start"
  }
}
```

---

### App Not Binding to Correct Host

**Error**: Application not accessible from Docker/Kubernetes

**Cause**: App binding to `localhost` instead of `0.0.0.0`

**Solution**: Update your app to bind to `0.0.0.0`:

```javascript
// Express
app.listen(process.env.PORT || 3000, '0.0.0.0');

// Fastify
fastify.listen({ port: process.env.PORT || 3000, host: '0.0.0.0' });

// NestJS
await app.listen(process.env.PORT || 3000, '0.0.0.0');
```

---

### TypeScript Compilation Errors

**Error**: `Cannot find module` or type errors

**Solutions**:

1. **Use Node.js 22.19+** (native type stripping support)

2. **Keep TypeScript for type-checking only (optional)**:
```bash
npm install typescript --save-dev
```

3. **Update watt.json commands**:
```json
{
  "application": {
    "commands": {
      "development": "node --watch src/index.ts",
      "build": "echo 'No build step required'",
      "production": "node src/index.ts"
    }
  }
}
```

---

### Build Fails in Production

**Error**: Build succeeds locally but fails in CI/CD

**Solutions**:

1. **Check Node.js version in CI**:
```yaml
# GitHub Actions
- uses: actions/setup-node@v4
  with:
    node-version: '22'
```

2. **Ensure all dependencies are in package.json**:
```bash
# Check for missing deps
npm ls
```

3. **Clear npm cache**:
```bash
npm cache clean --force
rm -rf node_modules package-lock.json
npm install
```

---

### Health Check Failing

**Error**: Container keeps restarting / Health check timeout

**Solutions**:

1. **Ensure health endpoint exists**:
```javascript
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

2. **Increase startup time**:
```yaml
# Kubernetes
startupProbe:
  httpGet:
    path: /health
    port: 3000
  failureThreshold: 30
  periodSeconds: 5
```

3. **Check application logs**:
```bash
# Docker
docker logs <container>

# Kubernetes
kubectl logs <pod>
```

---

### Memory Issues

**Error**: `JavaScript heap out of memory`

**Solutions**:

1. **Increase Node.js memory**:
```bash
NODE_OPTIONS="--max-old-space-size=4096" npm start
```

2. **Update watt.json**:
```json
{
  "application": {
    "commands": {
      "production": "node --max-old-space-size=4096 dist/index.js"
    }
  }
}
```

3. **Check for memory leaks**:
```bash
# Profile with clinic
npm install -g clinic
clinic doctor -- node dist/index.js
```

---

### Environment Variables Not Loading

**Error**: `process.env.VAR_NAME` is undefined

**Solutions**:

1. **Check .env file exists**:
```bash
ls -la .env
```

2. **Ensure correct format**:
```
# .env
VAR_NAME=value
# No spaces around =
# No quotes needed for simple values
```

3. **Load dotenv early** (if not using a framework with built-in support):
```javascript
// First line of entry file
require('dotenv').config();
```

4. **For production, set vars directly**:
```bash
export VAR_NAME=value
# or
VAR_NAME=value npm start
```

---

### Next.js Specific Issues

**Static Export Error**:
```
Error: Static export is not supported with Watt
```
**Solution**: Change `next.config.js`:
```javascript
// Remove or change:
// output: 'export'

// Use:
module.exports = {
  // output: 'standalone' for Docker
}
```

**Turbopack Issues**:
Disable Turbopack temporarily:
```json
{
  "scripts": {
    "dev": "next dev"  // Remove --turbo flag
  }
}
```

---

### Fastify Plugin Errors

**Error**: `Plugin must be a function`

**Solution**: Check plugin registration:
```javascript
// Correct
fastify.register(require('@fastify/cors'));

// Incorrect
fastify.register('@fastify/cors');  // String, not module
```

---

### NestJS Build Errors

**Error**: `nest-cli.json not found`

**Solution**:
```bash
# Ensure NestJS CLI is installed
npm install @nestjs/cli --save-dev

# Create nest-cli.json
{
  "compilerOptions": {
    "assets": ["**/*.graphql"],
    "watchAssets": true
  }
}
```

---

### Docker Build Fails

**Error**: `npm ERR! code ENOENT` during Docker build

**Solutions**:

1. **Check .dockerignore**:
Don't ignore `package.json` or `package-lock.json`

2. **Use correct COPY order**:
```dockerfile
# Copy package files first
COPY package*.json ./
RUN npm ci

# Then copy source
COPY . .
```

3. **Verify all files exist**:
```bash
docker build --no-cache -t app .
```

---

### Kubernetes Pod CrashLoopBackOff

**Diagnosis**:
```bash
# Check pod status
kubectl describe pod <pod-name>

# Check logs
kubectl logs <pod-name> --previous
```

**Common causes**:
1. Health check failing (see above)
2. Missing environment variables
3. Database connection failing
4. Insufficient resources

---

### Composer Service Discovery Issues

**Error**: Service not found or routing not working

**Solutions**:

1. **Use `internal://` origin for in-memory communication**:
```json
{
  "composer": {
    "services": [
      {
        "id": "api",
        "origin": "internal://api",
        "proxy": { "prefix": "/api" }
      }
    ]
  }
}
```

2. **Check service ID matches directory name**:
The service `id` must match the directory name under `web/`.

3. **Verify route priority**:
More specific prefixes should come after less specific ones:
```json
{
  "services": [
    { "id": "frontend", "proxy": { "prefix": "/" } },
    { "id": "api", "proxy": { "prefix": "/api/v1" } }
  ]
}
```

---

### Worker Configuration Issues

**Error**: Workers not starting or wrong service scaled

**Solutions**:

1. **Use service directory name as key**:
```json
{
  "workers": {
    "storefront": 4,
    "api": 2
  }
}
```
Use `"storefront"`, not `"web/storefront"` or the full path.

2. **Environment variable for dynamic config**:
```json
{
  "workers": {
    "storefront": "{PLT_STOREFRONT_WORKERS}"
  }
}
```

---

### Environment Variable Injection Issues

**Error**: `{VAR_NAME}` appearing literally instead of value

**Solutions**:

1. **Use correct syntax**:
```json
{
  "port": "{PORT}",
  "workers": "{PLT_WORKERS}"
}
```
Note: Use `{VAR_NAME}`, not `${VAR_NAME}` or `$VAR_NAME`.

2. **Ensure variable is set**:
```bash
export PORT=3000
# or
PORT=3000 npm run dev
```

3. **Check .env file is loaded**:
The `.env` file must be in the project root.

---

### Next.js ISR Cache Issues in Multi-Worker Setup

**Error**: Cache invalidation not propagating to all workers

**Solutions**:

1. **Use `revalidateTag()` for cache invalidation**:
```typescript
import { revalidateTag } from 'next/cache';
revalidateTag('products');
```
This works across all Watt workers.

2. **For distributed deployments, use Valkey/Redis**:
```json
{
  "cache": {
    "adapter": "valkey",
    "url": "{PLT_VALKEY_HOST}"
  }
}
```

3. **Create a revalidation API endpoint**:
```typescript
// app/api/revalidate/route.ts
export async function POST(request) {
  const { tags } = await request.json();
  for (const tag of tags) {
    revalidateTag(tag);
  }
  return Response.json({ revalidated: true });
}
```

---

### Inter-Service Communication Not Working

**Error**: `fetch('http://service.plt.local/...')` failing

**Solutions**:

1. **Use correct hostname pattern**:
```javascript
// Correct
fetch('http://api.plt.local/users')

// Incorrect
fetch('http://api/users')
fetch('http://localhost:3001/users')
```

2. **Ensure service is running**:
```bash
wattpm info  # List running services
```

3. **Check service health**:
```bash
curl http://localhost:3042/api/health
```

---

## Debugging Tips

### Enable Debug Logging
```bash
PLT_SERVER_LOGGER_LEVEL=debug npm run dev
```

### Check Watt Version
```bash
npx wattpm --version
```

### Validate Configuration
```bash
# Check watt.json schema
node -e "
const config = require('./watt.json');
console.log(JSON.stringify(config, null, 2));
"
```

### Network Debugging
```bash
# Check if port is listening
netstat -tlnp | grep 3000

# Test health endpoint
curl -v http://localhost:3000/health
```

## Getting Help

- **Platformatic Docs**: https://docs.platformatic.dev
- **GitHub Issues**: https://github.com/platformatic/platformatic/issues
- **Discord Community**: https://discord.gg/platformatic
