---
title: 'After the Outbox: The Saga Pattern for Multi-Service Transactions in Spring Boot'
date: 2026-09-29T08:45:00-04:00
draft: false
ShowToc: true
description: >-
  The transactional outbox guarantees your event gets published. But what runs
  the business transaction across services? The saga pattern: a sequence of
  local transactions with compensating actions, in choreography and
  orchestration flavors. Full Spring Boot + Kafka code, the failure cases
  interviews probe, and the interview one-liner.
tags:
  - spring-boot
  - java
  - kafka
  - backend
  - system-design
  - microservices
  - interview
categories: article
keywords:
  - saga pattern spring boot
  - saga orchestration vs choreography
  - compensating transactions microservices
  - distributed transactions saga
---

It's 3 AM again. A customer placed an order, payment went through, and inventory reservation failed — the warehouse is out of stock. The customer has been charged for something that will never ship. The order row exists and the customer was charged, but nothing will ship — and no single transaction can unwind it, because there is no `@Transactional` that spans your order service, your payment service, and your inventory service. Each one already committed its own local transaction.

This is the problem the transactional outbox does *not* solve. The outbox guarantees your event gets out of the service. It says nothing about what happens when the business transaction spans three databases owned by three services. That's the saga's job.

## Why you can't just "make it atomic"

The textbook answer is two-phase commit: a coordinator asks every participant to prepare, waits for unanimous yes, then tells everyone to commit. In a classroom, it works. In production microservices, it requires every participant to promise to commit before anyone actually does — holding that promise across network round-trips while the coordinator collects unanimous agreement. One slow participant stalls everyone, and one crashed participant blocks recovery. Distributed transactions across services trade availability for a consistency guarantee that most business flows don't actually need — the order doesn't have to be *atomically* consistent, it has to end up *correctly* consistent.

The industry's answer: break the distributed transaction into a **sequence of local transactions**, one per service. Each step commits independently. If a later step fails, you run **compensating transactions** that undo the business effects of the earlier steps — not a physical rollback, a *semantic* undo. Refund the payment. Release the reservation. Cancel the order. The system converges on a consistent business state. This is the **saga pattern**, and the consistency it gives you is eventual, not ACID.

That distinction matters, so let's be precise: a saga does not give you atomicity. There will be moments — while the saga is running, while it's compensating — when the system's state is visibly inconsistent. The guarantee is weaker and more honest: the saga is designed to always terminate in a business-consistent state, success or compensated failure. But a compensation that keeps failing can leave it stuck indefinitely — which is why "compensation failed" has to be the loudest alert in the system, not a log line.

## The flow we'll build

Same example throughout: place an order across three services.

```
create order → charge payment → reserve inventory → (confirm order)
```

If payment fails: cancel the order.
If inventory fails: refund the payment, cancel the order.

There are two ways to coordinate this. They look similar in a diagram and feel very different in production.

## Choreography: events, no boss

In choreography, there is no coordinator. Each service reacts to events, does its local transaction, and publishes the next event. The saga emerges from the chain.

```java
// OrderService — kicks it off
@Transactional
public Order placeOrder(PlaceOrder cmd) {
    Order order = orderRepository.save(new Order(cmd));
    outbox.save("orders", new OrderCreated(order.getId(), cmd.items(), cmd.amount()));
    return order;
}
```

