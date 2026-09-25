---
title: 'Exactly-Once POSTs: Idempotency Keys in Spring Boot (Stripe-Style)'
date: 2026-09-25T09:20:00-04:00
draft: false
ShowToc: true
description: >-
  Retried POSTs double-charge customers. Stripe solved it with the
  Idempotency-Key header — build the same thing in Spring Boot: DB-backed key
  store, request fingerprinting, race-safe claiming, response replay, and TTL
  cleanup, with tests.
tags:
  - spring-boot
  - java
  - backend
  - api
  - interview
categories: article
keywords:
  - idempotency key spring boot
  - exactly once POST api
  - stripe idempotency pattern
  - duplicate request handling java
---

The client times out waiting for your `POST /orders`. It retries. Your server processes it twice. The customer gets charged twice, and your support queue gets a new entry titled in all caps.

Stripe solved this a decade ago with one header: `Idempotency-Key`. The client generates a UUID, sends it with the request, and retries with the *same* key — Stripe guarantees one effect per key. It's the industry reference implementation ([docs](https://docs.stripe.com/api/idempotent_requests), [engineering blog](https://stripe.com/blog/idempotency)), and there's no reason your API can't offer the same contract. Let's build it.

## The contract (Stripe-style)

| Rule | Behavior |
|---|---|
| Header | `Idempotency-Key: <uuid-v4>` on mutating requests |
| Same key + same params, retried | Return the **original response** — status and body — without re-executing |
| Same key + different params | Error — the key is bound to the request it first arrived with (Stripe: 400, IETF draft: 422) |
| Same key while first request still in flight | `409 Conflict` — "another request is using this key, retry later" |
| Retention | 24 hours; after that the key is treated as new |

