# Inside Kafka

_apache kafka · from the log up_

> Almost every Kafka mistake comes from thinking it's a message queue. It isn't. It's a _durable, replayable log_ that many readers walk independently — and once that clicks, the rest is detail.

---

## Contents

1. [It's a log, not a queue](#its-a-log-not-a-queue)
2. [Topics and partitions](#topics-and-partitions)
3. [The producer](#the-producer)
4. [Durability and acks](#durability-and-acks)
5. [The consumer](#the-consumer)
6. [Consumer groups](#consumer-groups)
7. [Offsets and commits](#offsets-and-commits)
8. [Rebalancing](#rebalancing)
9. [Delivery semantics](#delivery-semantics)
10. [Transactions](#transactions)
11. [Serialization](#serialization)
12. [Retention and compaction](#retention-and-compaction)
13. [Spring and Streams](#spring-and-streams)
14. [Configs that matter](#configs-that-matter)
15. [Ways to get hurt](#ways-to-get-hurt)

---

## It's a log, not a queue

A queue hands you a message and forgets it. Kafka appends to a file and remembers — every reader tracks their own position, and nothing is consumed away.

![A partition as an append-only sequence of numbered offsets, with a producer appending at the end and two consumers reading at independent positions](diagrams/inside-kafka-01.svg)

**Reading is not destructive.** A consumer's position is just a number it remembers. Move the number backwards and you replay history — which is why Kafka works for event sourcing, backfills, and adding a new service that needs the last month of traffic.

Three consequences follow immediately, and they explain most of Kafka's design:

- **Many independent consumers, no fan-out cost.** Ten services reading the same topic don't make ten copies. They make ten offsets.
- **Data lives on a clock, not on consumption.** Messages are deleted when they age out (or get compacted), never because someone read them.
- **Order is a property of the log, not the topic.** Which brings us to partitions.

## Topics and partitions

A topic is a name. A **partition** is the actual log. Splitting a topic into partitions is what buys parallelism — and what costs you global ordering.

![A record key hashed to choose one of three partitions, with all records sharing a key landing in the same partition](diagrams/inside-kafka-02.svg)

**The ordering rule, in full:** Kafka guarantees order _within a partition_ and nowhere else. If you need events for one entity processed in sequence, give them all the same key.

> ⚠️ **Partition count is close to permanent**
>
> You can add partitions to a topic, but you cannot remove them — and adding them **changes the hash mapping**, so `user-42` may move from P1 to P4. Records already written stay where they are, and per-key ordering breaks across the boundary. Size partitions for the parallelism you'll need later, not the load you have today.

## The producer

Thread-safe, asynchronous, and batching by default. One instance per application is the normal shape — sharing it across threads is not just allowed, it's the point.

| Property           | Value                  |
| ------------------ | ---------------------- |
| **Thread-safe**    | Yes — share it         |
| **send() returns** | Future, immediately    |
| **Batching**       | batch.size / linger.ms |
| **Idempotence**    | `On by default (3.0+)` |
| **Must call**      | `close() or flush()`   |

```java
            Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer",   StringSerializer.class.getName());
props.put("value.serializer", StringSerializer.class.getName());

// try-with-resources — close() flushes pending batches. Skip it and
// in-flight records are silently lost.
try (Producer<String, String> producer = new KafkaProducer<>(props)) {

    ProducerRecord<String, String> record =
        new ProducerRecord<>("orders", "user-42", "{\"total\":19.99}");

    // Asynchronous — returns before the broker has seen anything.
    producer.send(record, (meta, ex) -> {
        if (ex != null) log.error("send failed", ex);
        else log.info("p{} offset {}", meta.partition(), meta.offset());
    });

    // Synchronous — correct, and roughly 100x slower. Kills batching.
    RecordMetadata m = producer.send(record).get();
}

```

### Batching is where the throughput comes from

The producer does not send one record per call. It accumulates records per partition and ships them as a batch, which is why Kafka can move millions of small messages a second.

```java
            props.put("batch.size",  16384);   // bytes per batch — 16KB default
props.put("linger.ms",   10);      // wait this long to fill a batch (0 default)
props.put("compression.type", "lz4");   // compresses the whole batch
props.put("buffer.memory", 33554432);      // 32MB of unsent records

```

`linger.ms=0` means "send as soon as a thread is free" — which under load still batches, because batches form while the previous send is in flight. Raising it to 5–20ms trades a little latency for a large throughput gain. If `buffer.memory` fills, `send()` blocks, then throws `TimeoutException`.

## Durability and acks

Each partition has one leader and some followers. `acks` decides how many of them must have the record before the producer calls it done — and it is the single most consequential producer setting.

![The three acks settings compared against a leader and two follower replicas, showing what each one risks](diagrams/inside-kafka-03.svg)

**The trap in `acks=all`:** "all in-sync replicas" can mean _one_, if the followers have fallen behind and dropped out of the ISR. Without `min.insync.replicas`, acks=all degrades silently into acks=1.

```java
            // The durable combination. Both halves are required.
props.put("acks", "all");              // producer side
// min.insync.replicas=2        broker/topic side

// With replication.factor=3 and min.insync.replicas=2 you can lose
// one broker and keep writing. Lose two, and the producer gets
// NotEnoughReplicasException — which is correct: it refuses to
// accept a write it cannot make durable.

```

> **Idempotence is on by default now**
>
> Since Kafka 3.0, `enable.idempotence=true` is the default. It stamps each record with a producer id and sequence number so a retried send cannot create a duplicate, and it implies `acks=all`, `retries>0`, and `max.in.flight.requests.per.connection<=5`. If you explicitly set `acks=1`, you are silently turning idempotence off.

## The consumer

A single-threaded poll loop. Unlike the producer, `KafkaConsumer` is **not thread-safe** — one consumer per thread, always.

| Property           | Value                |
| ------------------ | -------------------- |
| **Thread-safe**    | No                   |
| **Model**          | Pull, not push       |
| **poll() returns** | A batch of records   |
| **Heartbeats**     | Background thread    |
| **Liveness**       | max.poll.interval.ms |

```java
            props.put("group.id", "billing-service");
props.put("key.deserializer",   StringDeserializer.class.getName());
props.put("value.deserializer", StringDeserializer.class.getName());
props.put("enable.auto.commit", false);       // commit deliberately
props.put("auto.offset.reset",  "earliest");  // only when no offset exists

try (Consumer<String, String> consumer = new KafkaConsumer<>(props)) {
    consumer.subscribe(List.of("orders"));

    while (running) {
        ConsumerRecords<String, String> records =
            consumer.poll(Duration.ofMillis(100));

        for (ConsumerRecord<String, String> r : records) {
            process(r.key(), r.value());     // must be FAST — see below
        }
        consumer.commitSync();               // after processing = at-least-once
    }
}

```

> ⚠️ **The loop is also the heartbeat for liveness**
>
> Heartbeats run on a background thread, so a slow `process()` won't miss them — but `max.poll.interval.ms` (5 minutes by default) measures the gap between _polls_. Exceed it and the broker assumes you're dead, kicks you out, and reassigns your partitions — while you're still working on them. Either keep the batch fast, or lower `max.poll.records`.

## Consumer groups

A group is a set of consumers sharing one `group.id`. Kafka assigns each partition to **exactly one** consumer in the group — which makes partition count a hard ceiling on parallelism.

![Four partitions distributed across two, four and five consumers, showing that a fifth consumer sits idle](diagrams/inside-kafka-04.svg)

**Scaling out is really scaling partitions.** To double throughput you need double the partitions first; the consumers follow. This is why undersized partition counts are so hard to live with later.

## Offsets and commits

A committed offset says "the group has finished everything before this point". _When_ you commit — before or after processing — is the entire difference between losing messages and duplicating them.

![Committing before processing risks loss on a crash, committing after risks duplicates](diagrams/inside-kafka-05.svg)

**There is no third option without transactions.** Pick at-least-once, then design the handler so running it twice is harmless — an upsert keyed by event id, rather than a blind insert.

```java
            // Auto-commit: every 5s in the background. Convenient, and it can
// both lose and duplicate, because the timer knows nothing about
// whether you finished processing.
props.put("enable.auto.commit", true);

// Manual, synchronous — blocks, retries, throws on failure.
consumer.commitSync();

// Manual, asynchronous — faster, no retry on failure.
consumer.commitAsync((offsets, ex) -> { if (ex != null) log.warn("…", ex); });

// The standard shutdown pattern: async while running, sync at the end.
try {
    while (running) { … consumer.commitAsync(); }
} finally {
    try { consumer.commitSync(); } finally { consumer.close(); }
}

// Per-partition control when you need exact positions
consumer.commitSync(Map.of(
    new TopicPartition("orders", 0),
    new OffsetAndMetadata(lastProcessedOffset + 1)));   // note the +1

```

The `+1` matters: a committed offset is the _next_ record to read, not the last one processed. Committing the offset you just handled makes you reprocess it forever.

## Rebalancing

When a consumer joins, leaves, or is presumed dead, partitions are redistributed. Classically this stopped the whole group — and it's the most common source of Kafka pain in production.

| Trigger                   | What happens                         | Usually because                         |
| ------------------------- | ------------------------------------ | --------------------------------------- |
| A consumer joins          | Partitions redistributed             | Scaling up, or a deploy                 |
| A consumer leaves cleanly | Its partitions reassigned            | `close()` was called                    |
| Heartbeat stops           | Evicted after `session.timeout.ms`   | Crash, GC pause, network                |
| Poll gap too long         | Evicted after `max.poll.interval.ms` | **Slow processing** — the usual culprit |
| Topic gains partitions    | Redistributed                        | Someone rescaled the topic              |

```java
            // Cooperative rebalancing moves only the partitions that must move,
// instead of revoking everything from everyone.
props.put("partition.assignment.strategy",
          CooperativeStickyAssignor.class.getName());

// Hook in to flush state and commit before losing a partition.
consumer.subscribe(List.of("orders"), new ConsumerRebalanceListener() {
    public void onPartitionsRevoked(Collection<TopicPartition> parts) {
        consumer.commitSync();      // last chance before they move
    }
    public void onPartitionsAssigned(Collection<TopicPartition> parts) {
        // warm caches, seek to a custom position, etc.
    }
});

```

Newer Kafka versions are moving to a broker-coordinated protocol (KIP-848) that removes most stop-the-world behaviour entirely. Check what your cluster and client versions actually support before relying on it — the rollout spans several releases.

## Delivery semantics

Three levels, and the honest summary is that most systems should choose the middle one and make their handlers idempotent.

- **At-most-once** — Commit before processing. Never duplicates, sometimes loses. Acceptable for metrics, almost nothing else.

Commit after processing. Never loses, sometimes duplicates. **The default choice.**

Transactions across consume-process-produce. Real, but only within Kafka — and it costs throughput.

> **What "exactly-once" actually covers**
>
> Kafka's exactly-once applies to a **read-process-write cycle inside Kafka**: consume from a topic, produce to another topic, commit offsets, all atomically. It does _not_ extend to your database, your payment provider, or the email you send. The moment your handler touches an external system, you are back to at-least-once plus idempotency — so build for that first.

## Transactions

An atomic write across multiple partitions, with the consumer's offset commit folded into the same transaction.

```java
            props.put("transactional.id", "order-processor-1");   // stable, unique per instance
Producer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    producer.beginTransaction();
    try {
        for (var r : records) {
            producer.send(new ProducerRecord<>("invoices", transform(r.value())));
        }
        // the offsets ride inside the transaction — this is the trick
        producer.sendOffsetsToTransaction(offsetsOf(records), consumer.groupMetadata());
        producer.commitTransaction();
    } catch (KafkaException e) {
        producer.abortTransaction();     // nothing downstream sees it
    }
}

// The reader must opt in, or it will see aborted records too.
consumerProps.put("isolation.level", "read_committed");

```

> ⚠️ **Both halves or neither**
>
> A downstream consumer with the default `isolation.level=read_uncommitted` will happily read records from transactions that were later aborted. Producing transactionally while reading non-transactionally gives you all of the cost and none of the guarantee.

## Serialization

Kafka moves bytes and has no opinion about them. Everything about compatibility is your problem — which is what schema registries exist to solve.

| Format          | Schema               | Good for                       | Cost                               |
| --------------- | -------------------- | ------------------------------ | ---------------------------------- |
| `String / JSON` | None enforced        | Getting started, debugging     | Verbose; breaks silently on change |
| `Avro`          | Registry             | The common production choice   | Needs a schema registry            |
| `Protobuf`      | Registry or compiled | Polyglot shops, gRPC alongside | Build-step coupling                |
| `ByteArray`     | Yours                | Full control, custom framing   | You own every mistake              |

```java
            // A custom serializer is just two methods.
public class OrderSerializer implements Serializer<Order> {
    private final ObjectMapper mapper = new ObjectMapper();

    @Override public byte[] serialize(String topic, Order data) {
        if (data == null) return null;
        try { return mapper.writeValueAsBytes(data); }
        catch (Exception e) { throw new SerializationException(e); }
    }
}

// Schema compatibility is the thing that bites at 2am:
//   BACKWARD  — new readers can read old data   (add optional fields)
//   FORWARD   — old readers can read new data   (remove optional fields)
//   FULL      — both

```

## Retention and compaction

Two cleanup policies with completely different purposes: one throws away old _time_, the other throws away old _versions_.

|                | `cleanup.policy=delete`         | `cleanup.policy=compact`                 |
| -------------- | ------------------------------- | ---------------------------------------- |
| Keeps          | Everything newer than the limit | The latest record per key, forever       |
| Controlled by  | `retention.ms, retention.bytes` | `min.cleanable.dirty.ratio`              |
| Default        | 7 days                          | —                                        |
| Think of it as | An event stream                 | A changelog — a table in disguise        |
| Deletion       | By age                          | By key, via a **tombstone** (null value) |

Compaction is what makes a topic usable as durable state: replay it from the beginning and you get the current value for every key that has ever existed. That's exactly how Kafka Streams backs a `KTable`, and how `__consumer_offsets` itself works.

## Spring and Streams

In practice most Java teams reach one of two layers above the raw client.

### Spring for Apache Kafka

```java
            @KafkaListener(topics = "orders", groupId = "billing-service")
public void onOrder(ConsumerRecord<String, Order> record,
                    Acknowledgment ack) {
    process(record.value());
    ack.acknowledge();          // with AckMode.MANUAL
}

@Autowired KafkaTemplate<String, Order> template;
template.send("orders", order.userId(), order);

// Retries and a dead-letter topic, declaratively
@RetryableTopic(attempts = "3", backoff = @Backoff(delay = 2000))

```

### Kafka Streams

A library, not a cluster — it runs inside your JVM and uses Kafka itself for state and fault tolerance.

```java
            StreamsBuilder b = new StreamsBuilder();

b.<String, Order>stream("orders")
 .filter((k, v) -> v.total() > 100)
 .groupByKey()
 .aggregate(Totals::empty, (k, v, agg) -> agg.plus(v))   // a KTable
 .toStream()
 .to("big-spenders");

// KStream = an event per record.  KTable = the latest value per key.
// exactly-once is one line here:
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, "exactly_once_v2");

```

## Configs that matter

### Producer

| Setting               | Default  | Why you'd change it                                 |
| --------------------- | -------- | --------------------------------------------------- |
| `acks`                | `all`    | Never lower it unless losing data is genuinely fine |
| `enable.idempotence`  | `true`   | Leave it on; setting acks=1 disables it             |
| `linger.ms`           | `0`      | 5–20ms buys a lot of throughput                     |
| `batch.size`          | `16384`  | Raise alongside linger.ms                           |
| `compression.type`    | `none`   | lz4 or zstd — usually a clear win                   |
| `max.in.flight…`      | `5`      | Must stay ≤5 for idempotent ordering                |
| `delivery.timeout.ms` | `120000` | The real end-to-end send deadline                   |

#### Consumer

| Setting                | Default            | Why you'd change it                                  |
| ---------------------- | ------------------ | ---------------------------------------------------- |
| `enable.auto.commit`   | `true`             | Turn it off and commit deliberately                  |
| `auto.offset.reset`    | `latest`           | `earliest` if a new group must see history           |
| `max.poll.records`     | `500`              | Lower it when processing is slow                     |
| `max.poll.interval.ms` | `300000`           | The rebalance trigger everyone hits                  |
| `session.timeout.ms`   | `45000`            | Failure detection speed                              |
| `isolation.level`      | `read_uncommitted` | `read_committed` whenever producers use transactions |

#### Choosing quickly

| I want to…                               | Use                         |
| ---------------------------------------- | --------------------------- |
| Events for one entity must stay in order | `key = entity id`           |
| I must not lose a record                 | `acks=all + min.insync=2`   |
| I need more consumer throughput          | `more partitions`           |
| Duplicates would be harmful              | `idempotent handler`        |
| Atomic consume-process-produce           | `transactions`              |
| The topic is really a lookup table       | `cleanup.policy=compact`    |
| Rebalances keep stalling the group       | `CooperativeStickyAssignor` |
| A poison message blocks the partition    | `dead-letter topic`         |

## Ways to get hurt

### Sharing a consumer across threads

`KafkaProducer` is thread-safe; `KafkaConsumer` is emphatically not. It throws `ConcurrentModificationException` if it catches you, but the checking isn't exhaustive. One consumer per thread, or hand records to a worker pool _and_ take over offset management yourself — because once processing is asynchronous, "commit after processing" no longer means anything.

### Expecting ordering across a topic

Ordering exists only inside a partition. A null key spreads records across all of them, so two events for the same user can be processed out of order by different consumers. If order matters, key by the entity the order is about.

### Slow processing causing endless rebalances

```java
            // 500 records × 2s each = 1000s ≫ max.poll.interval.ms (300s)
// → evicted mid-batch → partitions reassigned → the replacement
//   is just as slow → the group never stabilises.

props.put("max.poll.records", 50);              // smaller bites
props.put("max.poll.interval.ms", 600000);      // or a longer deadline

```

### Committing the offset you just processed

A committed offset is the _next_ record to read. Commit `lastProcessed` instead of `lastProcessed + 1` and that record is redelivered on every restart, forever.

### A poison message parked in front of the partition

One record that always throws blocks everything behind it — partition ordering guarantees it. Without a dead-letter topic or a skip-after-N-attempts rule, that partition stops permanently while the others carry on, which makes the failure look intermittent.

### Forgetting `close()`

The producer buffers. Kill the process without closing and whatever hadn't been flushed is gone — with no error, because `send()` returned successfully long before. Use try-with-resources, and register a shutdown hook for long-lived producers.

### Setting `acks=1` for speed

It looks like a cheap latency win. It also silently disables idempotence and accepts data loss on any leader failover. If the write matters, the cost of `acks=all` is smaller than you expect — the followers replicate in parallel.

> **The short version**
>
> It's a log. Order lives in the partition, parallelism is capped by the partition count, and the offset is just a number you control. Produce with `acks=all`, consume with manual commits after processing, make the handler idempotent, and you have covered most of what goes wrong in production.