Note the `outbox.save`, not `kafka.send` — publishing straight from the transaction is the dual-write problem from the [outbox post](https://blogs.rameskum.com/posts/transactional-outbox-spring-boot-kafka/). Every event in this article goes to the outbox table in the same local transaction; a relay publishes them to Kafka. The saga coordinates the business transaction, the outbox makes its messaging durable.

```java
// PaymentService — reacts to OrderCreated
@KafkaListener(topics = "orders", groupId = "payment")
@Transactional
public void onOrderCreated(OrderCreated e) {
    try {
        paymentGateway.charge(e.orderId(), e.amount());   // idempotency key = orderId
        outbox.save("payments", new PaymentCharged(e.orderId(), e.items()));
    } catch (PaymentDeclined ex) {
        outbox.save("payments", new PaymentFailed(e.orderId(), ex.getReason()));
    }
}
```

```java
// InventoryService — reacts to PaymentCharged
@KafkaListener(topics = "payments", groupId = "inventory")
@Transactional
public void onPaymentCharged(PaymentCharged e) {
    try {
        inventory.reserve(e.orderId(), e.items());
        outbox.save("inventory", new InventoryReserved(e.orderId()));
    } catch (OutOfStock ex) {
        outbox.save("inventory", new InventoryFailed(e.orderId()));
    }
}
```

And the compensation side — each service also listens for failure events and undoes its own work:

```java
// PaymentService — compensates when inventory fails
@KafkaListener(topics = "inventory", groupId = "payment-compensation")
@Transactional
public void onInventoryFailed(InventoryFailed e) {
    paymentGateway.refund(e.orderId());   // must be idempotent — this event can redeliver
    outbox.save("payments", new PaymentRefunded(e.orderId()));
}
```

This works, and for a simple linear flow it's the least code. Now ask the 3 AM questions:

- **Where is the state of the saga?** Nowhere. It's scattered across three services' databases and their consumer logs. To answer "where is order 8472 stuck?" you grep three codebases.
- **Who decides the compensation order?** Nobody explicitly. Each service reacts to what it hears. Add a fourth service — say, fraud check between payment and inventory — and every service's listener wiring changes.
- **What stops a compensation loop?** Discipline. A `PaymentRefunded` event landing on the wrong listener can re-trigger the saga. Event choreography has no single place that says "this saga is over."

Choreography optimizes for service autonomy. You pay for it in observability.

## Orchestration: one coordinator, explicit commands

In orchestration, a dedicated **saga orchestrator** owns the workflow. It sends commands to participants, tracks the saga's state in its own durable table, and on failure runs compensations itself, in reverse order.

```java
@Component
public class OrderSagaOrchestrator {

    public void runSaga(UUID sagaId, PlaceOrder cmd) {
        sagaState.start(sagaId);                          // durable state FIRST
        try {
            paymentClient.charge(sagaId, cmd.amount());   // step 1
            sagaState.stepDone(sagaId, "PAYMENT_CHARGED");

            inventoryClient.reserve(sagaId, cmd.items()); // step 2
            sagaState.stepDone(sagaId, "INVENTORY_RESERVED");

            shippingClient.ship(sagaId, cmd.address());   // step 3
            sagaState.complete(sagaId);
        } catch (SagaStepException e) {
            compensate(sagaId, e.failedStep());           // unwind, reverse order
        }
    }

    private void compensate(UUID sagaId, SagaStep failedStep) {
        // Compensate only the steps that actually completed,
        // in reverse order of execution.
        List<SagaStep> completed = sagaState.completedSteps(sagaId);
        Collections.reverse(completed);
        for (SagaStep step : completed) {
            try {
                step.compensate(sagaId);                   // idempotent, retried
            } catch (Exception e) {
                sagaState.compensationFailed(sagaId, step, e);
                // do NOT swallow — escalate to dead-letter / on-call
                throw new CompensationFailedException(sagaId, step, e);
            }
        }
        sagaState.compensated(sagaId);
    }
}
```

Two things to notice. First, the `saga_state` table: the orchestrator records its progress durably, so a crash mid-saga is usually recoverable — on restart it reads the state and resumes or compensates from the last known step. But durable state alone doesn't close every window: if the process dies after the payment succeeds and before `stepDone()` is recorded, recovery will retry a step that already executed — which is exactly why every step must be idempotent. An orchestrator without durable state is just a script with a fancy name; an orchestrator without idempotent steps is a duplicate-charge generator.

Second, notice what's missing from the orchestrator: business logic. It doesn't know how to charge a card or reserve stock. It knows the *sequence* and the *compensation mapping*. Keep it that way — the moment the orchestrator starts making business decisions, it becomes the distributed monolith everyone feared.

## Choreography vs orchestration: when to pick which

| | Choreography | Orchestration |
|---|---|---|
| Coordinator | None — services react to events | Central orchestrator issues commands |
| Saga state | Distributed across services | One `saga_state` table |
| Compensation | Each service listens for failure events | Orchestrator runs compensations in reverse |
| Debugging | Grep N services' logs | Read one state table |
| Coupling | Services know each other's event contracts | Services only know the orchestrator's commands |
| Failure mode | Silent stalls, event loops | Orchestrator is a critical component |

Rules of thumb that survive contact with production:

- **Choreography** for short, linear flows (2–3 steps) where teams own their services end-to-end and nobody will ever ask for an audit trail of the transaction.
- **Orchestration** for longer flows, complex compensation (conditional branches, partial compensation), or anything regulated — the central state table is the audit trail.

Most teams that start with choreography and grow past four services end up building an orchestrator anyway, except now it's called "the tracking service" and it was built at 3 AM. Skip the intermediate step.

## The hard parts (where interviews live)

The happy path is a tutorial. The failure handling is the job.

**Every step — including compensations — must be idempotent.** Steps get retried. Events redeliver. The refund for order 8472 might be invoked twice, and charging the customer twice to fix charging them once is the kind of bug that ends careers. Natural idempotency keys are everywhere here: the saga ID, the order ID. Use them as the dedup key on every participant, exactly like [idempotency keys](https://blogs.rameskum.com/posts/idempotency-keys-spring-boot-stripe-style/) on your APIs.

**Compensation can fail too.** The refund call can time out. The inventory release can hit a dead service. A saga whose compensation fails is stuck in a state no automatic logic should paper over: mark it, dead-letter it, page someone. Design the compensation path with retries and a bounded number of attempts, and make "compensation failed" the loudest alert in the system — a half-compensated saga is worse than a failed one, because it *looks* handled.

**Retry with backoff, and know when to stop.** Transient failures (network blip, brief downstream outage) deserve retries with exponential backoff. Permanent failures (payment declined, out of stock) deserve immediate compensation, not ten retries of a card that will never clear. Your step implementations need to distinguish the two — catch the domain exception separately from the infrastructure exception.

**The saga needs reliable messaging — which is where the outbox comes back in.** Whether the orchestrator sends commands or services publish events, a lost command means a stuck saga. If your saga steps communicate over Kafka, the [transactional outbox](https://blogs.rameskum.com/posts/transactional-outbox-spring-boot-kafka/) is what guarantees the command or event is durably recorded and reliably published with retries. Note the boundary: the outbox guarantees publication, not end-to-end processing — the downstream service still has to receive and idempotently handle the message. The patterns compose: outbox for durable publication, idempotency keys for safe retries, saga for the cross-service business transaction.

**Observability is a design requirement, not a nice-to-have.** Propagate the saga ID as a correlation ID through every command, event, and log line. Your 3 AM self should be able to run one query — against the orchestrator's state table or your log aggregator — and see the saga's full history: steps completed, step failed, compensations run.

## Gotchas worth knowing

- **Semantic vs physical rollback:** compensation is not `ROLLBACK`. Refunding a payment is a new business transaction with its own side effects (the customer sees a charge *and* a refund). Design compensations as first-class business operations, not afterthoughts.
- **Don't compensate what didn't happen:** the orchestrator compensates only steps recorded as completed in `saga_state`. A step that threw before committing gets no compensation — but "threw before committing" versus "committed then the ack was lost" is exactly why steps must be idempotent.
- **Parallel steps are possible, compensation gets harder:** independent steps (reserve inventory *and* run fraud check) can run concurrently, but partial failure means compensating a subset in the right order. Keep it sequential until you can prove you need parallel.
- **Timeouts are failures:** a step that never responds is a failed step. Every saga needs per-step timeouts, or one hung downstream service parks your saga forever.
- **Sagas and the outbox share a failure philosophy:** at-least-once delivery, idempotent handling, no lost work. If the outbox post's formula felt right, the saga is the same thinking applied one level up.

## The interview one-liner

*"A saga is a sequence of local transactions across services, where each step's business effect is undone by a compensating transaction if a later step fails. You trade ACID atomicity for eventual consistency — and every step, including the compensations, has to be idempotent because everything gets retried."*

If they ask choreography vs orchestration: *"Choreography is events with no boss — fine for simple linear flows, painful to debug. Orchestration is a coordinator issuing commands with a durable state table — central visibility, explicit compensation, the default for anything an auditor will ask about."*

If they ask why not 2PC: *"It needs every participant to promise to commit before anyone does — one slow participant stalls everyone, one crashed participant blocks recovery, and running XA across microservices is rare in practice because of that operational cost. Sagas are the industry's default answer."*

And if they ask what the outbox has to do with it: *"The saga coordinates the business transaction; the outbox guarantees the saga's commands and events are durably recorded and reliably published — not that they're processed end to end, that's what idempotent consumers are for. Outbox for durable publication, idempotency keys for safe retries, saga for cross-service consistency — that's the full stack."*
