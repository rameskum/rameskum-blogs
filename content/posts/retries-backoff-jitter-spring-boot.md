---
title: 'Retries Done Right: Backoff, Jitter, and the Storm That Takes Down Your Service'
date: 2026-11-20T21:45:00-05:00
draft: false
ShowToc: true
description: >-
  A downstream blips for two seconds and your service answers with forty
  requests. Retries are the only resilience pattern that can make an outage
  worse, and the difference between a retry that saves you and one that
  finishes the job is three settings: a timeout, a backoff with jitter, and
  a circuit breaker. This post builds the failure, measures it, then builds
  the chain with Spring Framework 7's built-in @Retryable — no spring-retry,
  no extra dependency — and Resilience4j for the breaker. Thirteen tests,
  run twice, every number below measured.
tags:
  - spring-boot
  - java
  - resilience
  - microservices
categories: article
keywords:
  - spring boot retry
  - Retryable backoff jitter
  - retry storm
  - resilience4j circuit breaker
  - spring framework 7 resilience
---

A payment provider goes down for two seconds. Ten of your requests are in flight when it happens, so ten of them fail. Each one retries three times. That is forty requests arriving at a service that is already struggling to stand up — and if every client waits exactly the same 200 milliseconds before retrying, those forty requests arrive in four synchronized waves, each one landing the instant the provider is most fragile. The provider recovers, sees the wave, falls over again, and your retries start their next round. This is a retry storm, and it is the only resilience pattern that can make an outage worse than doing nothing.

I measured it. Ten callers, each retrying three times against a failing downstream: **40 downstream hits** from 10 original requests. With jitter, those 40 hits spread over **548 milliseconds** instead of landing in lockstep. The rest of this post is the chain that keeps the first number from happening: a timeout so a call cannot hang forever, a retry with exponential backoff and jitter so the load spreads out, and a circuit breaker so a dead downstream stops receiving traffic at all. Each layer fixes a different failure, and each one has a trap that the tests below caught.

The setup: Spring Boot 4.2.0 (Spring Framework 7.1.0), Resilience4j 2.4.0, JUnit 6.1.3, JDK 25. The "downstream" is a fake that fails a configurable number of times and counts every call, so every number in this post is a count or a timestamp, not an assertion. Thirteen tests, run twice, all passing. The full harness is a plain `AnnotationConfigApplicationContext` — no server, no live downstream — and the verification note at the end says exactly what was and was not run.

## The chain, in one picture

A call to a dependency goes through three layers, in this order:

1. **Timeout.** The call must fail fast instead of hanging. Without it, a slow downstream holds your threads hostage while the retry logic waits politely.
2. **Retry with backoff and jitter.** Transient failures get a second chance, but each attempt waits longer than the last, and the wait is randomized so clients do not retry in lockstep.
3. **Circuit breaker.** When failures pass a threshold, the breaker opens and calls fail immediately without touching the downstream at all, until a trial call proves it has recovered.

The order matters. A retry without a timeout retries a call that is still hanging. A breaker without a retry trips on the first blip instead of the first real outage. A timeout without either is honest but unforgiving — every blip becomes a user-facing error. The three together are the standard chain, and the rest of this post builds each one and shows what breaks when you skip a step.

## Layer 1: the timeout — a call that cannot hang

Spring Boot 4.2 deprecates `RestTemplate` and `TestRestTemplate` support in favour of `RestClient` and `RestTestClient`, so the client here is `RestClient` with an explicit read timeout on its request factory:

```java
JdkClientHttpRequestFactory factory = new JdkClientHttpRequestFactory();
factory.setReadTimeout(Duration.ofMillis(200));
RestClient client = RestClient.builder().requestFactory(factory).build();

String body = client.get()
        .uri("http://127.0.0.1:" + port + "/slow")
        .retrieve()
        .body(String.class);
```

