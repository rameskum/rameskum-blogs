---
title: 'Circuit Breakers in Spring Boot: Resilience4j, Done Right'
date: 2026-09-30T06:59:00-04:00
draft: false
ShowToc: true
description: >-
  A circuit breaker stops your service from repeatedly invoking a dependency
  after observed failures or excessive latency cross configured thresholds.
  This guide wires up Resilience4j in Spring Boot: the three states, the
  configuration that actually matters, bulkhead pairing, and the five config
  mistakes that quietly neuter breakers in production.
tags:
  - spring-boot
  - java
  - backend
  - microservices
  - system-design
  - interview
categories: article
---

Your checkout service calls the payment gateway. One night the gateway has a bad deploy and starts timing out. Every synchronous checkout request thread now remains occupied for the full 10-second client timeout. Threads pile up, the pool exhausts, health checks fail, and the load balancer starts draining your perfectly healthy checkout pods. Nothing about checkout is broken — it just never stopped calling a dependency that was already dead.

Retries alone don't fix this; they make it worse. Three retries per request turns one timeout into four, and every attempt occupies a thread for another 10 seconds. (In Resilience4j, `maxAttempts` includes the initial call, so four total attempts means `maxAttempts: 4` — don't confuse "three retries" with the framework's attempt counting.) What you want is a component that notices the failure rate, stops calling the gateway for a while, and fails fast instead. That is the circuit breaker.

## The three states

A circuit breaker sits in front of the remote call and normally operates in three primary states — CLOSED, OPEN, and HALF_OPEN (Resilience4j also documents special states like METRICS_ONLY, DISABLED, and FORCED_OPEN):

```text
        failure rate >= threshold
 CLOSED ---------------------------> OPEN
   ^                                   |
   | probes succeed                    | wait duration elapsed
   |                                   | + incoming call arrives
   |                                   v
   +----------- HALF_OPEN <------------+
              (limited probes)
```

- **CLOSED** — normal operation. Calls pass through; outcomes are recorded in a sliding window.
- **OPEN** — the failure rate in the window crossed the threshold. Calls are rejected immediately with a `CallNotPermittedException`; the failing dependency gets a breather.
- **HALF_OPEN** — after the open-state wait elapses, the next incoming call becomes a probe (under the default configuration, the transition itself needs a call to arrive). A limited number of probe calls are allowed through: if they succeed, the breaker closes; if they fail, it re-opens.

OPEN does not mean forever. The breaker remains OPEN for at least `wait-duration-in-open-state`, then transitions to HALF_OPEN when the configured transition mechanism allows it.

## Setup

This example uses Resilience4j 2.4.0. For Spring Boot 4, use the `resilience4j-spring-boot4` starter; Boot 3 stacks use `resilience4j-spring-boot3`. The annotations are applied through AOP, so `spring-boot-starter-aop` is required. This example assumes Java 17+ and Spring Boot 4 — Resilience4j 2.x requires Java 17, and copying this into an older project will produce confusing build failures. (Check Maven Central for newer releases before pinning the version.)

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot4</artifactId>
    <version>2.4.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## The minimal breaker

Put the annotation on the method that makes the remote call — not on the controller, not on a method that calls it indirectly. The annotation is applied through a Spring proxy, so placing it at the boundary where the protected operation actually occurs keeps the resilience policy easy to reason about and avoids proxy/self-invocation surprises:

```java
@Slf4j   // Lombok; or declare the logger explicitly
@Service
public class PaymentsClient {

    private final RestClient payments;

    public PaymentsClient(RestClient.Builder builder) {
        this.payments = builder.baseUrl("https://payments.internal").build();
    }

    @CircuitBreaker(name = "payments", fallbackMethod = "chargeFallback")
    public ChargeResult charge(ChargeRequest request) {
        return payments.post()
                .uri("/charges")
                .body(request)
                .retrieve()
                .body(ChargeResult.class);
    }

    // Same return type, same parameters, optional Throwable
    // (or a more specific exception like CallNotPermittedException) last.
    private ChargeResult chargeFallback(ChargeRequest request, Throwable failure) {
        log.warn("payments breaker engaged, failing open for {}", request.orderId(), failure);
        return ChargeResult.deferred(request.orderId());
    }
}
```

The fallback contract is strict: the fallback method must match the guarded method's return type and arguments, optionally followed by a Throwable (or compatible exception type). If no compatible fallback signature can be resolved, the original exception propagates instead.

The fallback decides your failure posture. Returning a `deferred` marker that the rest of the system understands is a deliberate degraded mode. Returning an empty list that downstream code treats as authoritative truth is a lie that will page you at 3 a.m. Choose fallbacks that are honest about what happened.

