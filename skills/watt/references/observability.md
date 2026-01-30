# Observability in Platformatic Watt

Comprehensive guide for logging, tracing, and metrics in Watt applications.

---

## Custom Logging with Pino

Watt uses [Pino](https://github.com/pinojs/pino) for high-performance structured logging.

### Basic Configuration

```json
{
  "logger": {
    "level": "{PLT_SERVER_LOGGER_LEVEL}",
    "timestamp": "isoTime",
    "base": {
      "service": "my-app",
      "version": "1.0.0"
    }
  }
}
```

### Log Levels

Standard levels: `trace`, `debug`, `info`, `warn`, `error`, `fatal`, `silent`

**Custom levels:**
```json
{
  "logger": {
    "customLevels": {
      "verbose": 15,
      "notice": 35
    }
  }
}
```

### File Transport

Write logs to files:

```json
{
  "logger": {
    "transport": {
      "targets": [{
        "target": "pino/file",
        "options": {
          "destination": "{LOG_DIR}/app.log",
          "mkdir": true
        }
      }]
    }
  }
}
```

### Log Redaction

Automatically hide sensitive data:

```json
{
  "logger": {
    "redact": {
      "paths": [
        "req.headers.authorization",
        "req.headers.cookie",
        "password",
        "apiKey",
        "token",
        "*.password",
        "*.secret"
      ],
      "censor": "[REDACTED]"
    }
  }
}
```

### Custom Formatters

Create `formatters.js`:
```javascript
module.exports = {
  level(label, number) {
    return { level: label.toUpperCase() }
  },
  bindings(bindings) {
    return {
      pid: bindings.pid,
      host: bindings.hostname,
      service: bindings.service
    }
  },
  log(object) {
    return object
  }
}
```

Reference in config:
```json
{
  "logger": {
    "formatters": {
      "path": "./formatters.js"
    }
  }
}
```

### Programmatic Usage

```javascript
// Access the global logger
const logger = globalThis.platformatic.logger

logger.info('Application started')
logger.debug({ userId: 123 }, 'User login')
logger.error({ err }, 'Request failed')

// Child logger with context
const childLogger = logger.child({
  service: 'payment-api',
  requestId: req.id
})
childLogger.info('Processing payment')
```

---

## OpenTelemetry Tracing

Enable distributed tracing across your microservices.

### Basic Configuration

```json
{
  "telemetry": {
    "serviceName": "my-service",
    "version": "1.0.0",
    "exporter": {
      "type": "otlp",
      "options": {
        "url": "http://localhost:4318/v1/traces"
      }
    }
  }
}
```

### Exporter Types

| Type | Description | Default Endpoint |
|------|-------------|------------------|
| `otlp` | OpenTelemetry Protocol (recommended) | `http://localhost:4318/v1/traces` |
| `zipkin` | Zipkin format | `http://localhost:9411/api/v2/spans` |
| `jaeger` | Jaeger Thrift | `http://localhost:14268/api/traces` |

### OTLP Endpoints

- **HTTP**: `http://localhost:4318/v1/traces`
- **gRPC**: `http://localhost:4317`

### Environment Variables

```bash
# Service identification
OTEL_SERVICE_NAME=my-service
OTEL_SERVICE_VERSION=1.0.0

# Exporter configuration
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer xxx

# Resource attributes
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.namespace=myapp
```

---

## OpenTelemetry Logs

Send logs to OpenTelemetry collectors using `pino-opentelemetry-transport`.

### Installation

```bash
npm install pino-opentelemetry-transport
```

### Configuration

```json
{
  "logger": {
    "transport": {
      "target": "pino-opentelemetry-transport",
      "options": {
        "loggerName": "my-app",
        "serviceVersion": "1.0.0",
        "resourceAttributes": {
          "deployment.environment": "production"
        }
      }
    }
  }
}
```

### Environment Variables

```bash
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4318/v1/logs
OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer xxx
```

---

## Log-Trace Correlation

Automatically inject trace context into logs using `@opentelemetry/instrumentation-pino`.

### Installation

```bash
npm install @opentelemetry/instrumentation-pino
```

### Setup

```javascript
const { PinoInstrumentation } = require('@opentelemetry/instrumentation-pino')

const pinoInstrumentation = new PinoInstrumentation({
  logKeys: {
    traceId: 'trace_id',
    spanId: 'span_id',
    traceFlags: 'trace_flags'
  }
})
```

### Result

Logs within a traced request automatically include:
```json
{
  "level": "info",
  "msg": "Processing request",
  "trace_id": "abc123...",
  "span_id": "def456...",
  "trace_flags": "01"
}
```

---

## Prometheus Metrics

Expose metrics for Prometheus scraping.

### Configuration

```json
{
  "metrics": {
    "port": 9090,
    "hostname": "0.0.0.0",
    "auth": false
  }
}
```

### Default Metrics

Watt automatically exposes:
- `nodejs_*` - Node.js runtime metrics
- `http_request_duration_seconds` - HTTP request latency
- `http_requests_total` - HTTP request count

### Custom Metrics

```javascript
const client = require('prom-client')

// Counter
const requestCounter = new client.Counter({
  name: 'api_requests_total',
  help: 'Total API requests',
  labelNames: ['method', 'endpoint', 'status']
})

// Histogram
const responseTime = new client.Histogram({
  name: 'api_response_time_seconds',
  help: 'API response time',
  labelNames: ['method', 'endpoint'],
  buckets: [0.1, 0.5, 1, 2, 5]
})

// Gauge
const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
})

// Usage
requestCounter.inc({ method: 'GET', endpoint: '/api/users', status: 200 })
responseTime.observe({ method: 'GET', endpoint: '/api/users' }, 0.234)
activeConnections.set(42)
```

### Kubernetes ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: watt-app
spec:
  selector:
    matchLabels:
      app: watt-app
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

---

## Backend-Specific Configuration

### Jaeger

**Docker Compose:**
```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

**watt.json:**
```json
{
  "telemetry": {
    "serviceName": "my-service",
    "exporter": {
      "type": "otlp",
      "options": {
        "url": "http://jaeger:4318/v1/traces"
      }
    }
  }
}
```

### Datadog

**Datadog Agent configuration:**
```yaml
# datadog.yaml
otlp_config:
  receiver:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
      grpc:
        endpoint: 0.0.0.0:4317
```

**watt.json:**
```json
{
  "telemetry": {
    "serviceName": "my-service",
    "exporter": {
      "type": "otlp",
      "options": {
        "url": "http://datadog-agent:4318/v1/traces"
      }
    }
  }
}
```

**Environment:**
```bash
DD_SERVICE=my-service
DD_ENV=production
DD_VERSION=1.0.0
```

### New Relic

**Environment variables:**
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.nr-data.net
OTEL_EXPORTER_OTLP_HEADERS=api-key=YOUR_NEW_RELIC_LICENSE_KEY
OTEL_SERVICE_NAME=my-service
```

**watt.json:**
```json
{
  "telemetry": {
    "serviceName": "my-service",
    "exporter": {
      "type": "otlp",
      "options": {
        "url": "{OTEL_EXPORTER_OTLP_ENDPOINT}/v1/traces",
        "headers": {
          "api-key": "{NEW_RELIC_LICENSE_KEY}"
        }
      }
    }
  }
}
```

### Grafana Stack (Tempo + Loki + Prometheus)

**Docker Compose:**
```yaml
services:
  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo.yaml"]
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

**Logs to Loki (pino-loki):**
```bash
npm install pino-loki
```

```json
{
  "logger": {
    "transport": {
      "target": "pino-loki",
      "options": {
        "host": "http://loki:3100",
        "labels": { "app": "my-service" }
      }
    }
  }
}
```

### AWS CloudWatch / X-Ray

**Using AWS Distro for OpenTelemetry (ADOT):**

```yaml
# ADOT collector config
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  awsxray:
    region: us-east-1

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [awsxray]
```

**watt.json:**
```json
{
  "telemetry": {
    "serviceName": "my-service",
    "exporter": {
      "type": "otlp",
      "options": {
        "url": "http://adot-collector:4318/v1/traces"
      }
    }
  }
}
```

### GCP Cloud Trace

**Environment:**
```bash
GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json
OTEL_EXPORTER_OTLP_ENDPOINT=https://cloudtrace.googleapis.com
```

Or use the OpenTelemetry Collector with GCP exporter.

### Azure Monitor

**Installation:**
```bash
npm install @azure/monitor-opentelemetry-exporter
```

**Environment:**
```bash
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;IngestionEndpoint=https://xxx.in.applicationinsights.azure.com/
```

---

## Multiple Transports

Send logs to multiple destinations:

```json
{
  "logger": {
    "level": "info",
    "transport": {
      "targets": [
        {
          "target": "pino-pretty",
          "level": "info",
          "options": { "colorize": true }
        },
        {
          "target": "pino/file",
          "level": "warn",
          "options": { "destination": "./logs/error.log" }
        },
        {
          "target": "pino-opentelemetry-transport",
          "level": "info",
          "options": { "loggerName": "my-app" }
        },
        {
          "target": "pino-loki",
          "level": "info",
          "options": { "host": "http://loki:3100" }
        }
      ]
    }
  }
}
```

---

## Full Observability Stack (Docker Compose)

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
      - "9090:9090"  # Metrics
    environment:
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3000
      - PLT_SERVER_LOGGER_LEVEL=info
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4318
    depends_on:
      - jaeger
      - prometheus

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "4317:4317"
      - "4318:4318"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9091:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
```

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'watt-app'
    static_configs:
      - targets: ['app:9090']
```

---

## Resources

- [Platformatic Logging Guide](https://docs.platformatic.dev/docs/guides/logging)
- [Distributed Tracing with Platformatic](https://blog.platformatic.dev/distributed-tracing-with-platformatic-and-open-telemetry)
- [Pino Documentation](https://getpino.io/)
- [OpenTelemetry JS](https://opentelemetry.io/docs/languages/js/)
- [pino-opentelemetry-transport](https://github.com/pinojs/pino-opentelemetry-transport)
