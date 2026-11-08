---
title: 'Bulkheads in Spring Boot: Stop One Slow Dependency From Sinking the Service'
date: 2026-11-08T12:00:00+05:30
draft: false
ShowToc: true
description: >-
  One slow downstream, one shared thread pool, and suddenly your health
  endpoint can't answer either. Bulkheads cap how much of your service one
  dependency can consume — first with the native
  @ConcurrencyLimit (Spring Framework 7), then with Resilience4j's semaphore
  and thread-pool bulkheads. Every claim below is measured, not asserted: 7 tests, real
  timings, on Spring Boot 4.1.2.
tags:
  - java
  - spring-boot
  - backend
  - microservices
categories: article
keywords:
  - spring boot bulkhead pattern
  - resilience4j bulkhead semaphore threadpool
  - spring concurrencylimit annotation
  - bulkhead vs circuit breaker
  - thread pool exhaustion spring boot
---

A ship stays afloat when one compartment floods because bulkheads — the watertight walls between sections — keep the water in one place. Your service has the same problem in reverse: one slow dependency floods the shared thread pool, and everything else drowns with it. A payment provider that takes 2 seconds instead of 200 milliseconds doesn't just make payments slow. It occupies every request thread until the pool is full, and then your health endpoint, your login page, and your fast internal calls all queue behind it. The service isn't broken. It's busy — on someone else's problem.

A bulkhead is the wall: a cap on how many threads, or how many concurrent calls, one dependency is allowed to consume. This post builds the failure first, measures it, then fixes it three ways — Spring Boot 4's native `@ConcurrencyLimit`, Resilience4j's semaphore bulkhead, and Resilience4j's thread-pool bulkhead — and finishes with the sizing rule that decides which one you actually need.

The setup: Spring Boot 4.1.2 (Spring Framework 7.0.10), Resilience4j 2.4.0 with the `resilience4j-spring-boot4` starter, JDK 25. The "downstream" is a fake service that sleeps and counts how much concurrency it saw. Seven tests, all passing, all with real timings — the numbers below are measured, not asserted. (The concurrency tests fire 5 callers at once from a 5-thread pool through a small helper that releases them behind a shared latch and records each outcome as `OK:` or `FAIL:<ExceptionName>`.)

## The failure, measured

The demo needs a small stage. Imagine a 2-thread server pool — the whole service, reduced to its essence. Two slow calls arrive, each taking 2 seconds. Then a fast call arrives.

```java
@Service
public class SaturationService {
    public String slowUnprotected() {
        sleep(2000);
        return "slow-ok";
    }

    public String fast() {
        return "fast-ok";
    }
}
```

```java
@Test
void withoutBulkheadSlowCallsStarveFastOnes() throws Exception {
    ExecutorService serverPool = Executors.newFixedThreadPool(2);
    Future<String> slow1 = serverPool.submit(saturationService::slowUnprotected);
    Future<String> slow2 = serverPool.submit(saturationService::slowUnprotected);
    Thread.sleep(100); // let both slow calls occupy the pool
    long start = System.currentTimeMillis();
    Future<String> fast = serverPool.submit(saturationService::fast);
    assertEquals("fast-ok", fast.get(10, TimeUnit.SECONDS));
    long fastLatency = System.currentTimeMillis() - start;
    assertTrue(fastLatency >= 1500, "fast call waited behind the slow ones, was " + fastLatency);
}
```

The fast call takes ~2 seconds. Not because it is slow — it returns instantly — but because there is no thread to run it on. Both threads are busy waiting on the slow dependency. Scale the numbers up and this is exactly a production incident: a downstream degrades, your pool saturates, your p99 explodes on *every* endpoint, and the autoscaler adds replicas that saturate in exactly the same way.

Now the same stage, with one annotation on the slow method:

```java
@Service
public class BulkheadedSaturationService {
    @Bulkhead(name = "saturation", type = Bulkhead.Type.SEMAPHORE)
    public String slow() {
        sleep(2000);
        return "slow-ok";
    }

    public String fast() {
        return "fast-ok";
    }
}
```

```properties
resilience4j.bulkhead.instances.saturation.max-concurrent-calls=1
resilience4j.bulkhead.instances.saturation.max-wait-duration=0
```

```java
@Test
void withBulkheadFastCallSurvives() throws Exception {
    ExecutorService serverPool = Executors.newFixedThreadPool(2);
    Future<String> slow1 = serverPool.submit(bulkheadedSaturationService::slow);
    Thread.sleep(100); // let the first slow call take the single permit
    Future<String> slow2 = serverPool.submit(bulkheadedSaturationService::slow);
    long start = System.currentTimeMillis();
    Future<String> fast = serverPool.submit(bulkheadedSaturationService::fast);
    assertEquals("fast-ok", fast.get(10, TimeUnit.SECONDS));
    long fastLatency = System.currentTimeMillis() - start;
    assertTrue(fastLatency < 1000, "fast call ran immediately, was " + fastLatency);
    ExecutionException rejected = assertThrows(ExecutionException.class, slow2::get);
    assertInstanceOf(BulkheadFullException.class, rejected.getCause());
}
```

