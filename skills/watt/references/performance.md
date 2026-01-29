# Performance Optimization for Platformatic Watt

## How Watt Achieves Better Performance

Watt leverages **SO_REUSEPORT**, a Linux kernel feature that distributes connections across multiple Node.js processes with zero coordination overhead. This eliminates the ~30% performance tax that PM2 and the cluster module impose through IPC-based load balancing.

Instead of having a master process coordinate workers, Watt lets the Linux kernel handle load distribution directly.

### Benchmark Results

Tests show three distinct performance tiers:
- **Watt & Deno**: ~11-14ms average latency (fastest)
- **Node.js**: ~20ms average latency
- **Bun**: ~246ms average latency (highest)

Watt achieves up to **93% latency improvement** compared to traditional approaches.

---

## Worker Configuration

### Setting Workers

Configure workers via environment variable or watt.json:

```bash
# Environment variable
PLT_NEXT_WORKERS=4
```

```json
// watt.json
{
  "runtime": {
    "workers": {
      "static": "{PLT_NEXT_WORKERS}"
    }
  }
}
```

### CPU Scaling Rule

**Critical**: CPU limits must scale proportionally with workers.

| Workers | CPU Request | CPU Limit |
|---------|-------------|-----------|
| 1 | 1000m | 1000m |
| 2 | 2000m | 2000m |
| 4 | 4000m | 4000m |

Formula: `CPU = workers × 1000m`

---

## Next.js Specific Optimizations

### 1. Standalone Output Mode

Use Next.js standalone output for smaller Docker images:

```javascript
// next.config.mjs
export default {
  output: 'standalone',
};
```

### 2. Multithreaded SSR

Watt provides multithreaded Server-Side Rendering configured to handle multiple concurrent SSR requests efficiently.

```json
{
  "runtime": {
    "workers": {
      "static": "{PLT_NEXT_WORKERS}"
    }
  }
}
```

### 3. Distributed Caching with Valkey/Redis

Share cache across pod replicas to reduce API load:

```json
{
  "cache": {
    "adapter": "valkey",
    "url": "{PLT_VALKEY_HOST}"
  }
}
```

Benefits:
- Shared ISR cache across all replicas
- Reduced database/API load
- Consistent cache state

### 4. Incremental Static Regeneration (ISR)

Configure time-based revalidation:

```typescript
// In your page
export const revalidate = 10; // Revalidate every 10 seconds
```

---

## Kubernetes Resource Configuration

### Recommended Pod Resources

```yaml
resources:
  requests:
    cpu: "1000m"      # Guaranteed minimum
    memory: "256Mi"
  limits:
    cpu: "1000m"      # Maximum allowed
    memory: "1024Mi"
```

### Scaling with Workers

For 4 workers:
```yaml
resources:
  requests:
    cpu: "4000m"
    memory: "1024Mi"
  limits:
    cpu: "4000m"
    memory: "2048Mi"
```

### Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
```

---

## Optimized watt.json for Production

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

---

## Optimized Dockerfile (Standalone Mode)

```dockerfile
# Build stage
FROM node:22-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM node:22-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3000
ENV PLT_SERVER_HOSTNAME=0.0.0.0

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy standalone output
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

---

## Kubernetes Deployment for Performance

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextjs-watt
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nextjs-watt
  template:
    metadata:
      labels:
        app: nextjs-watt
    spec:
      containers:
        - name: app
          image: your-registry/nextjs-watt:latest
          ports:
            - containerPort: 3000
            - containerPort: 9090  # Prometheus metrics
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
            - name: PLT_SERVER_HOSTNAME
              value: "0.0.0.0"
            - name: PLT_NEXT_WORKERS
              value: "2"
            - name: PLT_VALKEY_HOST
              value: "valkey://cache:6379"
          resources:
            requests:
              cpu: "2000m"    # 2 workers × 1000m
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "1024Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
```

---

## Prometheus Monitoring

Watt automatically exposes metrics on port 9090:

```yaml
# ServiceMonitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nextjs-watt-monitor
spec:
  selector:
    matchLabels:
      app: nextjs-watt
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

Metrics include:
- Node.js runtime metrics
- HTTP request latency/throughput
- Worker utilization
- Cache hit rates

---

## Performance Checklist

- [ ] Use `output: 'standalone'` in next.config.mjs
- [ ] Configure `PLT_NEXT_WORKERS` based on available CPU
- [ ] Set CPU limits = workers × 1000m
- [ ] Enable distributed caching with Valkey/Redis
- [ ] Configure health checks with appropriate delays
- [ ] Enable Prometheus metrics for monitoring
- [ ] Use multi-stage Docker builds
- [ ] Set `NODE_ENV=production`

---

## Resources

- [Deploy Next.js in Kubernetes with Watt](https://docs.platformatic.dev/docs/guides/deployment/nextjs-in-k8s)
- [93% Faster Next.js in Kubernetes](https://blog.platformatic.dev/93-faster-nextjs-in-your-kubernetes)
- [Addressing Overprovisioning via Multiple Workers](https://blog.platformatic.dev/addressing-overprovisioning-performance-issues-in-nodejs-via-multiple-workers)
- [k8s-watt-performance-demo Repository](https://github.com/platformatic/k8s-watt-performance-demo)