The test points this client at a local server that sleeps two seconds before answering. The call fails with a `ResourceAccessException` in about 200 milliseconds — not after the two-second sleep. Without the timeout, the same call waits the full two seconds, and a retry policy wrapped around it would wait two seconds *per attempt* before it even starts counting.

Boot also offers global defaults — `spring.http.clients.connect-timeout` and `spring.http.clients.read-timeout` — if you want every client in the service to share a budget. But set the timeout per client when the dependencies differ: a ledger inside your cluster and a bank gateway across the internet should never share a timeout, because the same 5 seconds is generous for one and reckless for the other.

One rule before the next layer: a timeout is what makes a retry safe to attempt at all. Retry a call with no timeout and a slow downstream turns each attempt into a thread held for the whole hang, multiplied by the retry count. That is how a two-second blip becomes a thread-pool outage, which is the failure the [bulkheads guide](/posts/bulkheads-spring-boot/) measures.

## Layer 2: the retry — Spring Framework 7 has one built in

This is the part that changed with Boot 4. The retry support most Spring code uses today comes from the separate `spring-retry` library, enabled with `@EnableRetry`. Since Framework 7.0, retry lives in the core framework itself — `org.springframework.resilience.annotation.Retryable`, in `spring-context`, switched on with `@EnableResilientMethods`. The Spring Retry project is in maintenance mode. You need no extra dependency, and you need no `spring-boot-starter-aspectj` either — a claim you will find in migration guides that my harness disproves: every test below ran with no AspectJ weaver on the classpath at all. (Resilience4j's own annotations *are* AspectJ aspects and do need the starter. Framework's `@Retryable` is a plain bean post-processor, and that is a different mechanism.)

```java
@Configuration
@EnableResilientMethods
class RetryConfig { }

@Component
class PaymentService {
    private final FlakyDownstream downstream;

    @Retryable(includes = IOException.class, maxRetries = 3, delay = 100, multiplier = 2, maxDelay = 1000)
    public String charge() throws IOException {
        return downstream.call();
    }
}
```

The defaults, from the Framework reference docs and confirmed by the test run: at most **3 retry attempts after the initial failure**, a **1-second delay** between attempts, and the last exception propagated to the caller when the attempts run out. Total attempts is always `1 + maxRetries` — a method with the default annotation is invoked up to four times, and the test proved it: a downstream that fails three times and then succeeds returns `"ok"` on the fourth call, after about three seconds of one-second waits.

The attributes, all on the one annotation:

| Attribute | Meaning |
| --- | --- |
| `includes` / `excludes` (`value` is a shortcut for `includes`) | Which exceptions are worth retrying |
| `maxRetries` | Attempts *after* the first call (default 3) |
| `delay` | Initial wait in milliseconds (default 1000) |
| `multiplier` | Backoff factor applied to the wait |
| `maxDelay` | Ceiling on the wait |
| `jitter` | Randomness added to the wait |
| `timeUnit` | Unit for `delay`, `maxDelay` and `jitter` |
| `timeout` | Cap on the total time spent retrying — once it passes, no further attempt starts |
| `predicate` | A `MethodRetryPredicate` class, for anything subtler than exception types |

Two things Framework 7 does *not* ship: a circuit breaker and a rate limiter. The reference docs list exactly `@Retryable`, `@ConcurrencyLimit`, and programmatic retry. For a breaker you still need Resilience4j, which is layer 3 below.

If you need the programmatic form — a retry around a few lines rather than a whole method — it is `RetryTemplate` with a `RetryPolicy`, and the builder lives on the policy, not the template. This is the one place a tutorial snippet will mislead you: `RetryTemplate.builder()` does not exist, and the compiler says so.

```java
RetryPolicy policy = RetryPolicy.builder()
        .maxRetries(2)
        .delay(Duration.ofMillis(50))
        .build();
RetryTemplate template = new RetryTemplate(policy);
String result = template.execute(() -> downstream.call());
```

Note that `execute` declares a checked `RetryException`, so the lambda's exceptions surface as `RetryException` when the policy is exhausted rather than as the original exception — one more reason the annotation form is the default choice and the template is the escape hatch.