The second slow call is rejected immediately with a `BulkheadFullException` — its thread is freed in milliseconds instead of 2 seconds — and the fast call runs on it at once. The slow dependency still fails, but the failure is now *contained*: one dependency's bad day costs you one permit's worth of work, not the whole pool. That containment is the entire point. A bulkhead does not fix the slow dependency. It decides how much of your service the slow dependency is allowed to take with it.

## Option 1: the native one — @ConcurrencyLimit

Spring Framework 7.0 shipped resilience features in the core framework, and the bulkhead is one of them: `@ConcurrencyLimit`, in `org.springframework.resilience.annotation`. No extra library. You enable it with `@EnableResilientMethods` on a configuration class:

```java
@SpringBootApplication
@EnableResilientMethods
public class BulkheadCheckApplication { }
```

```java
@Service
public class NativeService {
    @ConcurrencyLimit(2)
    public String limited() {
        return downstream.call();
    }
}
```

Two things to know before you use it. First, the default policy **blocks**: when the limit is reached, excess callers wait until a slot frees — it behaves like a pool limit, not a rejection. Five callers against a limit of 2, with each call taking 300ms, all complete, never more than 2 at once, in about 900ms:

```java
@Test
void nativeDefaultBlocksInsteadOfFailing() throws Exception {
    long start = System.currentTimeMillis();
    List<String> results = fireConcurrently(5, nativeService::limited);
    long elapsed = System.currentTimeMillis() - start;
    assertEquals(5, results.stream().filter(r -> r.startsWith("OK:")).count(), "all callers complete");
    assertEquals(2, downstream.maxInFlight(), "never more than 2 concurrent");
    assertTrue(elapsed >= 900, "3 batches of 300ms = at least 900ms, was " + elapsed);
}
```

Second, there is a reject policy, and it is explicit (the `policy` attribute arrived in Framework 7.0.3, so make sure you are past that patch release):

```java
@ConcurrencyLimit(limit = 2, policy = ThrottlePolicy.REJECT)
public String limitedReject() {
    return downstream.call();
}
```

```java
@Test
void nativeRejectPolicyFailsFast() throws Exception {
    List<String> results = fireConcurrently(5, nativeService::limitedReject);
    assertEquals(2, results.stream().filter(r -> r.startsWith("OK:")).count());
    assertEquals(3, results.stream().filter(r -> r.equals("FAIL:InvocationRejectedException")).count());
}
```

Rejected callers get an `InvocationRejectedException` (a `RejectedExecutionException`). The annotation also works at the type level, applying to every proxy-invoked method in the class, and `limitString` takes a property placeholder or SpEL expression if the limit should come from config.

One correction to a claim you'll find in migration guides: there is no `spring-boot-starter-resilience` that wires this up — I checked Maven Central, the artifact does not exist. The native annotations live in `spring-context` itself, and `@EnableResilientMethods` is all the wiring there is.