## The configuration that actually matters

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: count_based      # or time_based
        sliding-window-size: 20               # 20 calls, or 20 seconds for time_based
        minimum-number-of-calls: 10           # don't evaluate the rate before this many calls
        failure-rate-threshold: 50            # open at >= 50% failures
        slow-call-duration-threshold: 2s      # calls slower than this count as slow
        slow-call-rate-threshold: 80          # open at >= 80% slow calls
        wait-duration-in-open-state: 30s      # cooldown before probes
        permitted-number-of-calls-in-half-open-state: 3   # probes
        automatic-transition-from-open-to-half-open-enabled: false
        ignore-exceptions:                    # business errors, not outages
          - org.springframework.web.client.HttpClientErrorException$NotFound
    instances:
      payments:
        base-config: default
      fraud-check:
        base-config: default
        failure-rate-threshold: 30            # override per dependency
```

A few of these deserve unpacking, because the defaults are the first trap.

**The sliding window type.** `count_based` (the default) evaluates the last N calls: trip based on the last N calls. `time_based` evaluates calls from the last N seconds: trip based on activity during the last N seconds. For low-traffic dependencies, a time-based window can better represent a recent outage, but `minimum-number-of-calls` still applies — the failure/slow-call rates aren't calculated until the minimum has been recorded. Make sure the minimum can actually be reached during the window.

**`minimum-number-of-calls` (default: 100).** minimum-number-of-calls controls when the failure rate is allowed to be calculated. If it is higher than the number of calls represented by your configured sliding window, the breaker effectively cannot evaluate the failure rate until enough calls have accumulated. Choose it deliberately based on your traffic volume and the amount of evidence you want before opening the breaker. With the defaults, the breaker needs 100 recorded calls before the failure rate can even be evaluated; at 10 calls/minute, that means roughly 10 minutes of failure before it can open. A dev test with five forced failures "proves" nothing because the rate was never computed — and if you shrink the window to 20 without lowering the minimum, the rate may never be evaluated at all.

**Slow calls.** Failures aren't the only signal. A dependency that returns 200 OK after 25 seconds is failing you just as thoroughly. `slow-call-duration-threshold` marks calls as slow, and `slow-call-rate-threshold` opens the breaker on slowness alone. The default slow-call rate threshold is 100%, so latency alone can open a default-configured breaker, but only when every evaluated call is slow. Most applications should configure a lower threshold if latency itself should open the breaker.

**`automatic-transition-from-open-to-half-open-enabled` (default: false).** With the default, the breaker leaves OPEN only when a call arrives after the wait expires — that first post-cooldown call becomes the probe. Set it to `true` and a monitoring thread transitions the breaker on a timer instead, so probes start without waiting for traffic; that thread has a resource cost, so only enable it when you need prompt recovery detection on low-traffic dependencies. Know which behavior you configured, because it changes what your first post-outage call experiences.

**`ignore-exceptions` vs `record-exceptions`.** By default every exception counts as a failure. If a 404 is a valid business outcome for this dependency, exclude it with `ignore-exceptions`; it still propagates to the caller, it just doesn't move the breaker. A 404 could equally indicate a misconfigured or wrongly-deployed endpoint, so don't write "404s are always business errors" as a universal rule — qualify it per dependency.

## Five mistakes that neuter breakers

**1. Leaving `minimum-number-of-calls` at 100 on a low-traffic endpoint.** With the defaults (a 100-call window, minimum 100), the breaker physically cannot open until 100 calls are recorded — and if you shrink the window without lowering the minimum, the failure rate may never be evaluated at all. Your staging test fails 8 calls in a row, the breaker stays closed, and you conclude "breakers don't work." Size the minimum to the traffic the dependency actually sees.

**2. Counting business errors as failures.** Covered above — `ignore-exceptions` for 4xx-style responses that represent correct behavior. The corollary: don't ignore the exceptions that represent real failure either. An over-broad `ignore-exceptions: java.lang.Exception` can effectively disable failure-based opening, because exceptions no longer contribute to the failure rate. (Note the qualifier: a breaker can also open on slow-call rate, so "can never open" would be too absolute.)

**3. Self-invocation.** The annotation works through a Spring AOP proxy. If `charge()` calls another `@CircuitBreaker`-annotated method on the same bean, the proxy is bypassed and the annotation does nothing — no counting, no opening, no fallback. Put the remote call on its own bean (like `PaymentsClient` above) so every call goes through the proxy.

**4. Retrying blindly alongside the breaker.** When you stack `@Retry` and `@CircuitBreaker` on one method, the annotation order in your code does not decide the nesting — two separate Spring AOP aspects do. Resilience4j's documented default is Retry outside CircuitBreaker (`Retry(CircuitBreaker(...))`), so every retry attempt passes through the breaker and is recorded as its own call. On a dead dependency, three attempts per logical call means three recorded failures and three times the hammering — the retry policy trips the breaker faster than the outage would, and a `CallNotPermittedException` from the inner breaker can itself be retried by the outer retry, which is pure waste.

Decide explicitly which nesting you want, because they produce different behavior and metrics:

- **Retry outside the breaker** (the default): each retry attempt is visible to the breaker. Retries of a genuinely dead dependency inflate the recorded failure rate and can trip the breaker on transient blips a single attempt would have absorbed.
- **Breaker outside retry**: the whole retry sequence counts as one call — the breaker records only the final outcome. A call that fails once and succeeds on retry is recorded as a single success.

For a payments call you usually want the second, so make it explicit rather than relying on the default:

```yaml
resilience4j:
  circuitbreaker:
    circuit-breaker-aspect-order: 2   # higher value = outer aspect
  retry:
    retry-aspect-order: 1
