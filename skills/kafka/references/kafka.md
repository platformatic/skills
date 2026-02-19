# Kafka Integration with Platformatic Watt

Platformatic provides high-performance Kafka libraries for building event-driven microservices with Watt.

---

## Kafka Libraries Overview

| Package | Purpose |
|---------|---------|
| `@platformatic/kafka` | Modern TypeScript Kafka client (producer/consumer) |
| `@platformatic/kafka-hooks` | Kafka-to-HTTP webhooks and request/response patterns |
| `@platformatic/watt-plugin-kafka-health` | Consumer lag monitoring for Watt |
| `@platformatic/kafka-opentelemetry` | OpenTelemetry instrumentation |
| `@platformatic/rdkafka` | Native librdkafka wrapper (high performance) |

---

## @platformatic/kafka

A modern, high-performance, pure TypeScript/JavaScript Kafka client with no native dependencies.

### Features

- **High Performance**: Optimized for speed
- **Pure JavaScript**: No native addons required
- **Type Safety**: Full TypeScript support
- **Streaming Consumers**: Node.js Readable streams
- **Flexible Serialization**: Pluggable serializers/deserializers
- **Connection Management**: Automatic pooling and recovery

### Requirements

- Node.js: 20.19.4+, 22.18.0+, or 24.6.0+
- Kafka: 3.5.0 to 4.0.0

### Producer Example

```typescript
import { Producer, stringSerializers } from '@platformatic/kafka'

const producer = new Producer({
  clientId: 'my-producer',
  bootstrapBrokers: ['localhost:9092'],
  serializers: stringSerializers
})

await producer.connect()

await producer.send({
  messages: [{
    topic: 'events',
    key: 'user-123',
    value: JSON.stringify({ name: 'John', action: 'login' })
  }]
})

await producer.close()
```

### Consumer Example

```typescript
import { Consumer, stringDeserializers } from '@platformatic/kafka'

const consumer = new Consumer({
  groupId: 'my-consumer-group',
  bootstrapBrokers: ['localhost:9092'],
  deserializers: stringDeserializers
})

await consumer.connect()

const stream = await consumer.consume({ topics: ['my-topic'] })

// Async iteration
for await (const message of stream) {
  console.log('Received:', message.value)
  await message.commit()
}
```

### Event-Based Consumer

```typescript
const stream = await consumer.consume({ topics: ['my-topic'] })

stream.on('data', async (message) => {
  console.log('Received:', message.value)
  await message.commit()
})

stream.on('error', (error) => {
  console.error('Consumer error:', error)
})
```

---

## @platformatic/kafka-hooks

Bridges Kafka and HTTP services, enabling webhook patterns and request/response over Kafka.

### Installation with Watt

```bash
npx wattpm@latest create
# Select @platformatic/kafka-hooks from the list
```

### Configuration (platformatic.json)

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/kafka-hooks/1.0.0.json",
  "brokers": ["localhost:9092"],
  "topics": [
    {
      "topic": "orders",
      "url": "http://api.plt.local/webhooks/orders",
      "method": "POST",
      "retries": 3,
      "retryDelay": 1000,
      "dlq": "orders-dlq"
    },
    {
      "topic": "notifications",
      "url": "http://notification-service.plt.local/notify",
      "method": "POST",
      "dlq": "notifications-dlq"
    }
  ],
  "requestResponse": [
    {
      "path": "/api/process/:id",
      "requestTopic": "process-requests",
      "responseTopic": "process-responses",
      "timeout": 30000
    }
  ],
  "consumer": {
    "groupId": "kafka-hooks-consumer"
  },
  "concurrency": 10
}
```

### Architecture Patterns

#### 1. Webhook Pattern (Kafka → HTTP)

Kafka messages are automatically forwarded to HTTP endpoints:

```
┌─────────┐    ┌─────────────┐    ┌─────────────┐
│  Kafka  │───▶│ kafka-hooks │───▶│ HTTP Service│
│  Topic  │    │   (Watt)    │    │             │
└─────────┘    └─────────────┘    └─────────────┘
                      │
                      ▼ (on failure)
               ┌─────────────┐
               │     DLQ     │
               │    Topic    │
               └─────────────┘
```

Configuration:
```json
{
  "topics": [
    {
      "topic": "source-topic",
      "url": "http://target-service/webhook",
      "method": "POST",
      "retries": 3,
      "retryDelay": 1000,
      "dlq": "source-topic-dlq"
    }
  ]
}
```

#### 2. Request/Response Pattern (HTTP → Kafka → HTTP)

HTTP clients make requests through Kafka with correlation IDs:

```
┌──────────┐    ┌─────────────┐    ┌───────────┐    ┌──────────┐
│  Client  │───▶│ kafka-hooks │───▶│  Request  │───▶│  Worker  │
│  (HTTP)  │    │   (Watt)    │    │   Topic   │    │ Service  │
└──────────┘    └──────┬──────┘    └───────────┘    └────┬─────┘
      ▲                │                                  │
      │                │           ┌───────────┐          │
      │                └───────────│  Response │◀─────────┘
      │                            │   Topic   │
      └────────────────────────────└───────────┘