### Trap 1: the default retries everything, including the failures that can never succeed

With no `includes`, `@Retryable` retries *every* exception. A 400 Bad Request, a validation failure, a `NullPointerException` from your own bug — all retried three times, one second apart, to no effect except delay and load. The test for this one is blunt: a method annotated with `includes = IOException.class` that throws `IllegalStateException` hits the downstream exactly once.

```java
@Retryable(includes = IOException.class, maxRetries = 3, delay = 10)
public String chargeIllegal() {
    downstream.callIllegal(); // throws IllegalStateException
    return "ok";
}
```

The rule: always say which failures are transient. For an HTTP client that is usually `ResourceAccessException` (the timeout and connection failures) plus the 5xx statuses and 429 — and never the 4xx statuses, which are the downstream telling you the request itself is wrong. If the retry decision needs more than an exception type — retry on 503 but not on 500, say — that is what `predicate` is for.

### Trap 2: the two libraries count to three differently

This is the trap that bites during a migration from spring-retry or Resilience4j to the built-in annotation, because both libraries use the number 3 and mean different things by it.

- Framework's `@Retryable`: `maxRetries = 3` means three retries **after** the first call. Four invocations.
- Resilience4j's `Retry`: `maxAttempts = 3` means three attempts **in total**. Three invocations.

Both defaults are 3. The test proved both sides: the default `@Retryable` ran the method four times, and `Retry.ofDefaults("payment")` against an always-failing downstream ran it three times. If you port a Resilience4j config to `@Retryable` by copying the number, you silently add one extra attempt to every failing call — and in a storm, one extra attempt per caller is the difference between 30 and 40 downstream hits.

### Trap 3: calling your own method skips the retry entirely

Like `@Transactional`, `@Retryable` works through a proxy that Spring wraps around the bean. A call from another bean goes through the proxy and gets retried. A call from inside the same class goes through `this` and does not.

```java
@Retryable(includes = IOException.class, maxRetries = 3, delay = 10)
public String charge() throws IOException { return downstream.call(); }

public String chargeViaSelf() throws IOException { return charge(); } // no retry
```

The test calls `chargeViaSelf()` against an always-failing downstream: one downstream hit, exception propagated immediately. The same applies to `final` methods and `final` classes, which cannot be subclassed into a proxy at all. If a retry silently never happens, check the call path before the configuration — the annotation is fine, the call never reached the proxy.

### Trap 4: without the switch, the annotation is silently ignored

Spring Boot does not turn resilience on for you. A `@Retryable` method in a context without `@EnableResilientMethods` runs exactly once, with no warning, no log line, and no error. The test built the same beans in a plain `@Configuration` class and the always-failing downstream was hit once. This is the most expensive trap in the list, because everything looks right — the annotation is there, the code compiles, the method runs — and the retry simply never happens. It is also the easiest to test for: one test that fails the downstream once and counts the calls.

### Trap 5: fixed backoff synchronizes your retries — jitter is what spreads them

Backoff alone does not fix the storm. It only moves it. With a fixed 200-millisecond wait, every client that failed at the same instant retries at the same instant — four synchronized waves instead of one synchronized flood. Jitter breaks the synchronization by randomizing the wait, and the exact semantics matter: in Framework 7, jitter is *added* to the delay, a uniform random amount between 0 and the jitter value. It is not plus-or-minus.

The measurement: `delay = 200, jitter = 100, multiplier = 1`, five runs, three waits each. The nine gaps between consecutive attempts measured **232, 277, 289, 234, 291, 237, 222, 237, and 292 milliseconds** — every one of them above the 200-millisecond base and below the 300-millisecond ceiling. No jitter, and every gap would have been exactly 200.

