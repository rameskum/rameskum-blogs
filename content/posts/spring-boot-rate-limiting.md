---
title: 'Rate Limiting in Spring Boot: The System Design Interview Answer, in Code'
date: 2026-09-20T12:40:00-04:00
draft: false
ShowToc: true
description: >-
  Rate limiting is a production necessity and a senior interview staple. This
  guide builds three working limiters in Spring Boot — fixed window, token
  bucket with Bucket4j, and a distributed sliding window on Redis with Lua —
  with tests, and shows how to talk through the tradeoffs in an interview.
tags:
  - java
  - spring-boot
  - rate-limiting
  - interview
categories: article
keywords:
  - spring boot
  - rate limiting
  - token bucket
  - sliding window
  - redis
  - system design interview
---

Rate limiting shows up in two places: your on-call rotation and your system design interview. Both reward the same thing — knowing which algorithm to pick, why, and what breaks at scale.

This guide builds three working limiters in Spring Boot, each with tests:

1. **Fixed window** — the simplest thing that works, hand-rolled.
2. **Token bucket** — smooth, burst-friendly, via Bucket4j.
3. **Distributed sliding window** — precise and shared across instances, on Redis with Lua.

Then we close with the part interviews actually grade: how to talk through the tradeoffs.

## The algorithms in 60 seconds

- **Fixed window:** count requests per key in the current window (e.g. 100/minute). Dead simple. Flaw: a burst at the end of one window plus a burst at the start of the next lets through 2x the limit.
- **Sliding window log:** store a timestamp per request, evict old ones, count what remains. Precise. Flaw: memory grows with request volume.
- **Token bucket:** tokens refill at a steady rate; each request spends one. Allows bursts up to the bucket capacity while keeping the average rate. The best default for APIs.
- **Leaky bucket:** like token bucket but requests queue and drain at a constant rate. Smooth output, adds latency under load.

For most services, token bucket is the answer. For "exactly N requests per rolling window" guarantees, sliding window on Redis is the answer. Fixed window is the answer when you want something simple on a single instance and can tolerate the boundary burst.

## Build 1: fixed window, hand-rolled

Start with the simplest limiter so the moving parts are visible. Inject a `Clock` so tests can control time without sleeping.

```java
public class FixedWindowRateLimiter {

    private final int maxRequests;
    private final Duration windowSize;
    private final Clock clock;
    private final ConcurrentHashMap<String, Window> windows = new ConcurrentHashMap<>();

    private record Window(long windowStartMillis, AtomicInteger count) {}

    public FixedWindowRateLimiter(int maxRequests, Duration windowSize) {
        this(maxRequests, windowSize, Clock.systemUTC());
    }

    FixedWindowRateLimiter(int maxRequests, Duration windowSize, Clock clock) {
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
        this.clock = clock;
    }

    public boolean tryAcquire(String key) {
        long now = clock.millis();
        Window window = windows.compute(key, (k, existing) -> {
            if (existing == null || now - existing.windowStartMillis() >= windowSize.toMillis()) {
                return new Window(now, new AtomicInteger(1));
            }
            existing.count().incrementAndGet();
            return existing;
        });
        return window.count().get() <= maxRequests;
    }
}
```

