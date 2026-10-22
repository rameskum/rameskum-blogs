---
title: 'Dead Letter Queues: Where Your Poison Messages Actually Go'
date: 2026-10-14T21:00:00+05:30
draft: false
ShowToc: true
description: >-
  One malformed message, retried forever, blocking everything behind it. Dead
  letter queues are where poison messages go to stop hurting you — with
  backoff, attempt limits, and alerts. Spring Kafka @DltHandler, RabbitMQ DLX,
  and the two 2026 broker gotchas that break poison handling silently.
tags:
  - kafka
  - rabbitmq
  - spring-boot
  - backend
  - microservices
  - interview
categories: article
keywords:
  - dead letter queue
  - poison message handling
  - spring kafka DltHandler
  - rabbitmq dead letter exchange
  - kafka share group delivery count
---

Every message-driven system eventually meets its poison message: one malformed payload, one downstream bug, one record that will never process successfully — retried forever, burning CPU, and in the worst designs blocking every message queued behind it. The dead letter queue is the answer, but "just add a DLQ" hides three decisions: how many attempts before giving up, what counts as an attempt, and who gets woken up when something lands in the DLQ. Get any of them wrong and the DLQ is theater.

## The pattern: retry, give up, quarantine, inspect

The lifecycle every poison message should follow:

1. **Retry with backoff** — transient failures (downstream blip, brief lock contention) succeed on attempt 2 or 3. Immediate requeue without delay is a hot loop; always back off.
2. **Attempt limit** — after N failures, stop. The number is a business decision, not a technical one: how long can this message wait before a human looks at it?
3. **Quarantine** — route the message to the DLQ with its failure context (attempt count, last error, original routing info). Never drop it silently.
4. **Alert and replay** — someone owns the DLQ. Fixed the bug? Replay the quarantined messages. Message genuinely bad? Archive or discard it deliberately.

A DLQ nobody watches is a trash can with extra steps. We'll come back to that.

## Spring Kafka: `@RetryableTopic` + `@DltHandler`

Spring Kafka's non-blocking retry keeps poison messages out of the consumer's way — retries go to backoff topics instead of blocking the partition:

```java
@RetryableTopic(
    attempts = "4",
    backOff = @BackOff(delay = 1000, multiplier = 2, maxDelay = 5000),
    dltProcessingFailureStrategy = DltStrategy.FAIL_ON_ERROR,
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE
)
@KafkaListener(topics = "orders")
public void handle(Order order) {
    billingService.charge(order); // throws on poison
}

@DltHandler
public void dlt(Order order,
                @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String originalTopic,
                @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String error) {
    log.warn("Quarantined order {} from {}: {}", order.id(), originalTopic, error);
    alerting.notify("dlq", order, error);
}
```

What this gives you: 4 attempts with 1s → 2s → 4s backoff on dedicated retry topics (`orders-retry-0`, …), then the message lands on `orders-dlt` and your `@DltHandler` runs. (The `@RetryableTopic` default suffix is `-dlt`; the `.DLT` suffix belongs to `DeadLetterPublishingRecoverer` used outside the retry-topic feature.) A few details worth knowing:

- **The DLT is a normal topic.** Nothing auto-alerts, auto-retries, or auto-expires it. Your `@DltHandler` is where the "who gets woken up" decision lives — log it, page on it, or forward it to your incident channel.
- **Read the `DLT_*` headers, not the received topic.** Inside a `@DltHandler`, `KafkaHeaders.RECEIVED_TOPIC` is the DLT itself (`orders-dlt`) — that's the topic being consumed. The record's origin and failure live in the headers stamped on it when it was published to the DLT: `DLT_ORIGINAL_TOPIC`, `DLT_ORIGINAL_OFFSET`, `DLT_EXCEPTION_MESSAGE`, `DLT_EXCEPTION_FQCN`, `DLT_EXCEPTION_STACKTRACE`.
- **`dltProcessingFailureStrategy` decides what happens when DLT processing itself fails.** The default, `ALWAYS_RETRY_ON_ERROR`, forwards the failed record back to the DLT so it can be attempted again. `FAIL_ON_ERROR` (shown above) instead ends the consumer's execution without forwarding the record — and with the default error handler that can effectively drop it. Know which one you're running: the fail-fast option is the one that can lose the record.
- **Undeserializable payloads never reach `handle()`.** A malformed record fails during deserialization, before your listener runs — and without `ErrorHandlingDeserializer` wrapping your deserializer, it fails in the poll loop and can block the partition. With it, the failure becomes a `DeserializationException`, which is fatal: the record skips the retry topics and goes straight to the DLT, and `ALWAYS_RETRY_ON_ERROR` will not route it back to the DLT if DLT processing fails on it.

The interview follow-up: *"Why non-blocking retry instead of just sleeping in the listener?"* Because a sleeping consumer blocks its partition — every message behind the poison one waits. Retry topics let the other 99 messages flow while the poison one waits out its backoff.

## RabbitMQ: dead-letter exchanges and delivery limits

RabbitMQ's model is exchange-based. You declare the DLX wiring on the work queue itself:

```java
Map<String, Object> args = Map.of(
    "x-queue-type", "quorum",
    "x-delivery-limit", 5,
    "x-dead-letter-exchange", "orders.dlx",
    "x-dead-letter-routing-key", "orders.dead"
);
channel.queueDeclare("orders", true, false, false, args);
channel.queueDeclare("orders.dead", true, false, false,
    Map.of("x-queue-type", "quorum"));
channel.exchangeDeclare("orders.dlx", "direct", true);
channel.queueBind("orders.dead", "orders.dlx", "orders.dead");
```

