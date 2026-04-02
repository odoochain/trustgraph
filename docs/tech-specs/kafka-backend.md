# Kafka Pub/Sub Backend

## Overview

This spec covers adding Apache Kafka as a third pub/sub backend alongside Pulsar and RabbitMQ. The existing `PubSubBackend` protocol and `CLASS:TOPICSPACE:TOPIC` queue naming scheme apply unchanged. The Kafka backend would be selected via `PUBSUB_BACKEND=kafka`.

## Mapping to Kafka Concepts

### Queue naming

The `CLASS:TOPICSPACE:TOPIC` format maps to Kafka topics:

| Queue ID | Kafka topic |
|----------|-------------|
| `flow:tg:text-completion-request:default` | `tg.flow.text-completion-request.default` |
| `request:tg:config` | `tg.request.config` |
| `response:tg:config` | `tg.response.config` |
| `state:tg:config` | `tg.state.config` (compacted) |

The topicspace becomes the prefix. Dots replace colons. Straightforward.

### Consumer patterns

| Pattern | Kafka mechanism |
|---------|----------------|
| Shared (competing consumers) | Same consumer group ID. Kafka distributes partitions across group members. |
| Exclusive (broadcast) | Unique consumer group ID per consumer. Each gets all partitions = all messages. |

This mirrors the RabbitMQ approach — `consumer_type='shared'` uses a deterministic group ID derived from the subscription name, `consumer_type='exclusive'` uses a unique group ID (e.g. UUID).

### Queue classes

| Class | Kafka configuration |
|-------|-------------------|
| `flow` | Persistent topic, default retention (e.g. 7 days or size-based) |
| `request` | Persistent topic, short retention (e.g. 5 minutes). Kafka has no non-persistent mode. |
| `response` | Same as request — short retention. |
| `state` | Compacted topic (`cleanup.policy=compact`). Retains last value per key. Perfect for config push — new consumers see the latest config immediately. |

### Message properties

Kafka headers (key-value byte arrays) map directly to the properties dict. The `id` property used for request-response correlation becomes a Kafka header.

### Consumer positioning

- `initial_position='earliest'` → `auto.offset.reset=earliest` or `seek_to_beginning()`
- `initial_position='latest'` → `auto.offset.reset=latest` (default)

### Message sizes

All messages are under 5MB after the `stream-document` migration. Kafka's default `max.message.bytes` is 1MB but is commonly increased to 10MB+. A 5MB setting would cover all current message types.

## Negative Acknowledgement

This is the main design challenge. TrustGraph uses `negative_acknowledge()` for two purposes:

1. **Rate-limit retry** — `TooManyRequests` exception causes the message to be negatively acknowledged and redelivered after a delay. The Consumer retries for up to 7200 seconds.
2. **Error handling** — unhandled exceptions cause nack, triggering redelivery.

Kafka has no per-message nack. It uses offset-based commits — you commit the offset of the last successfully processed message, and on restart, consumption resumes from the last committed offset.

### Options

**Option A: Don't commit on failure, seek back.**

On nack, seek the consumer back to the failed message's offset. The message is redelivered immediately. Risk: if the consumer has already fetched later messages, those get redelivered too. With `max.poll.records=1`, this is safe but slower.

```python
def negative_acknowledge(self, message):
    # Seek back to this message's offset for immediate redelivery
    tp = TopicPartition(message.topic(), message.partition())
    self._consumer.seek(tp, message.offset())
```

**Option B: Pause partition and retry in-process.**

On nack, don't seek — let the application retry loop handle it (which it already does for rate limiting). Only commit after success. If the process crashes mid-retry, Kafka redelivers from the last committed offset on restart.

This matches TrustGraph's existing retry behaviour in `handle_one_from_queue` — the rate-limit retry loop already retries in-process before negatively acknowledging. The nack is a last resort that means "I give up, let someone else try."

```python
def negative_acknowledge(self, message):
    # Don't commit — message will be redelivered on restart/rebalance
    pass

def acknowledge(self, message):
    self._consumer.commit(offsets=[...])
```

**Option C: Publish to retry topic.**

On nack, publish the message to a retry topic (e.g. `tg.retry.text-completion-request`). A separate consumer reads from retry topics with a delay. Most complex but most flexible.

### Recommendation