Wire it into a filter. The key can be the client IP, an API key, or the authenticated user id — pick per endpoint.

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private final FixedWindowRateLimiter limiter =
            new FixedWindowRateLimiter(100, Duration.ofMinutes(1));

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {
        String key = request.getRemoteAddr();
        if (!limiter.tryAcquire(key)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("Retry-After", "60");
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\":\"rate limit exceeded\"}");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

Two things to notice. The `429` status plus `Retry-After` header is the standard contract — clients and interviewers both expect it. And the map grows with distinct keys, so in production you need eviction (a `Cache` from Caffeine with `expireAfterAccess` is the usual fix) or you have a slow memory leak keyed by attacker-controlled input.

Test it with a controllable clock — no `Thread.sleep` in tests:

```java
class FixedWindowRateLimiterTest {

    private final AtomicReference<Instant> now =
            new AtomicReference<>(Instant.parse("2026-09-20T12:00:00Z"));
    private final Clock clock = new Clock() {
        @Override public ZoneId getZone() { return ZoneOffset.UTC; }
        @Override public Clock withZone(ZoneId zone) { return this; }
        @Override public Instant instant() { return now.get(); }
    };

    private FixedWindowRateLimiter limiter;

    @BeforeEach
    void setUp() {
        limiter = new FixedWindowRateLimiter(5, Duration.ofMinutes(1), clock);
    }

    @Test
    void allowsUpToLimitWithinWindow() {
        for (int i = 0; i < 5; i++) {
            assertTrue(limiter.tryAcquire("user-1"));
        }
        assertFalse(limiter.tryAcquire("user-1"));
    }

    @Test
    void resetsWhenWindowExpires() {
        for (int i = 0; i < 5; i++) limiter.tryAcquire("user-1");
        now.set(now.get().plusSeconds(61));
        assertTrue(limiter.tryAcquire("user-1"));
    }

    @Test
    void tracksKeysIndependently() {
        for (int i = 0; i < 5; i++) limiter.tryAcquire("user-1");
        assertFalse(limiter.tryAcquire("user-1"));
        assertTrue(limiter.tryAcquire("user-2"));
    }
}
```

## Build 2: token bucket with Bucket4j

For a burst-friendly limiter, reach for [Bucket4j](https://github.com/bucket4j/bucket4j) instead of hand-rolling refill math:

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j_jdk17-core</artifactId>
    <version>8.20.0</version>
</dependency>
```

One bucket per key, built lazily. The configuration below reads as "capacity 100, refill 100 tokens per minute, greedily":

```java
@Component
public class Bucket4jRateLimiter {

    private final ConcurrentHashMap<String, Bucket> buckets = new ConcurrentHashMap<>();

    public boolean tryConsume(String key) {
        return buckets.computeIfAbsent(key, this::newBucket).tryConsume(1);
    }

    private Bucket newBucket(String key) {
        return Bucket.builder()
                .addLimit(limit -> limit
                        .capacity(100)
                        .refillGreedy(100, Duration.ofMinutes(1)))
                .build();
    }
}
```

The same filter shape from Build 1 works — swap the limiter. Bucket4j also ships a Spring Boot starter (`com.giffing.bucket4j.spring.boot.starter`) if you prefer annotation-driven config, and a distributed mode backed by Redis or Postgres via its proxy manager.

Test the capacity behavior without time control — exhausting a bucket needs no clock:

```java
class Bucket4jRateLimiterTest {

    private final Bucket4jRateLimiter limiter = new Bucket4jRateLimiter();

    @Test
    void allowsBurstUpToCapacityThenRejects() {
        for (int i = 0; i < 100; i++) {
            assertTrue(limiter.tryConsume("user-1"));
        }
        assertFalse(limiter.tryConsume("user-1"));
    }

    @Test
    void bucketsArePerKey() {
        for (int i = 0; i < 100; i++) limiter.tryConsume("user-1");
        assertFalse(limiter.tryConsume("user-1"));
        assertTrue(limiter.tryConsume("user-2"));
    }
}
```

## Build 3: distributed sliding window on Redis

In-memory limiters are per-instance. With three replicas behind a load balancer, a client gets 3x your limit. Share state in Redis. To make the check-and-record step atomic — and avoid the race between "check count" and "record request" — run a Lua script on the Redis server.

The script keeps a sorted set of request timestamps per key, evicts entries outside the window, and only records the request if the count is under the limit:

```lua
-- src/main/resources/lua/sliding-window.lua
-- KEYS[1] = rate limit key
-- ARGV[1] = now (millis), ARGV[2] = window (millis),
-- ARGV[3] = limit, ARGV[4] = unique member
local windowStart = tonumber(ARGV[1]) - tonumber(ARGV[2])
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, windowStart)
local count = redis.call('ZCARD', KEYS[1])
if count < tonumber(ARGV[3]) then
    redis.call('ZADD', KEYS[1], ARGV[1], ARGV[4])
    redis.call('PEXPIRE', KEYS[1], ARGV[2])
    return 1
end
return 0
```

```java
@Component
public class RedisSlidingWindowLimiter {

    private final StringRedisTemplate redis;
    private final DefaultRedisScript<Long> script;

    public RedisSlidingWindowLimiter(StringRedisTemplate redis) {
        this.redis = redis;
        this.script = new DefaultRedisScript<>();
        this.script.setLocation(new ClassPathResource("lua/sliding-window.lua"));
        this.script.setResultType(Long.class);
    }

    public boolean tryAcquire(String key, int limit, Duration window) {
        long now = System.currentTimeMillis();
        String member = now + ":" + UUID.randomUUID();
        Long allowed = redis.execute(script,
                List.of("ratelimit:" + key),
                String.valueOf(now),
                String.valueOf(window.toMillis()),
                String.valueOf(limit),
                member);
        return allowed != null && allowed == 1L;
    }
}
```

Why this shape matters in an interview: the `ZREMRANGEBYSCORE` + `ZCARD` + `ZADD` sequence must be atomic or two concurrent requests can both read a count under the limit and both proceed. Lua gives you that atomicity without a distributed lock. The `PEXPIRE` keeps dead keys from accumulating.

For the test, spin up real Redis with Testcontainers — Lua behavior is exactly what you are testing, so do not mock it:

```java
@Testcontainers
class RedisSlidingWindowLimiterTest {

    @Container
    static GenericContainer<?> redis =
            new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);

    private RedisSlidingWindowLimiter limiter;

    @BeforeEach
    void setUp() {
        LettuceConnectionFactory factory = new LettuceConnectionFactory(
                redis.getHost(), redis.getMappedPort(6379));
        factory.afterPropertiesSet();
        limiter = new RedisSlidingWindowLimiter(new StringRedisTemplate(factory));
    }

    @Test
    void enforcesLimitAcrossCalls() {
        for (int i = 0; i < 5; i++) {
            assertTrue(limiter.tryAcquire("user-1", 5, Duration.ofMinutes(1)));
        }
        assertFalse(limiter.tryAcquire("user-1", 5, Duration.ofMinutes(1)));
        assertTrue(limiter.tryAcquire("user-2", 5, Duration.ofMinutes(1)));
    }
}
```

## Externalize the configuration

Hardcoded limits get stale. Bind them to properties so each environment — and each endpoint — can differ:

```yaml
rate-limit:
  requests-per-minute: 100
  window: 1m