Quorum queues enforce `x-delivery-limit` themselves: after 5 failed deliveries the broker dead-letters the message automatically. **Classic queues have no such limit** — your consumer must track attempts (increment a header, check it, `basic.reject` without requeue past the threshold) or the poison message loops forever. If you're still on classic queues, that's a strong reason to migrate to quorum queues (or to count attempts in the consumer yourself).

One more declaration-level detail: dead-lettering itself is **at-most-once** by default — if the broker dies between removing the message from the source queue and enqueueing it on the target, the message is lost. Quorum queues can upgrade the handoff with `x-dead-letter-strategy: at-least-once`, which keeps the source copy until the target confirms — but it only takes effect with `x-overflow: reject-publish` on the source queue; with the default `drop-head` overflow it silently falls back to at-most-once.

## Gotcha #1 (2026): RabbitMQ 4.3 changed what counts as a delivery

This one broke real Spring AMQP integration tests. Since [RabbitMQ 4.3](https://www.rabbitmq.com/blog/2026/04/23/rabbitmq-4.3-release), quorum queues keep **two counters**: `x-acquired-count` goes up on every requeue, while `x-delivery-count` — the one the delivery limit is checked against — goes up only on a failed delivery.

The consequence, documented in [spring-amqp#3435](https://github.com/spring-projects/spring-amqp/issues/3435): a consumer failing with `basic.nack(requeue=true)` retries its poison message **indefinitely** with zero dead-lettering, because a nack no longer counts as a failed delivery, so `x-delivery-count` never reaches the limit. The same code failing with `basic.reject(requeue=true)` counts every failure and dead-letters once the limit is hit.

The rule for 4.3+: **fail with reject, not nack**, if you want the delivery limit to fire. And note the corollary — the limit is now a safety net for consumers that *crash*, not a brake on code that deliberately requeues. If your retry strategy is "nack and requeue," you need your own attempt counting regardless of the broker version.

One more from the same investigation: `x-delivery-count` travels with the message into the DLQ (it's an annotation, not just a header). A message that arrived in the DLQ with count 3 against a DLQ limit of 5 gets only 3 more failures before being dropped — and casually browsing the DLQ in the management UI can itself consume a message's remaining budget. Peek, don't poke.

## Gotcha #2 (2026): Kafka share groups archive — they don't DLQ

Kafka 4.2 made share groups production-ready: many consumers sharing partitions with per-record acknowledgements (`ACCEPT` / `RELEASE` / `REJECT`). The poison-message story here is genuinely better than classic consumer groups — a poison record no longer blocks its partition. But know the semantics:

- The broker enforces `group.share.delivery.count.limit` (**default 5**). Past that, the record is **archived**: no member of the group gets it again, but it is *not* copied anywhere — no DLQ topic, no alert, nothing. The record still exists on the log (a plain consumer can read it back), but your application will never see it again.
- KIP-1191 (dead-letter topics for share groups) is accepted but **not shipped** — so today, "archived" means your code must notice. Track `delivery_count` in the record metadata and `REJECT` (or copy to your own DLQ topic) *before* the broker archives it silently.

The interview answer: *"Share groups fix the head-of-line blocking that makes poison messages deadly in consumer groups, but their DLQ story is 'archive and move on' — if you need the quarantined payload for replay, build the DLQ copy yourself until KIP-1191 lands."*

## Alert vs auto-retry: the decision rules

Not every DLQ message deserves a page at 3 AM:

- **Alert immediately:** the DLQ is growing fast (a deploy just poisoned a whole message class), or the message type is revenue-critical (payments, order fulfillment).
- **Auto-retry on a schedule:** transient-looking failures where the downstream usually recovers — but cap the replays and count them, or you've built a slower poison loop.
- **Dashboard, reviewed daily:** low-volume DLQs on non-critical flows. The rule is *someone looks every day*, not *nobody looks ever*.

And the replay playbook: fix the bug first, then replay from oldest to newest, watching the DLQ drain rate. Replaying into an unfixed consumer is just re-poisoning with extra steps.

## The 2-minute interview answer

> "A poison message is a record that will never process successfully. Without handling, it either retries forever or blocks its partition. The pattern is: retry with exponential backoff on non-blocking retry topics, cap attempts, then quarantine to a DLQ with the failure context attached, and alert. In Spring Kafka that's `@RetryableTopic` plus `@DltHandler`, with the DLT as a normal topic your handler owns. In RabbitMQ you wire `x-dead-letter-exchange` on the queue and use `x-delivery-limit` on quorum queues — classic queues need the consumer to count attempts itself. Two current gotchas: since RabbitMQ 4.3, `basic.nack(requeue=true)` doesn't count toward the delivery limit — use reject — and Kafka share groups archive poison records after 5 deliveries by default instead of dead-lettering them, so build your own DLQ copy until KIP-1191 ships. And a DLQ nobody monitors is just a trash can — alert on growth, replay only after the fix."

## Cheat sheet

| Concern | Kafka (Spring) | RabbitMQ |
|---|---|---|
| Non-blocking retry | `@RetryableTopic` + backoff topics | Retry via DLX + TTL (or quorum redelivery) |
| Attempt cap | `attempts` on the annotation | `x-delivery-limit` (quorum only) |
| Quarantine | `orders-dlt` + `@DltHandler` | `x-dead-letter-exchange` + DLQ binding |
| Classic/consumer-group caveat | Blocking retry stalls the partition | Classic queues: consumer must count attempts |
| 2026 gotcha | Share groups *archive* after 5 — no DLQ yet | 4.3+: nack doesn't count; use reject |
| The human part | `@DltHandler` should alert, not just log | Monitor DLQ depth; replay after the fix |

Poison messages are inevitable; poison *outages* are a choice. Cap the attempts, quarantine with context, and make sure a human finds out — that's the whole pattern.
