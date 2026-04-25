# Event-Driven Payment Processing with Apache Kafka in a Digital Banking Context

**Fabio Fasciglione** · Senior Software Engineer  
April 2026

---

## Abstract

This paper presents the design and implementation of a cloud-native, event-driven payment processing system built on Apache Kafka and Spring Boot 3. The system models a core banking workflow composed of three independent microservices — payment initiation, fraud detection, and ledger recording — communicating exclusively through Kafka topics. The architecture addresses four fundamental requirements of high-throughput financial systems: guaranteed per-account message ordering, exactly-once producer semantics, non-blocking anomaly routing via Dead Letter Topics, and real-time consumer lag observability. A comparative analysis against RabbitMQ is provided to motivate key architectural decisions. Empirical results demonstrate sustained producer throughput exceeding 800 messages per second on commodity hardware, with consumer lag converging to zero under normal operating conditions.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Motivation: Kafka vs Traditional Message Brokers](#2-motivation-kafka-vs-traditional-message-brokers)
3. [System Architecture](#3-system-architecture)
4. [Topic Design and Partitioning Strategy](#4-topic-design-and-partitioning-strategy)
5. [Producer Guarantees: Idempotence and Durability](#5-producer-guarantees-idempotence-and-durability)
6. [Consumer Group Topology](#6-consumer-group-topology)
7. [Dead Letter Topic Pattern](#7-dead-letter-topic-pattern)
8. [Observability: Consumer Lag as a Bottleneck Indicator](#8-observability-consumer-lag-as-a-bottleneck-indicator)
9. [Real-Time Dashboard via Server-Sent Events](#9-real-time-dashboard-via-server-sent-events)
10. [Performance Evaluation](#10-performance-evaluation)
11. [Design Trade-offs and Limitations](#11-design-trade-offs-and-limitations)
12. [Conclusions](#12-conclusions)
13. [References](#13-references)

---

## 1. Introduction

Digital banking systems face a fundamental tension between consistency and throughput. A payment must be processed atomically — fraud-checked, ledger-recorded, and acknowledged — yet the volume of concurrent transactions in modern retail banking (often exceeding 10,000 transactions per second at peak) makes synchronous, point-to-point integration impractical. A payment API that sequentially calls a fraud service and then a ledger service introduces cascading latency and tight coupling: a slowdown in either downstream service degrades the entire pipeline.

Apache Kafka addresses this tension through the log abstraction: instead of routing messages directly between services, a producer appends events to a durable, partitioned log. Each consumer group reads from that log independently and at its own pace. The log is not emptied after consumption — it persists for a configurable retention period, enabling replay, audit, and decoupled scaling.

This paper documents a production-inspired implementation of such a system. While simplified for clarity (no persistent storage, no authentication), the architectural patterns demonstrated — partitioned ordering, idempotent production, consumer group isolation, Dead Letter Topic routing, and lag-based observability — are directly applicable to production banking environments.

The full source code is available in a private repository. This document serves as the architectural reference for the public showcase.

---

## 2. Motivation: Kafka vs Traditional Message Brokers

### 2.1 The RabbitMQ Model

RabbitMQ implements the Advanced Message Queuing Protocol (AMQP). Messages are routed from exchanges to queues; once a consumer acknowledges a message, it is removed from the queue. This model is well-suited to task distribution (work queues) and request-reply patterns, but presents limitations in banking contexts:

- **No replay.** A message consumed and acknowledged is gone. Reconstructing the state of an account requires a separate audit database.
- **Fan-out requires explicit configuration.** To deliver the same payment event to both a fraud service and a ledger service, an operator must configure a fanout exchange with two bound queues — one per subscriber. Adding a third subscriber (e.g., a regulatory reporting service) requires infrastructure changes.
- **No native per-entity ordering.** With multiple consumers on the same queue, message ordering is not guaranteed. Ensuring that credits and debits for the same account are applied in sequence requires consumer-side locking or dedicated queues per account — neither of which scales.

### 2.2 The Kafka Log Model

Kafka models a topic as an ordered, immutable, partitioned log stored on disk. Consumers read from the log by tracking an offset — a pointer to their position. The log is not modified by consumption.

| Property | Kafka | RabbitMQ |
|---|---|---|
| **Message persistence** | Disk log, configurable retention (default 7 days) | Deleted on acknowledgment |
| **Replay** | Any consumer group rewinds offset to re-read history | Not possible without separate storage |
| **Multiple subscribers** | Each consumer group reads full log independently | Fanout exchange + dedicated queue per subscriber |
| **Per-entity ordering** | Partition by entity key (e.g. `accountId`) | No native guarantee with concurrent consumers |
| **Peak throughput** | ~1M msg/s with batching and zero-copy I/O [1] | ~100K–500K msg/s depending on configuration |
| **Consumer lag visibility** | Native via `AdminClient.listOffsets` | Queue depth only; no per-consumer breakdown |
| **Exactly-once production** | Native with `enable.idempotence=true` | At-least-once; exactly-once requires app-level keys |
| **Regulatory audit trail** | Inherent in log retention | Requires external audit sink |

### 2.3 Why Kafka for Banking

The retained log directly maps to two regulatory requirements common in banking:

1. **Audit trail.** The complete history of payment events is preserved in Kafka for the duration of the retention period without additional infrastructure.
2. **Event replay for reconciliation.** When a downstream system (e.g., ledger) fails and recovers, it can re-consume events from its last committed offset rather than relying on manual data reloading.

These properties, combined with the ability to add consumer groups without modifying producer code, make Kafka the natural choice for event-driven banking systems.

---

## 3. System Architecture

The system comprises three Spring Boot microservices communicating through four Kafka topics:

```
REST Client
     │
     │  POST /api/payments
     ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        payment-service (:8080)                         │
│  ┌──────────────────┐   ┌─────────────────┐   ┌────────────────────┐  │
│  │ PaymentController│   │PaymentProducerSvc│   │  MetricsController │  │
│  │ POST /payments   │──►│ KafkaTemplate    │   │  AdminClient lag   │  │
│  │ POST /bulk       │   │ key = accountId  │   │  GET /metrics/lag  │  │
│  └──────────────────┘   └────────┬────────┘   └────────────────────┘  │
│                                  │                                      │
│  ┌──────────────────────────────►│◄──────────────────────────────────┐ │
│  │ DashboardConsumer             │                    SseService      │ │
│  │ payment.processed             │                    CopyOnWriteList │ │
│  │ payment.failed                │                    SSE broadcast   │ │
│  │ ledger.recorded               │                                    │ │
│  └───────────────────────────────┼────────────────────────────────────┘ │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │
                    topic: payment.initiated
                    3 partitions · key = accountId
                                   │
               ┌───────────────────┴───────────────────┐
               ▼                                       ▼
┌──────────────────────────┐           ┌──────────────────────────────┐
│  fraud-detection-service │           │       ledger-service         │
│          (:8081)         │           │           (:8082)            │
│  group: fraud-consumers  │           │   group: ledger-consumers    │
│  concurrency: 3          │           │   concurrency: 3             │
│  poll: 50 records        │           │   poll: 20 records           │
│  latency: 50–150ms       │           │   latency: 80–200ms          │
│                          │           │                              │
│  amount ≤ €10,000 ──────►│           │   always publishes to        │
│    → payment.processed   │           │   → ledger.recorded          │
│                          │           │                              │
│  amount > €10,000 ──────►│           └──────────────────────────────┘
│    → payment.failed(DLT) │
└──────────────────────────┘
```

Each service is an independently deployable unit. The only shared infrastructure is the Kafka broker. There are no synchronous HTTP calls between services.

---

## 4. Topic Design and Partitioning Strategy

### 4.1 Topic Configuration

| Topic | Partitions | Replication | Producer | Consumers |
|---|---|---|---|---|
| `payment.initiated` | **3** | 1 | payment-service | fraud-consumers, ledger-consumers |
| `payment.processed` | 1 | 1 | fraud-detection-service | dashboard-consumers |
| `payment.failed` | 1 | 1 | fraud-detection-service | dashboard-consumers |
| `ledger.recorded` | 1 | 1 | ledger-service | dashboard-consumers |

### 4.2 Partitioning by Account ID

The `payment.initiated` topic uses 3 partitions. The producer keys each message with the `accountId`:

```java
kafkaTemplate.send("payment.initiated", accountId, event);
```

Kafka's default partitioner applies `murmur2(key) % numPartitions`, deterministically assigning all messages with the same `accountId` to the same partition. This guarantees that payment events for a given account are always processed in the order they were produced — a mandatory invariant for correct ledger accounting (a debit cannot be applied before a credit that funds it).

### 4.3 Why Three Partitions

Partition count determines the maximum parallelism for a consumer group: a consumer group can have at most one active thread per partition. Three partitions allow both `fraud-consumers` and `ledger-consumers` to scale to three concurrent threads each, providing a 3× throughput multiplier over a single-partition design without sacrificing ordering per account.

---

## 5. Producer Guarantees: Idempotence and Durability

Payment processing has zero tolerance for duplicate messages. A network fault that causes a producer to retry a `send()` operation without idempotency guarantees results in duplicate payment events — a critical bug.

The producer is configured as follows:

```java
config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // exactly-once at broker
config.put(ProducerConfig.ACKS_CONFIG, "all");               // wait for all in-sync replicas
config.put(ProducerConfig.RETRIES_CONFIG, 3);                // retry on transient failure
config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);         // 16KB batch for throughput
config.put(ProducerConfig.LINGER_MS_CONFIG, 5);              // 5ms wait to fill batch
```

With `enable.idempotence=true`, the Kafka broker assigns each producer a Producer ID (PID) and tracks a sequence number per partition. If a duplicate message arrives (same PID + sequence), the broker silently discards it. This provides exactly-once semantics at the broker level without requiring two-phase commit or application-level deduplication.

`acks=all` ensures the broker does not acknowledge a write until all in-sync replicas have persisted the message. Combined with `retries=3`, this provides durability against broker failures without compromising the idempotency guarantee.

---

## 6. Consumer Group Topology

### 6.1 Independent Consumption

`fraud-detection-service` (group `fraud-consumers`) and `ledger-service` (group `ledger-consumers`) both subscribe to `payment.initiated`. Each group maintains an independent offset per partition in the internal Kafka topic `__consumer_offsets`. The two groups have no knowledge of each other; adding a third group (e.g., for regulatory reporting) requires no changes to existing services or producer code.

This is the defining advantage over queue-based brokers: the same event is delivered to all subscriber groups exactly once per group, with no duplication of the payload and no coordination between subscribers.

### 6.2 Concurrent Consumer Threads

Both services configure `concurrency = 3`, matching the partition count:

```java
factory.setConcurrency(3);
// Result: 3 KafkaMessageListenerContainer instances,
// each assigned to one partition of payment.initiated
```

Spring Kafka creates one `ConcurrentMessageListenerContainer` per listener, which internally spawns 3 threads. Thread-to-partition assignment is managed by Kafka's group coordinator and is rebalanced automatically if a consumer instance joins or leaves.

### 6.3 Intentional Throughput Asymmetry

The two consumers are configured with different processing latencies:

| Service | Simulated latency | Represents |
|---|---|---|
| `fraud-detection-service` | 50–150ms | ML fraud model inference |
| `ledger-service` | 80–200ms | Relational database write |

This asymmetry makes `ledger-consumers` the observable bottleneck: under bulk load, its consumer lag grows faster and drains slower than `fraud-consumers`. This behavior is intentionally demonstrable via the dashboard's lag visualization.

---

## 7. Dead Letter Topic Pattern

Payments exceeding €10,000 are classified as potentially fraudulent and must not be processed by the normal approval flow. However, blocking the consumer thread to handle such cases would reduce throughput for all other payments on the same partition.

The Dead Letter Topic (DLT) pattern resolves this: the fraud consumer publishes the anomalous event to a dedicated topic (`payment.failed`) and returns immediately, allowing the next record to be consumed without delay:

```java
if (event.amount().compareTo(FRAUD_THRESHOLD) > 0) {
    kafkaTemplate.send("payment.failed", event.accountId(),
        new PaymentFailedEvent(event.paymentId(), "AMOUNT_EXCEEDS_THRESHOLD", ...));
    // consumer returns — next record is processed immediately
} else {
    kafkaTemplate.send("payment.processed", event.accountId(), ...);
}
```

The `payment.failed` topic functions as a quarantine zone. In a production system, a separate DLT processor would:
- Alert the fraud operations team
- Trigger a manual review workflow
- Optionally re-inject the event into `payment.initiated` after manual clearance

The key property is that the main consumer's throughput is unaffected by the volume of flagged payments.

---

## 8. Observability: Consumer Lag as a Bottleneck Indicator

### 8.1 Definition

Consumer lag quantifies the processing backlog for a consumer group on a given partition:

```
lag(group, partition) = end_offset(partition) − committed_offset(group, partition)
```

A lag of zero means the consumer is current. A growing lag signals that the producer is outpacing the consumer — a bottleneck. The partition granularity of lag allows pinpointing which partition (and therefore which account range, given the partitioning strategy) is under pressure.

### 8.2 Implementation via AdminClient

```java
// MetricsController.java
Map<TopicPartition, OffsetAndMetadata> committed = adminClient
    .listConsumerGroupOffsets(group)
    .partitionsToOffsetAndMetadata().get();

Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> endOffsets = adminClient
    .listOffsets(latestRequest).all().get();

long lag = endOffsets.get(tp).offset() - committed.get(tp).offset();
```

The `AdminClient` queries the broker directly, without consuming any messages. This endpoint is polled every 2 seconds by the dashboard.

### 8.3 Scaling Response

When lag is consistently non-zero, the remediation path is:

1. **Increase consumer concurrency** — up to the partition count
2. **Increase partition count** — requires consumer group rebalance
3. **Optimize consumer processing** — reduce per-message latency (e.g., batch DB writes)

The dashboard makes this progression visually apparent: after a bulk send of 400 payments, `ledger-consumers` shows a higher peak lag and a slower drain rate than `fraud-consumers`, directly illustrating the DB write as the system bottleneck.

---

## 9. Real-Time Dashboard via Server-Sent Events

The `payment-service` exposes a live event feed using Spring's `SseEmitter`. The server pushes named events to all connected browser clients without polling:

```java
// SseService.java
public void broadcast(String eventName, Map<String, Object> data) {
    for (SseEmitter emitter : emitters) {
        emitter.send(SseEmitter.event().name(eventName).data(data));
    }
}
```

The browser subscribes with the native `EventSource` API and dispatches named events to separate handlers:

```javascript
const evtSource = new EventSource('/api/events/stream');
evtSource.addEventListener('payment-initiated', e => { /* update feed */ });
evtSource.addEventListener('fraud-result',      e => { /* update counters */ });
evtSource.addEventListener('ledger-result',     e => { /* update ledger column */ });
```

The dashboard provides the following observability surfaces:

| Panel | Data source | Update frequency |
|---|---|---|
| Live Event Feed | SSE push from `DashboardConsumer` | Real-time |
| Consumer Lag bars | `GET /api/metrics/lag` (AdminClient) | Every 2 seconds |
| Throughput counter | Producer internal `AtomicLong` | Every 2 seconds |
| Approved / DLT counters | SSE event count in browser | Real-time |

<!-- ADD DASHBOARD SCREENSHOT HERE -->

---

## 10. Performance Evaluation

### 10.1 Test Environment

- **Hardware:** Intel Core i7, 16GB RAM, Windows 11
- **Kafka:** Single broker, 1 replica, `confluentinc/cp-kafka:7.5.0`
- **JVM:** Eclipse Adoptium JDK 25.0.2
- **Load:** `POST /api/payments/bulk?count=400`

### 10.2 Results

Watch the demo video here:

[![Watch the demo](https://img.youtube.com/vi/P1z9_bcpGBk/maxresdefault.jpg)](https://youtu.be/P1z9_bcpGBk)

| Metric | Observed value |
|---|---|
| Producer throughput | **~850 msg/s** (400 messages, ~470ms wall time) |
| `fraud-consumers` peak lag | ~40–60 messages |
| `ledger-consumers` peak lag | ~120–180 messages |
| Lag convergence to zero | fraud: ~45s · ledger: ~80s |
| Fraud detection rate | ~20% of payments (amount > €10,000) routed to DLT |

### 10.3 Bottleneck Analysis

The results confirm the designed asymmetry: `ledger-consumers` accumulates approximately 3× more peak lag than `fraud-consumers` and takes ~80% longer to drain. The bottleneck is the simulated database write (80–200ms) vs. the fraud analysis (50–150ms). In a production environment, this would indicate that the ledger service requires either:

- Additional partitions + consumer threads, or
- A batch-write optimization to amortise database round-trip cost across multiple records

---

## 11. Design Trade-offs and Limitations

### 11.1 Event Class Duplication

Each service defines its own copy of the event records (`PaymentInitiatedEvent`, etc.) rather than sharing a common library. This is a deliberate trade-off: shared libraries create compile-time coupling and require coordinated deployments. The production-grade alternative is a **Schema Registry** (Apache Avro or Protocol Buffers with Confluent Schema Registry), which enforces schema compatibility at the broker level while keeping services fully independent.

The producer is configured not to embed Java class names in message headers (`noTypeInfo()`), and consumers deserialise using their local class with `USE_TYPE_INFO_HEADERS=false`. This decoupling allows independent evolution of each service's event model within schema compatibility constraints.

### 11.2 Single Broker, No Replication

The Docker Compose setup runs a single Kafka broker with `replication.factor=1`. This means the `acks=all` configuration provides no additional durability over `acks=1` in this environment. In production, a minimum of 3 brokers with `replication.factor=3` and `min.insync.replicas=2` is required for durability against broker failure.

### 11.3 No Transactional Outbox

The current implementation does not implement the Transactional Outbox pattern. In production, a service that writes to a database and publishes to Kafka must ensure atomicity: if the DB write succeeds but the Kafka publish fails, the system is in an inconsistent state. The Outbox pattern resolves this by writing events to a database table within the same transaction, with a separate relay process publishing them to Kafka.

### 11.4 Simulated Processing Latency

Fraud analysis and ledger writes are simulated with `Thread.sleep()`. The performance results reflect the threading and Kafka overhead, not the actual cost of ML inference or database I/O. The relative bottleneck (ledger slower than fraud) is valid, but absolute throughput figures should not be extrapolated to production workloads.

---

## 12. Conclusions

This paper presented a Kafka-based microservices architecture for digital banking payment processing, demonstrating six production-grade patterns: partitioned ordering by account ID, idempotent producer configuration, independent consumer group isolation, Dead Letter Topic routing, AdminClient-based lag observability, and real-time SSE-driven dashboard monitoring.

The experimental results show that the system sustains over 800 messages per second on commodity hardware with predictable lag behaviour under burst load. The intentional processing asymmetry between the fraud and ledger consumers demonstrates that consumer lag is a practical and actionable bottleneck indicator — a metric that directly informs horizontal scaling decisions in production.

The architectural choices documented here — particularly the use of Kafka's log model over traditional queue brokers — are motivated by concrete banking requirements: immutable audit trails, replay capability for reconciliation, and the ability to add downstream consumers without modifying existing services. These properties align directly with the regulatory and operational demands of modern digital banking platforms.

---

## 13. References

[1] Kreps, J., Narkhede, N., Rao, J. (2011). *Kafka: A Distributed Messaging System for Log Processing*. Proceedings of the NetDB Workshop, VLDB.

[2] Narkhede, N., Shapira, G., Palino, T. (2017). *Kafka: The Definitive Guide*. O'Reilly Media.

[3] Richardson, C. (2018). *Microservices Patterns*. Manning Publications. Chapter 3: Interprocess communication in a microservice architecture.

[4] Confluent Inc. (2024). *Kafka Documentation: Producer Configs — enable.idempotence*. https://kafka.apache.org/documentation/#producerconfigs

[5] Spring Framework (2024). *Spring Kafka Reference Documentation — Concurrent Message Listener Container*. https://docs.spring.io/spring-kafka/reference/

[6] Stopford, B. (2018). *Designing Event-Driven Systems*. O'Reilly Media. Chapters 4–6: Event Sourcing, CQRS, and the Log.

---

*Source code available upon request. Architecture and patterns validated on Apache Kafka 7.5 / Spring Boot 3.2 / Java 25.*