The storm test then shows what that spread buys. Ten threads, each retrying three times with `delay = 100, jitter = 100`: still 40 downstream hits — jitter does not reduce the number of attempts — but those hits spread over **548 milliseconds** instead of arriving in four lockstep waves. Jitter spreads load; it does not reduce it. Reducing the load is the breaker's job.

The backoff schedule itself is exact and worth knowing by heart. With `maxRetries = 5, delay = 100, multiplier = 2`, the waits are 100, 200, 400, 800, and 1600 milliseconds — 3,100 milliseconds total before the final failure surfaces. The test measured the elapsed time at just over 3.1 seconds with six downstream calls. Set `maxDelay` to cap the growth, or a service that retries a long-dead dependency will wait minutes between attempts that were never going to succeed — which is where `timeout`, the cap on total retry time, earns its place.

## Layer 3: the circuit breaker — stop calling a dead downstream

A retry policy answers "should I try again?" A breaker answers "should I try at all?" Resilience4j's breaker watches the failure rate over a sliding window, and when it passes the threshold it opens: calls fail immediately with `CallNotPermittedException`, the downstream is not touched, and your threads are freed in milliseconds instead of waiting out timeouts and retries. After a wait, it lets one trial call through in the half-open state; if that succeeds, it closes again.

```java
CircuitBreakerConfig config = CircuitBreakerConfig.custom()
        .failureRateThreshold(50)
        .slidingWindowSize(4)
        .minimumNumberOfCalls(4)
        .waitDurationInOpenState(Duration.ofMillis(200))
        .build();
CircuitBreaker breaker = CircuitBreaker.of("payment", config);
Supplier<String> guarded = CircuitBreaker.decorateSupplier(breaker, downstream::callUnchecked);
```

The test: four failures, and the breaker is open. A fifth call throws `CallNotPermittedException` and the downstream call count stays at four — the open breaker fails fast, exactly as advertised.

For the Spring wiring, the dependency is `io.github.resilience4j:resilience4j-spring-boot4:2.4.0` — the Boot 4 starter, which registers the breaker registries from `application.properties` at startup. There is no `spring-boot-starter-resilience` artifact; I checked Maven Central and it does not exist. The starter's annotations are AspectJ aspects, so *that* path does need `spring-boot-starter-aspectj` — the opposite of Framework's built-in `@Retryable`, which does not.

### Trap 6: one lucky call does not close the breaker

After the wait expires, the breaker goes half-open and permits trial calls. My first version of this test assumed one successful trial call would close it. It does not. The breaker stayed `HALF_OPEN`, because the default `permittedNumberOfCallsInHalfOpenState` is **10** — it needs ten consecutive successful trial calls before it trusts the downstream again. The test asserts that default directly from `CircuitBreakerConfig.ofDefaults()`, and the passing version sets the number to 1 explicitly so a single success closes it.

That default is the right one for production: one success after an outage proves almost nothing, and a breaker that reopens the floodgate on a single lucky call replays the storm it just stopped. But it is a default you should know is there, because a breaker that stays half-open through your first few successful calls looks broken if you expected it to close immediately — and "looks broken" is how a correct breaker gets deleted in an incident review.

## The chain, assembled

Put together, one dependency gets one client, one retry policy, and one breaker:

```java
@Configuration
@EnableResilientMethods
class PaymentConfig {

    @Bean
    RestClient paymentRestClient(RestClient.Builder builder) {
        return builder
                .requestFactory(factoryWithTimeouts())
                .baseUrl("https://payments.internal")
                .build();
    }

    private JdkClientHttpRequestFactory factoryWithTimeouts() {
        var httpClient = HttpClient.newBuilder()
                .connectTimeout(Duration.ofMillis(500))
                .build();
        var factory = new JdkClientHttpRequestFactory(httpClient);
        factory.setReadTimeout(Duration.ofMillis(2000));
        return factory;
    }

    @Bean
    CircuitBreaker paymentBreaker() {
        return CircuitBreaker.of("payment", CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .slidingWindowSize(10)
                .minimumNumberOfCalls(10)
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .build());
    }
}

@Component
class PaymentService {
    private final RestClient rest;
    private final CircuitBreaker breaker;

    @Retryable(includes = ResourceAccessException.class, maxRetries = 3,
               delay = 200, multiplier = 2, maxDelay = 2000, jitter = 100)
    public String charge(String id) {
        return breaker.executeSupplier(() ->
                rest.get().uri("/charges/{id}", id).retrieve().body(String.class));
    }
}
```

