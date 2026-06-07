# Migrating from node-rdkafka to @platformatic/kafka

This guide covers the main migration paths from [`node-rdkafka`](https://github.com/Blizzard/node-rdkafka) to `@platformatic/kafka`.

## Overview

`node-rdkafka` is a native binding around `librdkafka`. `@platformatic/kafka` is a modern JavaScript Kafka client with no native dependency.

| Aspect | node-rdkafka | @platformatic/kafka |
|---|---|---|
| Dependency model | Native addon, requires librdkafka-compatible build environment | Pure JavaScript |
| Client creation | `new Kafka.Producer()` / `new Kafka.KafkaConsumer()` | `new Producer()` / `new Consumer()` / `new Admin()` |
| Configuration | librdkafka string keys | JavaScript options |
| Connections | Explicit `connect()` plus `ready` events | Lazy connection on first operation, `close()` for cleanup |
| Producer API | `produce(topic, partition, value, key, timestamp, opaque, headers)` | `send({ messages })` |
| Consumer API | Event/callback based `consume()` | `consume()` returns a Node.js `Readable` stream |
| Stream helpers | `Producer.createWriteStream()`, `KafkaConsumer.createReadStream()` | Use `producer.asStream()` and `consumer.consume()` |
| Headers | Plain object or array depending on usage | `Map` or serializer/deserializer output shape |
| Shutdown | `disconnect(callback)` | `await close()` |

## Installation

Use the package manager already used by the project.

```bash
npm uninstall node-rdkafka
npm install @platformatic/kafka
```

For Yarn or pnpm, use the equivalent remove/add commands.

## Configuration Mapping

| node-rdkafka / librdkafka | @platformatic/kafka | Notes |
|---|---|---|
| `metadata.broker.list: 'localhost:9092'` | `bootstrapBrokers: ['localhost:9092']` | Split comma-separated strings into an array |
| `client.id` | `clientId` | Rename |
| `group.id` | `groupId` | Consumer option |
| `socket.timeout.ms` | `timeout` or `requestTimeout` | Choose based on original usage |
| `request.timeout.ms` | `requestTimeout` | Rename |
| `metadata.max.age.ms` | `metadataMaxAge` | Rename |
| `allow.auto.create.topics` | `autocreateTopics` | Rename |
| `enable.auto.commit` | Commit behavior in consumer flow | Prefer explicit commits for migrated code |
| `auto.offset.reset: 'earliest'` | `autoOffsetReset: 'earliest'` | Preserve policy |
| `security.protocol: 'ssl'` | `tls: {}` | Convert SSL options into `tls` |
| `security.protocol: 'sasl_ssl'` | `tls: {}`, `sasl: {...}` | Configure both TLS and SASL |
| `sasl.mechanisms: 'PLAIN'` | `sasl: { mechanism: 'PLAIN' }` | Mechanism values are uppercase |
| `sasl.username` | `sasl.username` | Move into `sasl` object |
| `sasl.password` | `sasl.password` | Move into `sasl` object |
| `compression.codec` | `compression` | Use strings such as `'gzip'`, `'snappy'`, `'lz4'`, `'zstd'` |

Native-only options such as queue tuning, polling intervals, callbacks, statistics intervals, and low-level socket options may not have direct equivalents. Do not drop production-critical options silently; flag them for review.

### Before

```js
const Kafka = require('node-rdkafka')

const producer = new Kafka.Producer({
  'metadata.broker.list': 'localhost:9092',
  'client.id': 'orders-api',
  'request.timeout.ms': 30000,
  'compression.codec': 'gzip'
})
```

### After

```js
import { Producer } from '@platformatic/kafka'

const producer = new Producer({
  bootstrapBrokers: ['localhost:9092'],
  clientId: 'orders-api',
  requestTimeout: 30000
})
```

Pass compression on `send()` when needed.

## Producer Migration

### Client Creation

#### Before

```js
const Kafka = require('node-rdkafka')

const producer = new Kafka.Producer({
  'metadata.broker.list': 'localhost:9092',
  'client.id': 'orders-api'
})

producer.connect()

producer.on('ready', () => {
  producer.produce('orders', null, Buffer.from(JSON.stringify({ id: 42 })), '42')
})
```

#### After

```js
import { Producer, stringSerializers } from '@platformatic/kafka'

const producer = new Producer({
  bootstrapBrokers: ['localhost:9092'],
  clientId: 'orders-api',
  serializers: stringSerializers
})

await producer.send({
  messages: [{
    topic: 'orders',
    key: '42',
    value: JSON.stringify({ id: 42 })
  }]
})
```

The topic moves from the first `produce()` argument to each message object.

### `produce()` Argument Mapping

| node-rdkafka `produce()` argument | @platformatic/kafka message field |
|---|---|
| `topic` | `topic` |
| `partition` | `partition` |
| `value` | `value` |
| `key` | `key` |
| `timestamp` | `timestamp` |
| `headers` | `headers` |
| `opaque` | No direct equivalent |

### Delivery Reports

#### Before

```js
const producer = new Kafka.Producer({
  'metadata.broker.list': 'localhost:9092',
  dr_cb: true
})

producer.on('delivery-report', (error, report) => {
  if (error) {
    console.error(error)
    return
  }

  console.log(report.topic, report.partition, report.offset)
})
```

#### After

```js
const result = await producer.send({
  messages: [{ topic: 'orders', key: '42', value: JSON.stringify({ id: 42 }) }]
})

for (const offset of result.offsets) {
  console.log(offset.topic, offset.partition, offset.offset)
}
```

`@platformatic/kafka` returns send results instead of emitting delivery report events.

### Compression

#### Before

```js
const producer = new Kafka.Producer({
  'metadata.broker.list': 'localhost:9092',
  'compression.codec': 'gzip'
})
```

#### After

```js
await producer.send({
  compression: 'gzip',
  messages: [{ topic: 'orders', value: JSON.stringify({ id: 42 }) }]
})
```

Available algorithms include `'none'`, `'gzip'`, `'snappy'`, `'lz4'`, and `'zstd'`.

## Consumer Migration

### Event-Based Consumer

#### Before

```js
const Kafka = require('node-rdkafka')

const consumer = new Kafka.KafkaConsumer({
  'metadata.broker.list': 'localhost:9092',
  'group.id': 'orders-worker',
  'enable.auto.commit': false
})

consumer.connect()

consumer.on('ready', () => {
  consumer.subscribe(['orders'])
  consumer.consume()
})

consumer.on('data', message => {
  const value = JSON.parse(message.value.toString())
  console.log(value)
  consumer.commitMessage(message)
})
```

#### After

```js
import { Consumer, stringDeserializers } from '@platformatic/kafka'

const consumer = new Consumer({
  bootstrapBrokers: ['localhost:9092'],
  groupId: 'orders-worker',
  deserializers: stringDeserializers
})

const stream = await consumer.consume({ topics: ['orders'] })

for await (const message of stream) {
  const value = JSON.parse(message.value)
  console.log(value)
  await message.commit()
}
```

### Message Shape

| node-rdkafka message | @platformatic/kafka message |
|---|---|
| `message.topic` | `message.topic` |
| `message.partition` | `message.partition` |
| `message.offset` | `message.offset` (`bigint`) |
| `message.value` | `message.value` after deserialization |
| `message.key` | `message.key` after deserialization |
| `message.timestamp` | `message.timestamp` (`bigint`) |
| `consumer.commitMessage(message)` | `await message.commit()` |

### Batch Consumption

`node-rdkafka` can consume batches by passing a number to `consume(number, callback)`. With `@platformatic/kafka`, collect messages from the stream when batch processing is required.

```js
const batch = []

for await (const message of stream) {
  batch.push(message)

  if (batch.length < 100) {
    continue
  }

  await processBatch(batch)
  await Promise.all(batch.map(message => message.commit()))
  batch.length = 0
}
```

## Stream API Migration

### ProducerStream

Prefer direct `producer.send()` calls when stream semantics are not needed. If the original code depends on stream backpressure, batching, or piping, use `producer.asStream()`.

#### Before

```js
const stream = Kafka.Producer.createWriteStream({
  'metadata.broker.list': 'localhost:9092'
}, {}, {
  topic: 'orders'
})

stream.write(Buffer.from(JSON.stringify({ id: 42 })))
```

#### After

```js
import { Producer, ProducerStreamReportModes, stringSerializers } from '@platformatic/kafka'

const producer = new Producer({
  bootstrapBrokers: ['localhost:9092'],
  serializers: stringSerializers
})

const stream = producer.asStream({
  batchSize: 100,
  batchTime: 50,
  reportMode: ProducerStreamReportModes.BATCH
})

stream.on('delivery-report', report => {
  console.log(report)
})

stream.write({ topic: 'orders', value: JSON.stringify({ id: 42 }) })

await stream.close()
await producer.close()
```

`producer.asStream()` returns a writable object-mode `ProducerStream`. It accepts send options except `messages`, plus stream-specific options such as `batchSize`, `batchTime`, `reportMode`, and `highWaterMark`.

Producer streams are tracked by the producer. `await producer.close()` fails while producer streams are active; close streams first or use `await producer.close(true)` in shutdown paths where force-closing active streams is acceptable.

### ConsumerStream

`@platformatic/kafka` consumers already return a `Readable` stream.

#### Before

```js
const stream = Kafka.KafkaConsumer.createReadStream({
  'metadata.broker.list': 'localhost:9092',
  'group.id': 'orders-worker'
}, {}, {
  topics: ['orders']
})

stream.on('data', message => {
  console.log(message.value.toString())
})
```

#### After

```js
const stream = await consumer.consume({ topics: ['orders'] })

stream.on('data', async message => {
  console.log(message.value)
  await message.commit()
})
```

Use async iteration instead of `data` events when possible because it makes backpressure and error handling easier to reason about.

## Metadata and Admin Migration

`node-rdkafka` exposes metadata through producer or consumer clients. Prefer a dedicated `Admin` client with `@platformatic/kafka`.

### Before

```js
producer.getMetadata({ topic: 'orders', timeout: 5000 }, (error, metadata) => {
  if (error) {
    throw error
  }

  console.log(metadata.topics)
})
```

### After

```js
import { Admin } from '@platformatic/kafka'

const admin = new Admin({
  bootstrapBrokers: ['localhost:9092'],
  clientId: 'orders-admin'
})

const metadata = await admin.metadata({ topics: ['orders'] })
console.log(metadata.topics)

await admin.close()
```

If the original code creates topics or checks partition counts, map that logic to the corresponding `Admin` methods and preserve the same failure behavior.

## Error Handling

### Before

```js
producer.on('event.error', error => {
  console.error('Kafka error', error)
})
```

### After

```js
try {
  await producer.send({
    messages: [{ topic: 'orders', value: JSON.stringify({ id: 42 }) }]
  })
} catch (error) {
  console.error('Kafka error', error)
}
```

Handle errors at async operation boundaries. For consumer streams, handle stream errors too.

```js
stream.on('error', error => {
  console.error('Kafka consumer error', error)
})
```

## Shutdown

### Before

```js
producer.disconnect(() => {
  console.log('producer disconnected')
})

consumer.disconnect(() => {
  console.log('consumer disconnected')
})
```

### After

```js
await producer.close()
await consumer.close()
```

Always close producers, consumers, and admin clients during application shutdown.

## Migration Checklist

- Remove `node-rdkafka` and install `@platformatic/kafka`.
- Replace CommonJS imports with ESM imports where the project supports ESM.
- Replace `metadata.broker.list` with `bootstrapBrokers`.
- Replace `group.id` with `groupId`.
- Replace `Kafka.Producer` with `Producer`.
- Replace `Kafka.KafkaConsumer` with `Consumer`.
- Replace `Kafka.Producer.createWriteStream()` with `producer.asStream()` or direct sends.
- Replace `Kafka.KafkaConsumer.createReadStream()` with `consumer.consume()`.
- Replace `ready`, `data`, `event.error`, and delivery report flows with async operation handling.
- Replace `produce()` calls with `send({ messages })`.
- Move the topic into each produced message.
- Choose serializers and deserializers that match the existing payload format.
- Replace `commitMessage()` with `message.commit()`.
- Replace metadata calls with `Admin` APIs.
- Replace `disconnect()` with `close()`.
- Review native-only librdkafka options manually.
- Run tests and at least one produce/consume smoke test against Kafka.