Two details people miss: Stripe replays responses **including errors** (a retried 500 returns the stored 500, it doesn't re-run the charge), and the "have I seen this key" check must be arbitrated by the database — a check-then-act in application code double-charges under concurrency. The unique index is the lock.

## Design

Three ingredients, and dropping any one ships a broken implementation:

1. **Key persistence + response cache** — the `(key → response)` mapping, so retries replay instead of re-executing.
2. **Fingerprint check** — SHA-256 of method + path + query + body, so a buggy client reusing a key for a *different* request gets an error instead of the wrong cached response.
3. **In-progress claim** — the key row is inserted atomically in `IN_PROGRESS` state; a concurrent retry sees the claim and gets 409 instead of running the work twice.

State machine: `IN_PROGRESS → COMPLETED` (or `FAILED` on 5xx — still replayed, just labeled for observability).

```sql
-- V1__idempotency_keys.sql (Flyway)
CREATE TABLE idempotency_keys (
    idempotency_key VARCHAR(255) PRIMARY KEY,  -- client-supplied UUID
    endpoint        VARCHAR(512) NOT NULL,     -- POST /api/orders (scopes the key)
    fingerprint     CHAR(64)    NOT NULL,      -- SHA-256(method, path, query, body)
    status          VARCHAR(16) NOT NULL,      -- IN_PROGRESS, COMPLETED, FAILED
    response_status INT,
    response_body   TEXT,
    created_at      TIMESTAMPTZ NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_idem_expiry ON idempotency_keys (expires_at);
```

```java
@Entity
@Table(name = "idempotency_keys")
public class IdempotencyRecord {
    @Id
    private String idempotencyKey;
    private String endpoint;
    private String fingerprint;
    @Enumerated(EnumType.STRING)
    private IdempotencyStatus status;   // IN_PROGRESS, COMPLETED, FAILED
    private Integer responseStatus;
    @Lob
    private String responseBody;
    private Instant createdAt;
    private Instant expiresAt;
    // getters/setters omitted
}
```

## The filter

A `OncePerRequestFilter` wrapping the whole API. The flow:

```
POST /api/orders  +  Idempotency-Key: 550e8400-...
        │
        ▼
fingerprint = sha256("POST /api/orders " + body)
        │
        ▼
try INSERT (key, endpoint, fingerprint, IN_PROGRESS)
        │
   ┌────┴─────────────────────────────────┐
   │ won the race                         │ lost (key exists)
   ▼                                      ▼
execute request,                ┌─ IN_PROGRESS ──────────→ 409 retry later
capture status+body,            ├─ done + fingerprint ok ─→ replay stored response
store as COMPLETED/FAILED       └─ done + fingerprint differs → 422 wrong params
```

```java
@Component
public class IdempotencyFilter extends OncePerRequestFilter {

    private static final String HEADER = "Idempotency-Key";
    private static final Duration TTL = Duration.ofHours(24);
    private static final Set<String> MUTATING =
            Set.of("POST", "PUT", "PATCH", "DELETE");

    private final IdempotencyService idempotency;

    public IdempotencyFilter(IdempotencyService idempotency) {
        this.idempotency = idempotency;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        if (!MUTATING.contains(req.getMethod()) || isExcluded(req)) {
            chain.doFilter(req, res);
            return;
        }

        String key = req.getHeader(HEADER);
        if (key == null || key.isBlank() || key.length() > 255) {
            writeError(res, 400, "Idempotency-Key header (uuid v4) is required");
            return;
        }

        // cache the body so we can read it twice: once for the fingerprint, once downstream
        var cachedReq = new ContentCachingRequestWrapper(req);
        String fingerprint = sha256(req.getMethod() + " " + req.getRequestURI()
                + "?" + req.getQueryString() + "\n" + bodyOf(cachedReq));
        String endpoint = req.getMethod() + " " + req.getRequestURI();

        var existing = idempotency.tryClaim(key, endpoint, fingerprint);
        if (existing.isPresent()) {
            handleReplay(existing.get(), fingerprint, res);
            return;
        }

        var cachedRes = new ContentCachingResponseWrapper(res);
        try {
            chain.doFilter(cachedReq, cachedRes);
        } catch (Exception e) {
            idempotency.storeResult(key, 500, "{\"error\":\"internal error\"}",
                    IdempotencyStatus.FAILED);
            throw e;
        }
        int status = cachedRes.getStatus();
        idempotency.storeResult(key, status, bodyOf(cachedRes),
                status < 500 ? IdempotencyStatus.COMPLETED : IdempotencyStatus.FAILED);
        cachedRes.copyBodyToResponse();
    }

    private void handleReplay(IdempotencyRecord rec, String fingerprint,
                              HttpServletResponse res) throws IOException {
        if (rec.getStatus() == IdempotencyStatus.IN_PROGRESS) {
            writeError(res, 409, "a request with this Idempotency-Key is already in progress");
        } else if (!rec.getFingerprint().equals(fingerprint)) {
            writeError(res, 422, "Idempotency-Key was already used with different parameters");
        } else {
            res.setStatus(rec.getResponseStatus());
            res.setContentType("application/json");
            res.getWriter().write(rec.getResponseBody());
        }
    }
    // sha256(), bodyOf(), writeError(), isExcluded() omitted for brevity
}
```

The claim is the heart of the race safety — one insert, arbitrated by the primary key:

```java
@Service
public class IdempotencyService {

    private final IdempotencyRepository repo;

    /** @return empty if this call won the claim, otherwise the existing record */
    @Transactional
    public Optional<IdempotencyRecord> tryClaim(String key, String endpoint, String fingerprint) {
        var rec = new IdempotencyRecord();
        rec.setIdempotencyKey(key);
        rec.setEndpoint(endpoint);
        rec.setFingerprint(fingerprint);
        rec.setStatus(IdempotencyStatus.IN_PROGRESS);
        rec.setCreatedAt(Instant.now());
        rec.setExpiresAt(Instant.now().plus(TTL));
        try {
            repo.saveAndFlush(rec);          // PK collision ⇒ we lost the race
            return Optional.empty();
        } catch (DataIntegrityViolationException e) {
            return repo.findById(key);       // winner's record (or in-progress row)
        }
    }

    @Transactional
    public void storeResult(String key, int status, String body, IdempotencyStatus s) {
        repo.findById(key).ifPresent(r -> {
            r.setResponseStatus(status);
            r.setResponseBody(body.length() > 1_000_000 ? body.substring(0, 1_000_000) : body);
            r.setStatus(s);
        });
    }

    @Scheduled(cron = "0 0 * * * *")   // hourly purge; keys live 24h like Stripe
    @Transactional
    public void purgeExpired() {
        repo.deleteByExpiresAtBefore(Instant.now());
    }
}
```

Note the expired-key path falls out naturally: once the purge deletes a row, the next request with that key finds nothing and claims it fresh — exactly Stripe's "after 24h, the same key is treated as new."

## Tests

```java
@SpringBootTest
@AutoConfigureMockMvc
class IdempotencyFilterTest {

    @Autowired MockMvc mvc;
    @MockitoBean OrderService orders;   // the thing we must not run twice

    @Test
    void retryReplaysWithoutReexecuting() throws Exception {
        when(orders.place(any())).thenReturn(new Order("ord-1"));
        String key = UUID.randomUUID().toString();
        String body = "{\"sku\":\"kb-01\",\"qty\":1}";

        var first = mvc.perform(post("/api/orders")
                        .header("Idempotency-Key", key)
                        .contentType(APPLICATION_JSON).content(body))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();

        var replay = mvc.perform(post("/api/orders")
                        .header("Idempotency-Key", key)
                        .contentType(APPLICATION_JSON).content(body))
                .andExpect(status().isCreated())
                .andReturn().getResponse().getContentAsString();

        assertEquals(first, replay);
        verify(orders, times(1)).place(any());   // executed exactly once
    }

    @Test
    void sameKeyDifferentBodyIsRejected() throws Exception {
        String key = UUID.randomUUID().toString();
        mvc.perform(post("/api/orders").header("Idempotency-Key", key)
                        .contentType(APPLICATION_JSON).content("{\"sku\":\"a\"}"))
                .andExpect(status().isCreated());
        mvc.perform(post("/api/orders").header("Idempotency-Key", key)
                        .contentType(APPLICATION_JSON).content("{\"sku\":\"b\"}"))
                .andExpect(status().isUnprocessableEntity());
    }

    @Test
    void missingKeyIsRejected() throws Exception {
        mvc.perform(post("/api/orders")
                        .contentType(APPLICATION_JSON).content("{}"))
                .andExpect(status().isBadRequest());
    }

    @Test
    void concurrentRetries_onlyOneExecutes() throws Exception {
        var svc = new IdempotencyService(repo);   // real repo, H2/Testcontainers
        String key = UUID.randomUUID().toString();
        var barrier = new CyclicBarrier(2);
        var wins = new AtomicInteger();
        Runnable claim = () -> {
            await(barrier);
            if (svc.tryClaim(key, "POST /api/orders", "fp").isEmpty()) wins.incrementAndGet();
        };
        var t1 = Thread.ofVirtual().start(claim);   // virtual threads: cheap concurrency
        var t2 = Thread.ofVirtual().start(claim);
        t1.join(); t2.join();
        assertEquals(1, wins.get());   // the PK arbitrates: exactly one winner
    }
}
```

`retryReplaysWithoutReexecuting` is the test that matters — it's the double-charge scenario, automated.

## Gotchas worth knowing

- **Scope keys per endpoint (and per tenant).** Stripe scopes keys per API key; at minimum include the endpoint in the record so `POST /orders` and `POST /refunds` can never share a key's fate.
- **Fingerprint the full request.** Method + path + query string + body. A client that reuses a key across different payloads is buggy — the fingerprint turns that bug into a loud 422 instead of a silent wrong replay.
- **Generate keys client-side.** UUID v4 per user action (button click), not per HTTP attempt — the retry must carry the *same* key. Stripe's engineering blog goes further: derive keys from the business operation (`checkout:{cartId}:{attempt}`) so even a client restart retries safely.
- **Don't store PII-adjacent bodies blindly.** The response cache is a database table — cap body size (1 MB above), and think before caching endpoints that return tokens or PII.
- **Replay errors too.** If the first attempt 500'd, the retry gets the stored 500 — it must not re-execute the charge. This is the Stripe behavior most homegrown implementations get wrong.
- **Only for mutating methods.** GET/HEAD/OPTIONS are idempotent by HTTP semantics; the filter passes them through untouched.
- **24h is a policy, not physics.** Stripe picked 24h as long-enough-for-retries and short-enough-to-bound-storage. Pick yours deliberately and publish it — the IETF draft says the server SHOULD document its expiration.

## The interview one-liner

> "I'd accept an `Idempotency-Key` header on mutating endpoints, fingerprint the request, and insert a claim row keyed by the idempotency key — the primary key arbitrates concurrent retries, so exactly one request executes. Completed results (status + body, including errors) are stored for 24h and replayed; a concurrent retry gets 409, a reused key with different params gets 422, and an hourly job purges expired keys."

Follow-ups you'll get: "what if the process dies mid-request?" (the row stays `IN_PROGRESS` until the 24h TTL purges it — a retry then executes fresh; for stricter recovery you'd add a heartbeat/lease), "why not Redis?" (fine as a cache in front, but the database constraint is the source of truth — a cache eviction must never resurrect a charge), and "where does the key come from?" (client-generated UUID v4, one per user action, never per attempt).