Read it from the inside out: the `RestClient` fails fast on a hung downstream, `@Retryable` gives a transient failure three more chances with waits of roughly 200, 400, and 800 milliseconds — each plus up to 100 milliseconds of jitter — and the breaker stops the calls entirely once half of the last ten are failing. One thing to notice about the nesting: the retry wraps the breaker here, so the breaker records every attempt. A request that succeeds on its third try still logs two failures in the breaker, which means the breaker can open during exactly the blips the retry exists to absorb. If that bothers you — and it should — invert it: put the breaker outside the retry, so the breaker judges the logical call and the retry judges each attempt. In practice that means the breaker decorates the call to the `@Retryable` method, not the HTTP call inside it.

## When not to retry

Retries are for transient failures of idempotent operations. A GET that failed on a connection reset is safe to repeat. A POST that charges a card is not — unless the downstream is idempotent or you send an idempotency key with the request, so the second attempt is recognised as the same charge rather than a new one. Retry a non-idempotent write and the storm is not your worst problem: the double charge is.

The same rule applies to the failure itself. Retry timeouts, connection resets, 503s, and 429s — the failures that say "try again later." Do not retry 400s, 401s, or 404s — the failures that say "the request is wrong." And put a budget on the whole thing: `timeout` caps the total time a single call may spend retrying, so a retry policy can never turn a fast failure into a slow one. A retry that takes longer than the caller's own timeout is not resilience, it is a queue.

This is also where the chain meets the [circuit-breaker guide](/posts/circuit-breakers-resilience4j-spring-boot/) and the [bulkheads guide](/posts/bulkheads-spring-boot/): the breaker decides *whether* to call, the bulkhead decides *how much* of your service one dependency may consume, and the retry decides *how many chances* a single call gets. They are three different questions, and answering only one of them is how a service that is "resilient" on paper still falls over.

## Verification note

Thirteen tests, run twice on Spring Framework 7.1.0 (the version Spring Boot 4.2.0 ships — confirmed in the Boot BOM, `spring-framework.version` 7.1.0), Resilience4j 2.4.0, JUnit 6.1.3, Temurin JDK 25.0.5. The measured numbers: default `@Retryable` runs the method four times over ~3 seconds; the 100/200/400/800/1600 backoff totals ~3.1 seconds over six calls; jittered waits measured 222–292 milliseconds for a 200-millisecond base with 100 milliseconds of jitter; ten callers produce 40 downstream hits, spread over 548 milliseconds with jitter; the breaker opens after four failures at a 50% threshold and the downstream count stays at four while it is open; the half-open default is ten trial calls before closing.

Two things were corrected by the run itself. First, `RetryTemplate.builder()` does not exist — the builder is `RetryPolicy.builder()`, and the first compile failed on exactly that. Second, my first half-open test expected one success to close the breaker; the breaker stayed half-open, and the default of ten trial calls is the correct behaviour, not a bug. Both corrections are in the post above.

What was *not* run: no live downstream, no deployed service, no load test against real infrastructure. The storm is ten threads in one process against a fake that counts calls, and the timeout test is a local server that sleeps two seconds — it ran the same factory method as the assembled chain, with a 200-millisecond read timeout so the test stays fast, where the post shows 2 seconds as the production value. The numbers are real measurements of the retry, backoff, jitter, and breaker behaviour — not of a production topology. If your downstream has its own retry behaviour, its own timeouts, or a load balancer that retries for you, the amplification multiplies across those layers too, and that is a measurement worth making against your own stack before you ship the config.
