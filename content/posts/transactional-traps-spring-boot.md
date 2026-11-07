---
title: '@Transactional: The 5 Traps That Silently Lose Your Data'
date: 2026-11-07T12:00:00+05:30
draft: false
ShowToc: true
description: >-
  @Transactional looks like one annotation doing one job — until a checked
  exception commits your half-finished order, a self-invoked method runs with
  no transaction at all, or a readOnly method silently drops your write. Five
  traps, each reproduced with a failing test on Spring Boot 4.1, then fixed.
tags:
  - java
  - spring-boot
  - backend
  - interview
categories: article
keywords:
  - spring transactional traps
  - spring transactional rollback checked exception
  - spring self invocation transactional
  - unexpectedrollbackexception spring
  - transactional readonly hibernate
  - spring boot n+1 query problem
---

`@Transactional` is the most trusted annotation in Spring Boot — and the most quietly misunderstood. It looks like one annotation doing one job: wrap this method in a transaction. In practice it is a proxy, a set of defaults inherited from EJB, and a handful of hints to Hibernate, and each of those has a sharp edge. None of the five traps below throws an error at the point where you make the mistake. They all fail silently, in production, weeks later.

Every trap in this post is reproduced with a real test, not a thought experiment. The harness is Spring Boot 4.1.2 (Spring Framework 7.0.10, Hibernate 7.4.13) against an in-memory H2 database, JUnit 6, 11 tests — one that demonstrates each trap, one that proves each fix. The full setup is two services, four small entities, and a test class; the snippets below are the real code, trimmed only where the trimming is obvious.

The domain is deliberately boring: orders, audit logs, payments, customers. The bugs are the interesting part.

## The one idea that explains traps 1–3

Spring does not rewrite your class when it sees `@Transactional`. At startup it wraps your bean in a **proxy** — a stand-in object that every other bean actually calls. The proxy runs a `TransactionInterceptor` before your method body (begin the transaction), then commits or rolls back afterwards based on what your method did. Three consequences follow, and they explain the first three traps:

1. The interceptor only runs when a call **crosses the proxy** — i.e. arrives from outside the bean.
2. It only reacts to an exception that **escapes** your method. Catch it inside, and the proxy sees a clean return.
3. Its rollback rule is not "any exception". It is: roll back on `RuntimeException` and `Error`, commit on checked exceptions.

Keep those three in mind; each trap is one of them biting.

## Trap 1: the self-invocation that runs with no transaction

The setup: placing an order should always write an audit row, even if the payment step fails afterwards. So the audit method gets `REQUIRES_NEW` — its own transaction, independent of the outer one.

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrderSelfInvoked() {
        saveOrder("laptop");
        saveAudit("order placed");          // this.saveAudit(...)
        throw new IllegalStateException("payment failed");
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit(String message) {
        AuditLog log = new AuditLog();
        log.message = message;
        audits.save(log);
    }
}
```

You would expect: the order rolls back, the audit row survives in its own transaction. What actually happens: **the audit row rolls back with the order.** The `REQUIRES_NEW` never runs.

```java
@Test
void trap1_selfInvocationIgnoresRequiresNew() {
    assertThrows(IllegalStateException.class, orderService::placeOrderSelfInvoked);
    assertEquals(0, orders.count(), "order rolled back");
    assertEquals(0, audits.count(), "audit rolled back too — REQUIRES_NEW never ran");
}
```

The call `saveAudit(...)` inside `placeOrderSelfInvoked` is really `this.saveAudit(...)`. `this` is the raw object, underneath the proxy. The call never crosses the proxy boundary, so the `TransactionInterceptor` never runs, so the annotation on `saveAudit` is silently ignored — no transaction starts at all, and the audit insert just joins whatever transaction happens to be active (here, the outer one). There is no warning, no log line, nothing. The same bypass hits `@Cacheable`, `@Async`, and `@PreAuthorize`: any proxy-based annotation is dead on a self-invocation.

The fix is structural: move the method to a separate bean, so the call genuinely crosses a bean boundary and goes through that bean's proxy.

```java
@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveAudit(String message) { /* ... */ }
}

@Service
public class OrderService {
    private final AuditService auditService;

