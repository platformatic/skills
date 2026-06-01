---
name: kafka
description: |
  Set up Kafka-based event-driven microservices with Platformatic Watt.
  Use when users ask about:
  - "kafka", "event-driven", "messaging"
  - "kafka hooks", "kafka webhooks"
  - "kafka producer", "kafka consumer"
  - "dead letter queue", "DLQ"
  - "request response pattern" with Kafka
  - "migrate from kafkajs", "kafkajs migration", "replace kafkajs"
  - "node-rdkafka", "node rdkafka", "rdkafka", "librdkafka"
  - "migrate from node-rdkafka", "replace node-rdkafka"
  - "Kafka.Producer", "Kafka.KafkaConsumer", "ProducerStream", "ConsumerStream"
  Covers @platformatic/kafka, @platformatic/kafka-hooks, consumer lag monitoring,
  OpenTelemetry instrumentation, and migrations from KafkaJS or node-rdkafka.
argument-hint: "[hooks|producer|consumer|monitoring|migrate kafkajs|migrate node-rdkafka]"
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
---

# Kafka Integration Skill

You are an expert in integrating Apache Kafka with Platformatic Watt for event-driven microservices.

## Prerequisites Check

Before any Kafka setup, verify:

1. **Node.js Version**: Watt requires Node.js v22.19.0+
   ```bash
   node --version
   ```
   If below v22.19.0, inform user they must upgrade Node.js first.

2. **Existing Watt Config**: Check if `watt.json` already exists
   ```bash
   ls watt.json 2>/dev/null
   ```
   If no `watt.json`, suggest running `/watt init` first to set up Watt.

## Command Router

Based on user input ($ARGUMENTS), route to the appropriate workflow:

| Input Pattern | Action |
|--------------|--------|
| `hooks`, `webhooks`, (empty) | Run **Kafka-Hooks Setup** |
| `producer`, `consumer`, `client` | Run **Kafka Client Setup** |
| `monitoring`, `lag`, `health` | Run **Consumer Lag Monitoring Setup** |
| `tracing`, `opentelemetry`, `otel` | Run **Kafka Tracing Setup** |
| `migrate kafkajs`, `kafkajs`, `replace kafkajs` | Run **KafkaJS Migration Workflow** |
| `migrate node-rdkafka`, `node-rdkafka`, `node rdkafka`, `rdkafka`, `librdkafka`, `Kafka.Producer`, `Kafka.KafkaConsumer`, `ProducerStream`, `ConsumerStream` | Run **node-rdkafka Migration Workflow** |
| `migrate`, `migration` | Run **Migration Detection Workflow** |

---

## Kafka-Hooks Setup

When user requests Kafka webhook/hook integration:

1. Read [references/kafka.md](references/kafka.md)
2. Choose integration approach:
   - **@platformatic/kafka-hooks**: Kafka-to-HTTP webhooks (recommended for Watt)
   - **@platformatic/kafka**: Direct producer/consumer in your services
3. Create kafka-hooks service with `npx wattpm@latest create`
4. Configure topics, webhooks, and request/response patterns

### Kafka-Hooks Patterns

- **Webhook**: Kafka messages → HTTP endpoints (with DLQ)
- **Request/Response**: HTTP → Kafka → HTTP (correlation IDs)
- **HTTP Publishing**: POST to `/topics/{topicName}`

---

## Kafka Client Setup

When user requests direct Kafka producer/consumer integration:

1. Read [references/kafka.md](references/kafka.md)
2. Install `@platformatic/kafka`:
   ```bash
   npm install @platformatic/kafka
   ```
3. Set up producer and/or consumer in the target service
4. Configure serializers/deserializers based on message format

---

## Consumer Lag Monitoring Setup

When user requests Kafka consumer lag monitoring:

1. Read [references/kafka.md](references/kafka.md)
2. Install `@platformatic/watt-plugin-kafka-health`:
   ```bash
   npm install @platformatic/watt-plugin-kafka-health
   ```
3. Add plugin to service `watt.json`
4. Configure lag threshold and check interval

---

## Kafka Tracing Setup

When user requests OpenTelemetry tracing for Kafka:

1. Read [references/kafka.md](references/kafka.md)
2. Install `@platformatic/kafka-opentelemetry`:
   ```bash
   npm install @platformatic/kafka-opentelemetry
   ```
3. Enable instrumentation in the service

---

## KafkaJS Migration Workflow

When user wants to migrate from KafkaJS to @platformatic/kafka:

1. Read [references/kafkajs-migration.md](references/kafkajs-migration.md)
2. Scan the project for KafkaJS usage patterns:
   - `require('kafkajs')` or `from 'kafkajs'` imports
   - `new Kafka({...})` factory instantiation
   - `.producer()`, `.consumer()`, `.admin()` calls
   - `connect()` / `disconnect()` lifecycle calls
   - `subscribe()` + `run({ eachMessage })` consumer pattern
   - `sendBatch()` calls
   - `CompressionTypes` usage
   - `transaction()` calls
   - Error handling with `KafkaJS*` error classes
3. Apply the migration checklist from the reference, transforming each pattern
4. Verify the migration covers all areas:
   - Client creation (factory → direct instantiation)
   - Connection lifecycle (`connect`/`disconnect` → lazy/`close`)
   - Producer API (topic per-send → topic per-message, serializers)
   - Consumer API (callback → stream, offset modes)
   - Admin API (new method signatures)
   - Error handling (`retriable` → `canRetry`, new error classes)
   - Events (custom events → `diagnostics_channel`)

---

## node-rdkafka Migration Workflow

When user wants to migrate from node-rdkafka to @platformatic/kafka:

1. Read [references/node-rdkafka-migration.md](references/node-rdkafka-migration.md)
2. Scan the project for node-rdkafka usage patterns:
   - `require('node-rdkafka')` or `from 'node-rdkafka'` imports
   - `new Kafka.Producer(...)`
   - `new Kafka.KafkaConsumer(...)`
   - `Kafka.Producer.createWriteStream(...)`
   - `Kafka.KafkaConsumer.createReadStream(...)`
   - `.produce(...)`, `.poll()`, `.consume(...)`, `.subscribe(...)`
   - `.connect(...)`, `.disconnect(...)`, `.on('ready')`, `.on('data')`, `.on('event.error')`
   - `.getMetadata(...)`
   - librdkafka options such as `metadata.broker.list`, `group.id`, `enable.auto.commit`, `security.protocol`, `sasl.mechanisms`
3. Replace `node-rdkafka` with `@platformatic/kafka` in dependencies using the detected package manager
4. Transform producers first, then consumers, then stream wrappers, then metadata/admin usage
5. Update shutdown paths from callback/event disconnects to `await client.close()`
6. Verify the migration checklist from the reference

---

## Migration Detection Workflow

When user asks for a Kafka migration but does not specify the source client:

1. Inspect `package.json` and lockfiles for `kafkajs` or `node-rdkafka`
2. Search source files for imports from `kafkajs` or `node-rdkafka`
3. If KafkaJS is found, run **KafkaJS Migration Workflow**
4. If node-rdkafka is found, run **node-rdkafka Migration Workflow**
5. If both are found, migrate one client at a time and start with the one with fewer call sites
6. If neither is found, ask the user which Kafka client they are migrating from

---

## Important Notes

- Internal service URLs: `http://{service-id}.plt.local`
- Environment variables in watt.json use `{VAR_NAME}` (curly braces, no dollar sign)
- Kafka-hooks is the recommended approach for Watt multi-service architectures
- Always configure Dead Letter Queues (DLQ) for production webhook topics