```

Configuration:
```json
{
  "requestResponse": [
    {
      "path": "/api/compute/:taskId",
      "requestTopic": "compute-requests",
      "responseTopic": "compute-responses",
      "timeout": 30000
    }
  ]
}
```

#### 3. HTTP Publishing (HTTP → Kafka)

Publish messages to Kafka via HTTP:

```bash
curl -X POST http://localhost:3042/topics/my-topic \
  -H "Content-Type: application/json" \
  -H "x-plt-kafka-hooks-key: message-key" \
  -d '{"event": "user_signup", "userId": "123"}'
```

### Dead Letter Queue (DLQ)

Failed messages are routed to DLQ topics with metadata:

```json
{
  "originalTopic": "orders",
  "originalMessage": "base64-encoded-content",
  "error": "Connection refused",
  "retryCount": 3,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

Disable DLQ for a topic:
```json
{
  "topic": "logs",
  "url": "http://logger/ingest",
  "dlq": false
}
```

---

## Watt Multi-Service with Kafka

### Project Structure

```
kafka-app/
├── watt.json
├── package.json
├── docker-compose.yml
└── web/
    ├── gateway/
    │   └── platformatic.json
    ├── kafka-hooks/
    │   └── platformatic.json      # Kafka-hooks config
    ├── api/
    │   └── platformatic.json
    └── worker/
        └── src/
            └── consumer.ts        # Kafka consumer
```

### Root watt.json

```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/runtime/3.0.0.json",
  "entrypoint": "gateway",
  "web": [
    { "id": "gateway", "path": "web/gateway" },
    { "id": "kafka-hooks", "path": "web/kafka-hooks" },
    { "id": "api", "path": "web/api" },
    { "id": "worker", "path": "web/worker" }
  ],
  "runtime": {
    "logger": { "level": "{PLT_SERVER_LOGGER_LEVEL}" },
    "server": {
      "hostname": "{PLT_SERVER_HOSTNAME}",
      "port": "{PORT}"
    }
  }
}
```

### Kafka-Hooks Service Config

`web/kafka-hooks/platformatic.json`:
```json
{
  "$schema": "https://schemas.platformatic.dev/@platformatic/kafka-hooks/1.0.0.json",
  "brokers": ["{KAFKA_BROKERS}"],
  "topics": [
    {
      "topic": "orders-created",
      "url": "http://api.plt.local/webhooks/orders",
      "method": "POST",
      "dlq": "orders-created-dlq"
    },
    {
      "topic": "notifications",
      "url": "http://worker.plt.local/notify",
      "method": "POST"
    }
  ],
  "requestResponse": [
    {
      "path": "/async/process",
      "requestTopic": "process-requests",
      "responseTopic": "process-responses",
      "timeout": 60000
    }
  ],
  "consumer": {
    "groupId": "watt-kafka-hooks"
  },
  "concurrency": 20
}
```

---

## Consumer Lag Monitoring

Use `@platformatic/watt-plugin-kafka-health` to monitor consumer lag:

```json
{
  "plugins": [
    {
      "package": "@platformatic/watt-plugin-kafka-health",
      "options": {
        "brokers": ["{KAFKA_BROKERS}"],
        "groupId": "my-consumer-group",
        "lagThreshold": 1000,
        "checkInterval": 30000
      }
    }
  ]
}
```

Health endpoint reports lag status:
```json
{
  "status": "healthy",
  "lag": {
    "my-topic": {
      "partition-0": 50,
      "partition-1": 32
    }
  }
}
```

---

## OpenTelemetry Integration

Add tracing with `@platformatic/kafka-opentelemetry`:

```typescript
import { trace } from '@opentelemetry/api'
import { KafkaInstrumentation } from '@platformatic/kafka-opentelemetry'

const instrumentation = new KafkaInstrumentation()
instrumentation.enable()

// Traces are automatically created for produce/consume operations
```

---

## Docker Compose for Development

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3042:3042"
    environment:
      - KAFKA_BROKERS=kafka:9092
      - PLT_SERVER_HOSTNAME=0.0.0.0
      - PORT=3042
    depends_on:
      kafka:
        condition: service_healthy

  kafka:
    image: apache/kafka:3.9.0
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - kafka_data:/var/lib/kafka/data

volumes:
  kafka_data:
```

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `KAFKA_BROKERS` | Comma-separated broker list | `kafka:9092,kafka2:9092` |
| `KAFKA_GROUP_ID` | Consumer group ID | `my-app-consumers` |
| `KAFKA_CLIENT_ID` | Client identifier | `my-app` |

---

## Resources

- [@platformatic/kafka](https://github.com/platformatic/kafka)
- [@platformatic/kafka-hooks](https://github.com/platformatic/kafka-hooks)
- [Kafka-hooks Blog Post](https://blog.platformatic.dev/introducing-platformatickafka-hooks-bridging-kafka-and-http-with-ease)
- [@platformatic/watt-plugin-kafka-health](https://github.com/platformatic/watt-plugin-kafka-health)
- [@platformatic/kafka-opentelemetry](https://github.com/platformatic/kafka-opentelemetry)