The native bulkhead is a semaphore and nothing else. No fallback, no metrics endpoint, no per-instance configuration file. If that is all you need — cap the concurrency on one method — it is the right call, and it costs you zero dependencies. It is also the natural fit under virtual threads, where there is no pool size to act as an accidental limit (the [virtual threads post](https://blogs.rameskum.com/posts/spring-boot-virtual-threads/) builds the same idea by hand with a `Semaphore`). If you need any of the rest, keep reading.

## Option 2: Resilience4j semaphore bulkhead — fail fast, with a fallback

Resilience4j's semaphore bulkhead is the same idea with a wait budget: `maxConcurrentCalls` permits, and `maxWaitDuration` — how long a caller may wait for a permit before being rejected. Set the wait to zero and rejection is immediate:

```properties
resilience4j.bulkhead.instances.slow.max-concurrent-calls=2
resilience4j.bulkhead.instances.slow.max-wait-duration=0
```

```java
@Bulkhead(name = "slow", type = Bulkhead.Type.SEMAPHORE)
public String slow() {
    return downstream.call();
}
```

```java
@Test
void resilienceSemaphoreFastFails() throws Exception {
    List<String> results = fireConcurrently(5, resilienceService::slow);
    assertEquals(2, results.stream().filter(r -> r.startsWith("OK:")).count());
    assertEquals(3, results.stream().filter(r -> r.equals("FAIL:BulkheadFullException")).count());
    assertEquals(2, downstream.maxInFlight());
}
```

Two of five run; three fail fast with `BulkheadFullException`. The difference from the native default is the point: a semaphore with a zero wait budget turns overload into an immediate, catchable error instead of a queue. For a user-facing request path, an immediate 503 (or a fallback) beats a request that waits in line for a dependency that is already drowning.

And the fallback is why most teams end up here rather than at the native annotation:

```java
@Bulkhead(name = "slow", type = Bulkhead.Type.SEMAPHORE, fallbackMethod = "slowFallback")
public String slowWithFallback() {
    return downstream.call();
}

public String slowFallback(BulkheadFullException e) {
    return "cached";
}
```

```java
@Test
void resilienceSemaphoreFallback() throws Exception {
    List<String> results = fireConcurrently(5, resilienceService::slowWithFallback);
    assertEquals(2, results.stream().filter(r -> r.equals("OK:ok")).count());
    assertEquals(3, results.stream().filter(r -> r.equals("OK:cached")).count());
}
```

Rejected callers get the cached answer instead of an exception. The fallback method takes the same parameters as the original plus the exception as the last argument — the same convention as `@CircuitBreaker`'s fallback. A fallback that returns stale data, a default, or an empty result is usually the right degradation; a fallback that returns HTTP 200 with an empty body while the operation silently failed is not — make the degradation visible to the caller.

The dependency for Boot 4 is `io.github.resilience4j:resilience4j-spring-boot4:2.4.0` — the `spring-boot3` starter is the Boot 3 equivalent, and the old `resilience4j-spring-boot2` naming is gone. The starter also needs `spring-boot-starter-aspectj` (the annotations are AOP aspects) and it registers the bulkhead registries from `application.properties` at startup, plus Actuator health indicators and Micrometer metrics if those are on the classpath.

## Option 3: Resilience4j thread-pool bulkhead — don't even lend the thread

The semaphore bulkhead limits how many calls run at once, but the calls still run on the *caller's* thread. If the slow call blocks for 2 seconds, it blocks a request thread for 2 seconds — just fewer of them. The thread-pool bulkhead goes one step further: it runs the call on a dedicated pool, so the caller thread is never occupied at all.

```java
@Bulkhead(name = "slow-tp", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<String> slowAsync() {
    return CompletableFuture.completedFuture(downstream.call());
}
```

```properties
resilience4j.thread-pool-bulkhead.instances.slow-tp.max-thread-pool-size=2
resilience4j.thread-pool-bulkhead.instances.slow-tp.core-thread-pool-size=2
resilience4j.thread-pool-bulkhead.instances.slow-tp.queue-capacity=1
```

The method must return a `CompletableFuture` — the aspect submits the work to the bulkhead's pool and hands you the future. With a pool of 2 and a queue of 1, five calls admit exactly three (two running, one queued) and reject two:

```java
@Test
void resilienceThreadPoolIsolates() {
    List<CompletableFuture<String>> futures = new ArrayList<>();
    for (int i = 0; i < 5; i++) {
        futures.add(resilienceService.slowAsync());
    }
    int ok = 0;
    int rejected = 0;
    for (CompletableFuture<String> f : futures) {
        try {
            assertEquals("ok", f.join());
            ok++;
        } catch (CompletionException e) {
            assertInstanceOf(BulkheadFullException.class, e.getCause());
            rejected++;
        }
    }
    assertEquals(3, ok, "2 running + 1 queued complete");
    assertEquals(2, rejected, "2 rejected");
    assertTrue(downstream.threadsUsed().stream().anyMatch(t -> t.contains("bulkhead")),
            "work ran on bulkhead threads, saw " + downstream.threadsUsed());
    assertFalse(downstream.threadsUsed().contains("main"),
            "caller thread never did the work");
    assertEquals(2, downstream.maxInFlight());
}
```

Note how rejection arrives: through the annotation, a full pool never throws at the call site. The aspect converts the `BulkheadFullException` into an exceptionally completed future (the programmatic `ThreadPoolBulkhead` API throws it synchronously instead), so handle it where you consume the future — `join()`, `handle()`, or `exceptionally()` — not in a `try` around the call. In the test above I run all five calls and none threw synchronously. And note what the thread assertion proves: the work ran on threads named for the bulkhead, and the calling thread never touched the slow call. That is the isolation property a semaphore cannot give you: even the *waiting* is offloaded. The cost is the pool itself — threads, queue memory, and a `CompletableFuture` API that propagates through your service layer. Use it for calls that block (slow HTTP, legacy SOAP, JDBC) and keep the semaphore for calls that are merely concurrent.

## Sizing: the only rule that matters

A bulkhead limit is a guess about three numbers, and getting any of them wrong makes the bulkhead either decoration or a new bottleneck:

1. **How much the downstream can actually take.** If the provider handles 20 concurrent requests before its own latency explodes, a limit of 50 protects nothing and a limit of 5 wastes capacity you paid for. Measure the downstream, or start from its documented concurrency and tune from metrics.
2. **How much of your own pool you're willing to lend.** A semaphore limit of 25 on a 200-thread Tomcat pool means one dependency can never take more than an eighth of your server. A limit of 250 on the same pool means the bulkhead never engages — you have built a wall with no wall in it.
3. **The connection pool underneath.** If the slow call is a database query, the bulkhead limit and the HikariCP pool size are the same budget counted twice: 20 bulkhead permits feeding a 10-connection pool means 10 callers hold permits while waiting on connections the pool has not got to give, burning request threads until Hikari's connection timeout. Keep the bulkhead limit at or below the pool size of whatever the call actually consumes.

One instance per dependency, not one global limit — the whole point is that the payment provider's slowness must not spend the inventory service's budget. And watch the metrics the starter already exports: available concurrent calls, rejected calls, and wait time. A bulkhead that never rejects is either sized right or never engaged; a bulkhead that rejects constantly is a capacity conversation, not a config tweak.

## What bulkheads don't fix

A bulkhead contains a failure; it does not repair it. The slow dependency is still slow, and the rejected callers still need an answer — a fallback, a retry with backoff, or an honest error. Pair the bulkhead with a circuit breaker when the dependency is *failing*, not just slow: the breaker stops calling it at all for a while, the bulkhead caps the damage while it is merely slow. The two compose in that order — breaker outside, bulkhead inside — because there is no point spending a permit on a call the breaker already knows will fail. (That is also Resilience4j's default aspect order, and it means bulkhead rejections can count toward the breaker's failure rate unless you ignore `BulkheadFullException`. The [circuit breaker post](https://blogs.rameskum.com/posts/circuit-breakers-resilience4j-spring-boot/) covers the pairing and that gotcha in detail.)

A bulkhead also does not help when the bottleneck is your own CPU or memory: capping concurrency on a method that burns CPU just moves the queue, it doesn't shrink it. And a per-instance limit is per *process* — ten replicas behind a load balancer each allow 20 concurrent calls, so the downstream sees 200. For a global limit you need the downstream, a gateway, or a distributed limiter to enforce it.

## The checklist

| Approach | What it caps | When the limit is hit | Fallback | Cost |
| --- | --- | --- | --- | --- |
| Native `@ConcurrencyLimit` (BLOCK) | Concurrent invocations | Caller waits | None | Zero dependencies |
| Native `@ConcurrencyLimit` (REJECT) | Concurrent invocations | `InvocationRejectedException` | None | Zero dependencies |
| Resilience4j semaphore | Concurrent calls, with a wait budget | `BulkheadFullException` | Yes | One library |
| Resilience4j thread-pool | Threads + queue, caller thread never blocked | `BulkheadFullException` | Yes | One library + `CompletableFuture` API |

Pick the native annotation when the cap is the whole requirement. Pick the Resilience4j semaphore when overload should fail fast or degrade to a fallback. Pick the thread-pool when the call blocks and the caller thread is too valuable to lend. And whatever you pick, write the saturation test first — two slow calls, one fast call, a stopwatch — because a bulkhead you have never seen engage is a bulkhead you have only imagined.

## Sources

- Spring Framework reference, [Resilience Features](https://docs.spring.io/spring-framework/reference/core/resilience.html) — `@ConcurrencyLimit`, `ThrottlePolicy`, `@EnableResilientMethods`.
- Spring blog, [Core Spring Resilience Features: @ConcurrencyLimit, @Retryable, and RetryTemplate](https://spring.io/blog/2025/09/09/core-spring-resilience-features/) — the Framework 7.0 announcement.
- Resilience4j docs, [Bulkhead](https://resilience4j.readme.io/docs/bulkhead) and [Getting Started](https://resilience4j.readme.io/docs/getting-started-3) — semaphore vs thread-pool, configuration properties, fallback.
- Resilience4j 2.4.0 + `resilience4j-spring-boot4` (Maven Central; 2.4.0 is the only published version as of Nov 2026) — the Boot 4 starter; `resilience4j-spring-boot3` is the Boot 3 equivalent.
- All 7 tests run on Spring Boot 4.1.2 (Spring Framework 7.0.10, Resilience4j 2.4.0, JUnit 6.0.3, JDK 25), each run twice: native BLOCK/REJECT, Resilience4j semaphore fast-fail + fallback, thread-pool isolation, and the saturation demo with and without a bulkhead — all passing.