```

One more rule: put `fallbackMethod` on the outermost annotation. With the breaker outside, the fallback belongs on `@CircuitBreaker` — put it on the inner `@Retry` and the first attempt's failure is swallowed before retry ever runs.

And keep `CallNotPermittedException` out of the retry trigger list. CallNotPermittedException means the breaker has already decided that the dependency should not be called; retrying it doesn't give the dependency another chance. Retry's default is to retry any exception unless configured otherwise, so make the exclusion explicit:

```yaml
resilience4j:
  retry:
    instances:
      payments:
        max-attempts: 3          # total attempts, including the initial call — not "3 retries"
        wait-duration: 200ms
        ignore-exceptions:
          - io.github.resilience4j.circuitbreaker.CallNotPermittedException
```

Keep the total attempt count small — `maxAttempts: 2` or `3` — because every attempt is another hit on a dependency that is already struggling.

**5. A fallback that lies.** `return List.of()` on a catalog fetch that callers render as "no products found" turns an outage into silent wrongness. Prefer fallbacks that degrade honestly: throw a `ServiceUnavailableException` the API layer maps to 503, return a cached value stamped as stale, or return a marker the caller branches on.

## Pair it with a bulkhead

The breaker fails fast; the bulkhead caps concurrency. A semaphore bulkhead limits how many executions may enter the remote call concurrently, preventing an overloaded dependency from consuming unbounded application concurrency while the circuit breaker gathers enough failures to trip. That stays correct across blocking, reactive, and virtual-thread models, where "threads" may not be the right unit.

Note what it does *not* do: a semaphore bulkhead does not provide thread-pool isolation. If 25 synchronous executions enter it and block on I/O, those application threads are still occupied — the bulkhead limits how many requests are admitted, but the admitted ones can still block on the dependency. If you need actual execution isolation, consider a `FixedThreadPoolBulkhead`.

One more distinction worth internalizing: the sliding-window size controls how outcomes are *measured*; it is not a concurrency limit. If 1,000 calls arrive while the breaker is CLOSED, all 1,000 can be admitted unless another mechanism — like this bulkhead — limits concurrency.

```yaml
resilience4j:
  bulkhead:
    instances:
      payments:
        max-concurrent-calls: 25
        max-wait-duration: 0   # don't queue for a permit; reject immediately
```

```java
@Bulkhead(
    name = "payments",
    type = Bulkhead.Type.SEMAPHORE,
    fallbackMethod = "bulkheadFallback"
)
@CircuitBreaker(
    name = "payments",
    fallbackMethod = "circuitBreakerFallback"
)
public ChargeResult charge(ChargeRequest request) { ... }

private ChargeResult bulkheadFallback(ChargeRequest request, BulkheadFullException ex) {
    log.warn("payments bulkhead saturated for {}", request.orderId());
    return ChargeResult.deferred(request.orderId());
}