    @Transactional
    public void placeOrderFixed() {
        saveOrder("laptop");
        auditService.saveAudit("order placed");   // crosses the proxy now
        throw new IllegalStateException("payment failed");
    }
}
```

```java
@Test
void fix1_separateBeanRequiresNewSurvives() {
    assertThrows(IllegalStateException.class, orderService::placeOrderFixed);
    assertEquals(0, orders.count(), "order rolled back");
    assertEquals(1, audits.count(), "audit committed in its own transaction");
}
```

Two alternatives exist and both are worse: inject the bean's own proxy into itself and call through it (works, smells), or enable `exposeProxy = true` and call `AopContext.currentProxy()` (couples your code to Spring AOP). Splitting the bean is the fix that stays fixed. One related rule from Spring's own docs: when using proxies, annotate only **public** methods — a `private @Transactional` method is silently ignored for the same reason. (AspectJ weaving, which modifies bytecode instead of proxying, does not have this limitation — but that is a different setup, not the Boot default.)

## Trap 2: the checked exception that commits

```java
@Service
public class PaymentService {
    @Transactional
    public void charge() throws InsufficientFundsException {
        Payment payment = new Payment();
        payment.status = "CHARGED";
        repo.save(payment);
        throw new InsufficientFundsException("no funds");  // checked
    }
}
```

The method throws. The caller sees the failure. The payment is in the database anyway.

```java
@Test
void trap2_checkedExceptionCommits() {
    assertThrows(InsufficientFundsException.class, paymentService::charge);
    assertEquals(1, payments.count(), "payment committed despite the failure");
}
```

This is the default rollback rule, inherited from EJB: Spring rolls back on unchecked exceptions (`RuntimeException` and `Error`) and **commits on checked exceptions**. The reasoning is that checked exceptions are part of a method's expected contract — recoverable business outcomes — while runtime exceptions mean something went wrong. Reasonable as a design; disastrous as a surprise, because in most codebases the exception type is chosen for readability, not for transaction semantics. A `charge()` that throws `InsufficientFundsException extends Exception` reads perfectly and commits a payment for an order that never happened.

The fix names the exception:

```java
@Transactional(rollbackFor = InsufficientFundsException.class)
public void chargeFixed() throws InsufficientFundsException { /* ... */ }
```

```java
@Test
void fix2_rollbackForCheckedException() {
    assertThrows(InsufficientFundsException.class, paymentService::chargeFixed);
    assertEquals(0, payments.count(), "payment rolled back");
}
```

The rule of thumb that survives contact with production: use unchecked exceptions for anything that should roll a transaction back, and reserve checked exceptions for outcomes the caller is expected to handle *without* undoing the work. If your codebase already has checked business exceptions everywhere, `rollbackFor` on each method works — and since Spring Framework 6.2 there is also a global switch: you can change the default rollback behaviour application-wide instead of annotating every method (the Spring team itself now recommends switching the default to `Exception.class` unless you rely on EJB-style checked-exception commit behaviour).

One more edge of the same rule: if you **catch** an exception inside the method and don't rethrow, the proxy sees a clean return and commits — the interceptor can only react to what escapes. Caught-and-swallowed is a commit, every time, unless you call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` yourself.

## Trap 3: the caught exception that poisons the outer transaction

This one is trap 2's evil twin, and it produces the most confusing error message in Spring: `UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only`.

```java
@Transactional
public void placeOrderCatchInner() {
    saveOrder("laptop");
    try {
        auditService.saveAuditRequiredAndFail("audit");  // REQUIRED, separate bean
    } catch (RuntimeException e) {
        // "handled" — or so it looks
    }
}
```

The inner method is in a separate bean with default propagation (`REQUIRED`), so it joins the outer transaction — same physical transaction. It throws a `RuntimeException`, which crosses its `@Transactional` boundary, and the interceptor marks the shared transaction **rollback-only**. Then the outer method catches the exception and returns normally. At commit time Spring finds the rollback-only flag on a method that finished cleanly, and throws:

```java
@Test
void trap3_caughtInnerFailureStillExplodes() {
    assertThrows(UnexpectedRollbackException.class, orderService::placeOrderCatchInner);
    assertEquals(0, orders.count(), "nothing committed");
}
```

Note what is surprising here: the exception you catch is not the exception you get. You catch the inner `RuntimeException` and handle it; the exception that reaches your caller is `UnexpectedRollbackException`, thrown from the commit — and by then the order you thought you saved is gone too.

The fix is to give the inner operation its own transaction, so its failure cannot poison the outer one:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditAndFail(String message) { /* save, then throw */ }

