---
title: 'Dual Writes Are Lies: The Transactional Outbox Pattern in Spring Boot + Kafka'
date: 2026-09-27T10:08:00-04:00
draft: false
ShowToc: true
description: >-
  Your database commit and your Kafka send can never be atomic. The
  transactional outbox pattern fixes dual writes for good: write the event to an
  outbox table in the same transaction as your business change, then relay it to
  Kafka with a polling publisher or Debezium CDC. Full Spring Boot code, the
  exact failure it prevents, and the interview one-liner.
tags:
  - spring-boot
  - java
  - kafka
  - backend
  - system-design
  - interview
categories: article
keywords:
  - transactional outbox pattern
  - spring boot kafka outbox
  - debezium outbox event router
  - dual write problem
---

It's 3 AM. A customer paid, the `orders` row is in your database, and the `OrderCreated` event never reached Kafka. The warehouse never shipped. The email never sent. Somewhere in your service there is code that looks like this:

```java
@Transactional
public Order placeOrder(PlaceOrder cmd) {
    Order order = orderRepository.save(new Order(cmd)); // write #1
    kafkaTemplate.send("orders", order.getId(), toJson(order)); // write #2 — outside the safety net
    return order;
}
```

Two writes. No shared atomicity. If the broker is down, if the pod gets killed, if the send times out *after* the commit — the database and the downstream world silently disagree, and you get to find out from an angry customer. This is the **dual-write problem**, and the fix is the transactional outbox pattern.

## Why you can't just "make them atomic"

The honest question first: why not wrap both in one transaction? Because you'd need XA / two-phase commit between Postgres and Kafka, coordinated through JTA. It technically exists. XA/two-phase commit can provide cross-system atomicity, but its operational and performance costs — broker and driver support, miserable throughput, recovery complexity — make the transactional outbox a much more common choice for this architecture.

## The pattern in one paragraph