private ChargeResult circuitBreakerFallback(ChargeRequest request, CallNotPermittedException ex) {
    log.warn("payments breaker open for {}", request.orderId());
    return ChargeResult.deferred(request.orderId());
}
```

Separate fallbacks make the two failure modes explicit: bulkhead full means *concurrency exhausted* (too many calls admitted at once), while the open breaker means the dependency *crossed the configured failure/slow-call threshold*. Each aspect's fallback handles the exception its own scope throws — the inner bulkhead fallback catches `BulkheadFullException`, the outer breaker fallback catches `CallNotPermittedException` and the recorded failures. You can share one fallback method if you want, but for teaching (and for distinct runbook responses), separate methods show which protection engaged.

By default, the bulkhead runs inside the circuit breaker. When saturated, it rejects with `BulkheadFullException`; unless you configure the circuit breaker to ignore that exception, those rejections can contribute to the breaker's failure rate. Together they bound both concurrency and repeated calls to a failing dependency.

## What a circuit breaker does not do

It helps to say what the pattern is *not*, because resilience features get conflated:

- It doesn't limit concurrency — that's the **bulkhead**.
- It doesn't retry — that's **retry**.
- It doesn't impose a request deadline — that's the **timeout** (client-side, per call).
- It doesn't make a dependency healthy. It just stops you from asking.
- It doesn't replace good client-side timeouts. Without them, every call still occupies a thread for the full wait while the breaker gathers statistics.

| Problem | Pattern |
|---|---|
| Dependency is failing repeatedly | Circuit breaker |
| Dependency is too slow | Timeout |
| Too many concurrent calls | Bulkhead |
| Transient failure | Retry |
| Need graceful degraded behavior | Fallback |

Pick the pattern that matches the failure you're actually defending against; stacking all of them on every call is how you get configuration nobody understands.

One related point, since this article's opening scenario is a 10-second client timeout: the circuit breaker *observes* call duration, but it does not enforce the deadline — that's the client's timeout. For asynchronous code built on `CompletableFuture`, Resilience4j's **TimeLimiter** is the primitive that bounds how long you'll wait, and it pairs with the breaker the same way a timeout does for blocking calls.

## Observe it

A breaker you can't see is a breaker you can't trust — but don't confuse visibility with liveness (more on that below). The endpoints exist, but they are not exposed by default:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,circuitbreakers,circuitbreakerevents
```

`GET /actuator/circuitbreakers` lists every instance and its current state; `/actuator/circuitbreakerevents` streams state transitions and call outcomes. With Micrometer on the classpath, Resilience4j publishes metrics such as `resilience4j.circuitbreaker.state` and `resilience4j.circuitbreaker.calls` (Prometheus exposes these using its naming conventions) — alert on the state gauge transitioning to OPEN, not just on the failure count. The transition is the event your runbook cares about.

To have breaker state appear in `/actuator/health`, you need two things — the health indicator registration *and* Spring Boot's health indicator toggle (they are separate concerns):

```yaml
management:
  health:
    circuitbreakers:
      enabled: true

resilience4j:
  circuitbreaker:
    configs:
      default:
        register-health-indicator: true
```

Read the warning below before you let an OPEN breaker take down your health endpoint.

### Don't let a breaker fail your health check

Exposing circuit-breaker state through application health has operational consequences. If a downstream dependency's breaker OPENs and your `/actuator/health` goes DOWN with it, you can build a chain reaction: dependency fails → breaker opens → health becomes DOWN → orchestrator/load balancer drains otherwise healthy instances → remaining instances get more traffic → more failures. The introduction to this article described exactly that drain; a breaker wired into liveness can recreate it.

So don't equate circuit-breaker state with application liveness or readiness automatically. A workable split:

- Keep breaker state on the dedicated endpoint (`/actuator/circuitbreakers`) and in metrics — that's your operational view.
- Feed it into health only for dependencies whose failure genuinely means *your* instance can't serve traffic (the primary database, say), not for every third-party API you call.

An OPEN breaker on a non-critical dependency must never remove a healthy pod.

## Test the states, not just the happy path

```java
@SpringBootTest
class PaymentsClientTest {

    @Autowired CircuitBreakerRegistry registry;
    @Autowired PaymentsClient client;

    @Test
    void opensAfterThresholdFailures() {
        CircuitBreaker breaker = registry.circuitBreaker("payments");
        forceFailures(breaker);
        assertThat(breaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }

    @Test
    void openBreakerRejectsAndUsesFallback() {
        CircuitBreaker breaker = registry.circuitBreaker("payments");
        forceFailures(breaker);
        // the next call is rejected immediately and hits the fallback
        assertThat(client.charge(failingRequest()).status())
                .isEqualTo(ChargeStatus.DEFERRED);
        assertThat(breaker.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }

    private void forceFailures(CircuitBreaker breaker) {
        // minimum-number-of-calls failures through the guarded method;
        // the fallback converts each one to DEFERRED, so no try/catch needed
        for (int i = 0; i < 10; i++) {
            client.charge(failingRequest());
        }
    }
}
```

If your test can't trip the breaker, neither can production — usually because `minimum-number-of-calls` is higher than the number of calls your test makes. The test and the config have to agree.

A stronger variant also verifies the dependency itself was never called after opening — with Mockito or WireMock, assert the stub saw exactly 10 invocations before and after the rejected call. That proves the fail-fast actually happened rather than inferring it from breaker state alone.

## The interview one-liner

> A circuit breaker stops calling a dependency that is already failing: once the failure rate over a recent window crosses a threshold, it rejects calls immediately for a cooldown, then transitions to HALF_OPEN and permits a configured number of probe calls before closing. In Resilience4j, size the sliding window and minimum calls to your actual traffic (and make the choice together with your latency and failure-detection objectives), don't count business errors as failures, keep retries small so they don't trip the breaker faster than the outage does, and pair it with a bulkhead so a slow dependency can't drain your thread pool while the breaker gathers statistics.