@Transactional
public void placeOrderCatchInnerFixed() {
    saveOrder("laptop");
    try {
        auditService.saveAuditAndFail("audit");
    } catch (RuntimeException e) {
        // audit rolled back on its own; the order still commits
    }
}
```

```java
@Test
void fix3_requiresNewIsolatesInnerFailure() {
    assertDoesNotThrow(orderService::placeOrderCatchInnerFixed);
    assertEquals(1, orders.count(), "order committed");
    assertEquals(0, audits.count(), "audit rolled back on its own");
}
```

`REQUIRES_NEW` suspends the outer transaction and starts a genuinely separate one — the inner failure rolls back only itself. The alternative, `Propagation.NESTED`, runs the inner work against a JDBC savepoint instead of a separate transaction: if the inner part fails it rolls back to the savepoint and the outer transaction can continue. That is cheaper than a second transaction, but it needs a database that supports savepoints, and it only works when the inner call goes through a proxy (trap 1 again).

The decision rule: if the inner operation must survive the outer transaction failing — audit logs, notifications, outbox writes — it needs `REQUIRES_NEW` in a separate bean. If it must fail *with* the outer transaction, plain `REQUIRED` is correct and you should let the exception propagate instead of catching it.

## Trap 4: the readOnly that isn't

`readOnly = true` sounds like a guarantee: this transaction will not write. It is not a guarantee. It is a hint — and Hibernate treats it in a way that produces two opposite surprises at once.

```java
@Transactional(readOnly = true)
public void updateWithoutFlush(Long id, String newStatus) {
    Payment payment = repo.findById(id).orElseThrow();
    payment.status = newStatus;
}
```

You would expect one of two things: either the write is rejected with an error, or it lands. Neither happens. The update **silently vanishes** — no exception, no log, the row keeps its old value:

```java
@Test
void trap4_readOnlyDropsUpdateWithoutFlush() {
    Payment seeded = new Payment();
    seeded.status = "PENDING";
    seeded = payments.save(seeded);
    readOnlyService.updateWithoutFlush(seeded.id, "SHOULD-NOT-STICK");
    assertEquals("PENDING", payments.findById(seeded.id).orElseThrow().status,
            "update vanished, no error raised");
}
```

Why: inside a readOnly transaction Spring puts the Hibernate session in read-only mode and sets its flush mode to MANUAL, so dirty checking is skipped and nothing is flushed at commit. Your change to a managed entity is simply never written.

The natural next thought is "fine, I'll flush explicitly" — and that fails too, which is the part that actually surprised me while writing this post:

```java
@Transactional(readOnly = true)
public void updateWithFlush(Long id, String newStatus) {
    Payment payment = repo.findById(id).orElseThrow();
    payment.status = newStatus;
    entityManager.flush();
}
```

```java
@Test
void trap4b_evenExplicitFlushCannotSaveIt() {
    Payment seeded = new Payment();
    seeded.status = "PENDING";
    seeded = payments.save(seeded);
    readOnlyService.updateWithFlush(seeded.id, "STILL-NOT-SAVED");
    assertEquals("PENDING", payments.findById(seeded.id).orElseThrow().status,
            "entities load read-only in a readOnly tx; flush finds nothing dirty");
}
```

I confirmed with SQL logging: the explicit `flush()` issues **no UPDATE at all**. Because the session is read-only, entities are loaded as read-only, so Hibernate's dirty checking skips them — there is nothing for the flush to write. `readOnly` does not throw, does not warn, and does not let you force the write through.

And then the third face of the same trap, in the opposite direction — a write that *does* land inside a readOnly transaction:

```java
@Transactional(readOnly = true)
public void insertWithoutFlush() {
    Payment payment = new Payment();
    payment.status = "INSERTED-DESPITE-READONLY";
    repo.save(payment);
}
```

```java
@Test
void trap4c_identityInsertFiresEvenInReadOnly() {
    readOnlyService.insertWithoutFlush();
    assertEquals(1, payments.count(),
            "IDENTITY insert fires on persist(), readOnly cannot drop it");
}
```

With an `IDENTITY` id, Hibernate must execute the INSERT the moment `persist()` runs — it needs the generated key back. That happens immediately, before flush mode matters, so the row lands in a transaction you marked read-only. (Some databases reject DML inside a read-only transaction at the JDBC level; H2 does not. The point stands either way: `readOnly` is a hint to the driver and the ORM, not an enforcement boundary.)

So `readOnly = true` is three things at once, none of them what the name promises: it silently drops updates to managed entities, it ignores your explicit flush, and it still lets some inserts through. Use it for what it is — a performance hint that lets Hibernate skip dirty checking and lets the driver pick read optimisations — and never as a correctness mechanism. If a method must not write, that is a code-review rule, or a database user with read-only grants. Not an annotation.

## Trap 5: the N+1 hiding inside the transaction

This trap is different from the other four: nothing is lost, nothing throws. Your code just quietly does N+1 queries, and `@Transactional` is what hides it from you.

```java
@Transactional(readOnly = true)
public List<String> listAllOrderItemsLazy() {
    return customers.findAll().stream()
            .flatMap(c -> c.orders.stream())
            .map(o -> o.item)
            .toList();
}
```

`Customer.orders` is a lazy `@OneToMany`. Outside a transaction, touching it throws `LazyInitializationException` — loud, obvious, fixed on the spot. Inside a `@Transactional` method the session is open, so every lazy access *works*. It just costs one extra query per customer. With 3 customers holding 2 orders each, Hibernate statistics report 4 prepared statements: 1 to load the customers, 1 per customer to load their orders. With 3,000 customers it is 3,001.

```java
@Test
void trap5_lazyLoadingIsNPlusOne() {
    seedCustomers(3, 2);
    Statistics statistics = stats();
    var items = customerService.listAllOrderItemsLazy();
    assertEquals(6, items.size());
    assertEquals(4, statistics.getPrepareStatementCount(),
            "1 statement for customers + 1 per customer for orders");
}
```

(The count is prepared statements, not query executions — Hibernate's statistics count collection fetches separately from query executions, so `getQueryExecutionCount()` reports 1 here and hides the problem exactly the way the transaction does. Count statements.)

The fix is a fetch join — one query, orders loaded with the customers:

```java
@Query("select distinct c from Customer c left join fetch c.orders")
List<Customer> findAllWithOrders();
```

```java
@Test
void fix5_fetchJoinIsOneQuery() {
    seedCustomers(3, 2);
    Statistics statistics = stats();
    var items = customerService.listAllOrderItemsFetchJoin();
    assertEquals(6, items.size());
    assertEquals(1, statistics.getPrepareStatementCount(), "single statement");
}
```

Fetch joins and `@EntityGraph` are the usual fixes; `@BatchSize` is the middle ground when you genuinely need the collections lazily but want them loaded in batches instead of one-by-one. The detection habit matters more than the fix: enable Hibernate statistics (or a query counter) in tests and assert on statement counts for any method that walks a lazy association. The `LazyInitializationException` you get without a transaction is, perversely, the *helpful* version of this bug — it fails in development. The transactional version fails in production, as latency.

## The checklist

| Trap | Symptom | Root cause | Fix |
| --- | --- | --- | --- |
| Self-invocation | `REQUIRES_NEW` silently ignored | Call never crosses the proxy | Separate bean; annotate only public methods |
| Checked exception | Failure commits half the work | Rollback only on unchecked, by default | `rollbackFor`, or unchecked exceptions |
| Caught inner failure | `UnexpectedRollbackException` at commit | Inner failure marks the shared tx rollback-only | `REQUIRES_NEW` (or `NESTED` savepoint) in a separate bean |
| `readOnly` | Writes vanish silently — or land anyway | Read-only session + MANUAL flush; IDENTITY inserts fire on `persist()` | Treat `readOnly` as a perf hint, never a guard |
| N+1 inside the tx | Works fine, costs N queries | Lazy loading succeeds while the session is open | Fetch join / `@EntityGraph` / `@BatchSize`; assert statement counts in tests |

The through-line: `@Transactional` is not a declaration about your method. It is a declaration about a proxy, a rollback rule, and a Hibernate session — and each of those has defaults that were chosen for a different era's problems. The annotation works exactly as documented; the traps are what happens when you assume it works as named. Read the three proxy rules at the top of this post once more, write one test per trap against your own services, and these five will never cost you a 3 a.m. page.

## Sources

- Spring Framework reference, [Declarative transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html) — rollback rules, propagation, and the proxy model.
- Spring Data JPA reference, [Transactions](https://docs.spring.io/spring-data/jpa/reference/jpa.transactions.html) — `readOnly` as a hint; Hibernate flush mode set to `NEVER`/`MANUAL` in read-only transactions.
- Spring Framework 6.2 release notes — global default rollback configuration.
- Baeldung, [Handling UnexpectedRollbackException in Spring](https://www.baeldung.com/spring-unexpected-rollback-exception) — the rollback-only propagation scenario.
- All 11 tests run on Spring Boot 4.1.2 (Spring Framework 7.0.10, Hibernate 7.4.13, H2 2.4.240, JUnit 6.0.3, JDK 25): 5 trap demonstrations, 5 fixes, plus the readOnly flush probe — all passing.