```

```java
@ConfigurationProperties(prefix = "rate-limit")
public record RateLimitProperties(int requestsPerMinute, Duration window) {}
```

## How to talk through this in an interview

The code gets you in the door; the reasoning gets you the offer. Interviewers grade the discussion, so structure it:

**1. Clarify the dimension.** Per user, per IP, per API key, per endpoint, or global? Sustained rate, burst allowance, or both? "100 requests per minute per user" and "10,000 requests per second globally" are different systems.

**2. Propose with tradeoffs, not just a name.** "I'd start with token bucket: it allows short bursts, which real clients produce, while bounding the average. Fixed window is simpler but lets through 2x at the boundary — here's the scenario." Draw the boundary-burst case; it is the classic follow-up.

**3. Go distributed before they ask.** "In-memory state doesn't survive multiple replicas. I'd share counters in Redis, and use a Lua script so check-and-increment is atomic." Mention that sticky sessions are an alternative but push the complexity into routing.

**4. Name the failure modes.** What happens when Redis is down — fail open (availability) or fail closed (protection)? The answer depends on the endpoint: fail closed for payments, fail open for read APIs. Mention clock skew (use Redis server time via `TIME` if paranoid), hot keys hammering one Redis slot, and memory growth from unbounded keys.

**5. Cover the contract.** `429 Too Many Requests`, `Retry-After`, and the `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` headers. Rate limits your clients can't see will generate support tickets.

**6. Say where it runs.** API gateway (Spring Cloud Gateway's `RequestRateLimiter`) for coarse global limits, in-service filter for per-user or per-endpoint rules. Both, usually.

## Production checklist

- Enforce at the gateway for global abuse protection and in the service for business rules.
- Return `429` with `Retry-After` and the `X-RateLimit-*` headers on every rejection.
- Emit a metric on rejections (`http.server.requests` tagged by outcome, or a dedicated counter) and alert on spikes — a sudden wall of 429s is either an attack or a misconfigured client.
- Size Redis for the key count: sliding-window logs store one entry per request inside the window. At high volume, consider the sliding-window counter approximation instead.
- Bound in-memory key sets with eviction; never let attacker-controlled keys grow a map forever.
- Document the limits for your API consumers. The kindest rate limiter is one nobody is surprised by.