**Option B** is the simplest and matches TrustGraph's actual behaviour. The in-process retry loop handles transient failures (rate limits). The nack is only reached after the retry timeout expires (7200 seconds). At that point, not committing the offset is the right thing — the message will be retried when the consumer restarts or rebalances.

The only downside: if a message is permanently unprocessable, it blocks that partition until the consumer is restarted. This is acceptable for TrustGraph's workload — permanently bad messages are rare, and operator intervention is expected.

## Thread Safety

The `confluent-kafka` Python client is thread-safe. Unlike pika, a single producer or consumer can be used from multiple threads. This means:

- No thread-local connections needed (unlike RabbitMQ backend)
- The per-task consumer model from the Consumer class still works but isn't strictly necessary for thread safety
- A single producer can be shared across threads

This simplifies the implementation significantly compared to the RabbitMQ backend.

## Topic Creation

Kafka topics can be auto-created if `auto.create.topics.enable=true` on the broker. Otherwise, topics need to be created via the admin API before use.

For the `state` class (compacted topics), auto-creation won't set the right cleanup policy. Options:

1. Create compacted topics during init (like `tg-init-trustgraph` creates Pulsar namespaces)
2. Use the Kafka admin API in the backend to create topics with the right config on first use
3. Require manual topic creation for state-class topics

Option 2 is the most user-friendly — the backend creates the topic with `cleanup.policy=compact` when `create_producer` or `create_consumer` is called for a state-class queue.

## Implementation

### Files

- `trustgraph-base/trustgraph/base/kafka_backend.py` — ~250 lines
- `trustgraph-base/trustgraph/base/pubsub.py` — add `kafka` branch to factory, add Kafka CLI args
- `trustgraph-base/pyproject.toml` — add `confluent-kafka` dependency

### Classes

```
KafkaMessage         — wraps confluent_kafka.Message, implements Message protocol
KafkaBackendProducer — wraps confluent_kafka.Producer
KafkaBackendConsumer — wraps confluent_kafka.Consumer
KafkaBackend         — implements PubSubBackend, handles topic mapping and creation
```

### Configuration

Environment variables:
- `KAFKA_BOOTSTRAP_SERVERS` — default `kafka:9092` (container) / `localhost:9092` (standalone)
- `KAFKA_SECURITY_PROTOCOL` — default `PLAINTEXT`
- `KAFKA_SASL_MECHANISM` — for authenticated clusters
- `KAFKA_SASL_USERNAME` / `KAFKA_SASL_PASSWORD`

CLI args via `add_pubsub_args`:
- `--kafka-bootstrap-servers`
- `--kafka-security-protocol`

### Key differences from RabbitMQ backend

| Aspect | RabbitMQ | Kafka |
|--------|----------|-------|
| Thread safety | Thread-local connections | Single shared connection |
| Topology | Topic exchange + queue bindings | Topics only |
| Non-persistent | Non-durable queues | Short retention topics |
| State/config | Named durable queue per subscriber | Compacted topic |
| Nack | `basic_nack(requeue=True)` | Don't commit offset |
| Message delivery | Push (basic_consume callback) | Poll (consumer.poll()) |
| Ordering | Not guaranteed across consumers | Per-partition (not needed) |

### init_trustgraph changes

For Kafka backend, `tg-init-trustgraph` would:
1. Skip Pulsar admin setup (same as RabbitMQ)
2. Optionally create the state topic with compaction enabled
3. Push config via `ConfigClient` (same as all backends)

## Estimated Effort

- Backend implementation: ~250 lines, straightforward mapping
- Factory/args: trivial, same pattern as RabbitMQ
- No framework changes needed (Consumer, Producer, Subscriber unchanged)
- No threading changes needed (confluent-kafka is thread-safe)
- Testing: unit tests for topic mapping + integration test against a broker
- Documentation: update pubsub-abstraction.md

The negative_acknowledge semantics (Option B) require no code beyond a no-op `negative_acknowledge` method and offset-based `acknowledge`.

## Operational Considerations

- Kafka requires ZooKeeper or KRaft for coordination — heavier infrastructure than RabbitMQ
- Partition count must be planned for parallelism (default 1 partition = no competing consumers)
- Compacted topics need monitoring to ensure compaction keeps up
- Consumer group management adds operational surface area

For small-to-medium TrustGraph deployments, RabbitMQ is likely the better choice. Kafka makes sense for deployments that already run Kafka or need very high throughput.
