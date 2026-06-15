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

## CPU Profiling with pprof

Use Watt's native `pprof` commands when you need CPU profile data for flamegraph analysis of a running Watt application.

Platformatic's profiling work spans CPU profiling, heap profiling, and flamegraph analysis. Use CPU profiles to find hot code paths and event-loop bottlenecks, and use heap snapshots/profiles when memory grows over time or objects are not being released.

Install `@platformatic/flame` when you want to turn pprof data into interactive HTML flamegraphs and markdown analysis:

```bash
npm install -g @platformatic/flame
```

### 1. Start the Application

Build and start the application in production mode so the profile matches production behavior as closely as possible:

```bash
wattpm build
wattpm start
```

### 2. Identify the Runtime and Application

In another terminal, list the running Watt applications and their application IDs:

```bash
wattpm ps
wattpm applications
```

If only one runtime or application is running, `wattpm pprof` can auto-detect it. Otherwise, pass the runtime ID and application ID explicitly.

### 3. Collect a CPU Profile

Start profiling, generate the workload you want to inspect, then stop profiling:

```bash
# Profile all applications in the detected runtime
wattpm pprof start

# Or profile one application explicitly
wattpm pprof start my-app api-application

# Exercise the slow endpoint or workload here

wattpm pprof stop my-app api-application
```

`wattpm pprof stop` saves profile data as `pprof-{application}-{timestamp}.pb` files in the current working directory.

### 4. Generate a Flamegraph

Use `@platformatic/flame` to generate an interactive HTML flamegraph and markdown analysis from the pprof file created by Watt:

```bash
flame generate pprof-api-application-2026-06-15T10-00-00-000Z.pb
```

To choose the HTML output path:

```bash
flame generate -o api-cpu-profile.html pprof-api-application-2026-06-15T10-00-00-000Z.pb
```

For more verbose markdown analysis:

```bash
flame generate --md-format=detailed pprof-api-application-2026-06-15T10-00-00-000Z.pb
```

### 5. Profile the Right Workload

- Profile `wattpm start`, not `wattpm dev`, when investigating production latency.
- Keep the profiling window short and focused on the slow request path.
- Use `wattpm logs` while profiling to correlate profile timing with application errors or slow operations.
- Profile a specific application when only one service is slow; profile all applications when the bottleneck is unclear.

### Memory Companion

For memory issues, use `wattpm heap-snapshot` instead of CPU profiling:

```bash
wattpm heap-snapshot my-app api-application
```

Heap snapshots are saved as `heap-{application}-{timestamp}.heapsnapshot` files and can be loaded in Chrome DevTools Memory tab.

Watt and `@platformatic/flame` support heap profiling alongside CPU profiling. Treat memory profiling as a separate diagnostic pass: first reproduce the memory growth, then capture heap data near the point where retained objects are visible.

For standalone Node.js scripts outside Watt, `@platformatic/flame` can collect CPU and heap profiles directly and generate flamegraphs when the process exits:

```bash
flame run server.js
```

This writes timestamped CPU and heap `.pb` files, interactive `.html` flamegraphs, and `.md` analysis files. CPU and heap files from the same run share a timestamp so they can be correlated.

Use manual mode when you want to start and stop profiling around a specific workload:

```bash
flame run --manual server.js
flame toggle
# Exercise the workload
flame toggle
```

For transpiled or bundled applications, pass sourcemap directories so generated stack frames can map back to original sources:

```bash
flame run --sourcemap-dirs=dist:build server.js
```

### When to Use Each Profile

| Symptom | Use | Why |
|---------|-----|-----|
| High CPU or slow requests | `wattpm pprof start` / `wattpm pprof stop` | Captures where CPU time is spent for flamegraph analysis |
| Memory growth or suspected leak | `wattpm heap-snapshot` | Captures retained objects for memory inspection |
| Existing `.pb` profile file | `flame generate profile.pb` | Creates interactive HTML and markdown analysis |
| Standalone Node.js script | `flame run server.js` | Captures CPU and heap profiles and generates flamegraphs on exit |
| Unknown bottleneck | CPU profile first, then heap snapshot if memory grows | CPU profiles explain hot paths; heap data explains retention |

### Flamegraph Analysis Notes

- Look for wide frames first; they represent where most sampled CPU time is spent.
- Focus on application frames before framework or runtime frames unless the runtime frames dominate the profile.
- Compare profiles before and after a change; single profiles are useful, but deltas make regressions easier to prove.
- Keep production profiles short and targeted so the captured data corresponds to the workload being investigated.

---

## Performance Checklist

- [ ] Use `output: 'standalone'` in next.config.mjs
- [ ] Configure `PLT_NEXT_WORKERS` based on available CPU
- [ ] Set CPU limits = workers × 1000m
- [ ] Enable distributed caching with Valkey/Redis
- [ ] Configure health checks with appropriate delays
- [ ] Enable Prometheus metrics for monitoring
- [ ] Use `wattpm pprof start` and `wattpm pprof stop` to capture CPU profiles for slow workloads
- [ ] Use multi-stage Docker builds
- [ ] Set `NODE_ENV=production`

---

## Resources

- [Deploy Next.js in Kubernetes with Watt](https://docs.platformatic.dev/docs/guides/deployment/nextjs-in-k8s)
- [93% Faster Next.js in Kubernetes](https://blog.platformatic.dev/93-faster-nextjs-in-your-kubernetes)
- [Addressing Overprovisioning via Multiple Workers](https://blog.platformatic.dev/addressing-overprovisioning-performance-issues-in-nodejs-via-multiple-workers)
- [Next-Gen Flamegraph for Node.js](https://blog.platformatic.dev/introducing-next-gen-flamegraphs-for-nodejs)
- [Heap Profiling Now in @platformatic/flame & Watt](https://blog.platformatic.dev/announcing-heap-profiling-support-in-platformaticflame-and-watt-runtime)
- [k8s-watt-performance-demo Repository](https://github.com/platformatic/k8s-watt-performance-demo)