Every state change that must produce an event writes a row to an `outbox` table **in the same database transaction** as the business change. A separate relay process reads committed outbox rows and publishes them to Kafka, marking them sent. The event is *in the database* before anyone promises anything to the network. If the relay crashes mid-batch, the row is still there — it just gets published on the next pass. The specific dual-write loss (database committed, Kafka send failed) becomes recoverable by construction. That said, the relay itself is a component you operate: it can stall, misroute, or fall behind, and aggressive retention or a stalled CDC slot can still bite you — so you monitor it like anything else critical. Duplicates remain the normal failure mode, and duplicates you already know how to handle (see [the idempotency-keys post](https://blogs.rameskum.com/posts/idempotency-keys-spring-boot-stripe-style/)).

## The outbox table

```sql
CREATE TABLE outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(64)  NOT NULL,   -- "Order"
    aggregate_id    VARCHAR(64)  NOT NULL,   -- the order id
    event_type      VARCHAR(64)  NOT NULL,   -- "OrderCreated"
    payload         JSONB        NOT NULL,   -- the event, as JSON
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ              -- NULL = not yet relayed
);
CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;
```

Flyway migration, committed next to your domain schema. The `aggregate_id` becomes the Kafka message key later — that's what keeps all events for one order on one partition, in order.

## Write side: one transaction, two rows

```java
@Entity
@Table(name = "outbox")
public class OutboxEvent {
    @Id private UUID id = UUID.randomUUID();
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    @Column(columnDefinition = "jsonb")
    private String payload;
    private Instant createdAt = Instant.now();
    private Instant publishedAt; // null until relayed
    // getters/setters omitted
}

@Service
public class OrderService {
    private final OrderRepository orders;
    private final OutboxRepository outbox;
    private final ObjectMapper mapper;

    @Transactional   // <-- the whole point: ONE transaction
    public Order placeOrder(PlaceOrder cmd) {
        Order order = orders.save(new Order(cmd));

        OutboxEvent event = new OutboxEvent();
        event.setAggregateType("Order");
        event.setAggregateId(order.getId().toString());
        event.setEventType("OrderCreated");
        event.setPayload(mapper.writeValueAsString(new OrderCreated(order)));
        outbox.save(event);

        return order; // commit persists BOTH or NEITHER
    }
}
```

Notice what's gone: `kafkaTemplate` is nowhere near this method. The service no longer knows Kafka exists. That's not incidental — it's the design. The write path's only dependency is the database.

## Relay option A: the polling publisher

A `@Scheduled` bean claims unpublished rows, sends them synchronously, and marks them published. The two load-bearing details: `FOR UPDATE SKIP LOCKED` (multiple relay replicas claim disjoint row sets, no double-send) and `fixedDelay` (counts from job completion, so a slow batch never overlaps itself).

```java
@Component
public class OutboxRelay {
    private final OutboxRepository outbox;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void relay() {
        List<OutboxEvent> batch = outbox.claimBatch(100); // SELECT ... FOR UPDATE SKIP LOCKED
        for (OutboxEvent e : batch) {
            try {
                // .get() blocks until the broker acks: at-least-once.
                // On timeout/exception the tx rolls back and the row stays unpublished.
                kafka.send(topicFor(e), e.getAggregateId(), e.getPayload()).get(10, SECONDS);
                e.setPublishedAt(Instant.now());
            } catch (Exception ex) {
                log.warn("outbox relay failed for {}, will retry next poll", e.getId(), ex);
                // leave unpublished — next poll retries it
            }
        }
    }
}
```

```sql
-- OutboxRepository.claimBatch, native query:
SELECT * FROM outbox
WHERE published_at IS NULL
ORDER BY created_at
LIMIT :batch
FOR UPDATE SKIP LOCKED
```

Honest trade-offs: polling adds up to one poll-interval of latency, and it puts a constant SELECT+UPDATE load on your write database. For most services that's completely fine — this is the option I'd start with. You switch when you outgrow it, and the outbox table contract doesn't change either way.

## Relay option B: Debezium CDC

If you already run Kafka Connect, or you need sub-second latency at high throughput, skip polling: Debezium tails the Postgres WAL and streams outbox inserts directly. `published_at` is never even written — the WAL *is* the queue.

```yaml
# docker-compose addition
services:
  postgres:
    image: postgres:17
    command: ["postgres", "-c", "wal_level=logical"]
  connect:
    image: debezium/connect:2.7
    depends_on: [kafka, postgres]
```

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "plugin.name": "pgoutput",
  "slot.name": "outbox_slot",
  "table.include.list": "public.outbox",
  "transforms": "outbox",
  "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.table.field.event.key": "aggregate_id",
  "transforms.outbox.table.field.event.payload": "payload",
  "transforms.outbox.route.by.field": "event_type",
  "transforms.outbox.route.topic.replacement": "${routedByValue}"
}
```

The Outbox Event Router SMT reads your row and produces a proper Kafka record: `aggregate_id` becomes the key, `payload` becomes the value, `event_type` routes to the topic (`OrderCreated` → topic `OrderCreated`). Your service's write path doesn't change at all — you just delete the polling relay.

The operational cost is real, though: you're now running Debezium + Kafka Connect, managing a logical replication slot, and one stalled consumer can grow your Postgres WAL. That's the price of near-real-time.

| | Polling relay | Debezium CDC |
|---|---|---|
| Delivery | At-least-once | At-least-once |
| Latency | One poll interval (~1s) | Near-real-time |
| Extra infra | None | Debezium + Kafka Connect + replication slot |
| DB load | Periodic SELECT + UPDATE | WAL reads only |
| Switch when | Default choice | Thousands of events/sec, or Connect already running |

## The exact failure it fixes

Walk the 3 AM scenario through the pattern:

1. `placeOrder` commits. Order row **and** outbox row are durable. Kafka is down — nobody cares yet.
2. The relay's `kafka.send(...).get()` throws. The relay transaction rolls back. `published_at` stays NULL.
3. Next poll: the row is still unpublished. Relay retries. And retries.
4. Broker recovers. The event publishes. Downstream catches up.

Without the outbox, step 2 is a lost event with no record it ever existed. With the outbox, step 2 is a row in a table with a NULL column — visible, queryable, monitorable. Alert on `SELECT count(*) FROM outbox WHERE published_at IS NULL AND created_at < now() - interval '5 minutes'` and you'll know about relay trouble before your customers do.

## The other half: duplicates

The outbox gives you durable, retryable event publication with at-least-once semantics. A relay crash between the broker ack and the `published_at` update means a duplicate. This is the correct trade-off — duplicates are detectable, lost events aren't — but your consumers must be idempotent. That's exactly what [idempotency keys](https://blogs.rameskum.com/posts/idempotency-keys-spring-boot-stripe-style/) are for. The full formula:

> **Transactional outbox + idempotent consumers gives you effectively-once business processing, even though the underlying delivery remains at-least-once.**

Neither half works alone. Together, they close the loop.

## Gotchas worth knowing

- **Ordering:** key every record by `aggregate_id` so all events for one aggregate land on one partition, in order. Cross-aggregate ordering is not guaranteed — design your consumers accordingly.
- **Table growth:** published rows are dead weight. Purge them on a schedule (`DELETE FROM outbox WHERE published_at < now() - interval '7 days'`), or you'll be explaining a 50 GB outbox table in a year.
- **Payload schema:** JSON is fine; version it (`event_type` + a schema version in the payload) because consumers will outlive your current model.
- **Don't over-engineer the start:** polling relay, one instance, 1-second interval. You can run a surprising amount of business through that.

## The interview one-liner

*"You can't atomically commit a database write and a Kafka send, so you write the event to an outbox table in the same DB transaction and let a relay publish it — at-least-once delivery, and the consumer deduplicates with idempotency keys."*

If they ask why not XA: *"Fragile, slow, and nobody operates it between a database and a broker. The outbox is the industry's answer."*
