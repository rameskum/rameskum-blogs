---
title: 'Declarative HTTP Clients in Spring Boot 4: Moving Beyond RestTemplate'
date: 2026-10-01T13:00:00-04:00
draft: false
ShowToc: true
description: >-
  Spring Framework 7 deprecates RestTemplate in favor of RestClient. Spring
  Boot 4 adds a second modernization path: declarative HTTP clients —
  @HttpExchange interfaces registered with @ImportHttpServices, configured
  through spring.http.serviceclient properties, speaking RFC 9457 ProblemDetail
  for errors, with retry from Framework 7's built-in @Retryable and no
  spring-retry dependency.
tags:
  - spring-boot
  - java
  - backend
  - microservices
  - interview
categories: article
---

Every Spring codebase has one: the `PaymentsClient` class. Sixty lines of `restTemplate.exchange()` with string-concatenated URLs, an `HttpEntity` built by hand for every POST, a try/catch that maps `HttpStatusCodeException` to something the caller understands, and a custom `ResponseErrorHandler` someone wrote in 2021 that nobody dares to touch.

It works. But the platform is moving on, and there are now two modernization paths worth knowing. The first is the modern imperative successor: Spring Framework 7 deprecates `RestTemplate` in favor of `RestClient` — new imperative clients should generally prefer `RestClient`, while existing working `RestTemplate` code doesn't require a flag-day rewrite. The second is the declarative HTTP client: an interface, some annotations, and no handwritten implementation class. It is not a replacement for `RestClient` — it is an interface/proxy abstraction that sits on top of an underlying HTTP client:

```text
HTTP clients
  ├── RestClient
  ├── WebClient
  └── RestTemplate

HTTP Service Client
  └── @HttpExchange interface
        └── proxy backed by one of the HTTP clients above
```

And `clientType` selects the *Spring client API* used by the proxy — not the network transport underneath it:

```text
HTTP Service Client
        |
        +-- REST_CLIENT --> RestClient --> JDK / Apache / Jetty / Reactor Netty
        |
        +-- WEB_CLIENT  --> WebClient  --> Reactor Netty / Jetty / Apache / JDK
```

The underlying transport implementation is a separate Boot configuration concern (Boot auto-detects and configures the available HTTP client libraries for imperative and reactive clients independently). Conflating "the proxy uses `RestClient`" with "requests go over the JDK client" is a common source of confusion — keep the two layers distinct.

This guide covers the second path: when the declarative client earns its place, how Boot 4 wires it with almost no code, and the testing and retry details that decide whether it survives production. Spring Framework supports HTTP Service Clients with multiple underlying HTTP client adapters; Spring Boot 4 adds grouped registration and configuration for managing these clients as application infrastructure.

> The examples follow the Spring Boot 4.0 / Framework 7.0 APIs as documented. If you're on a later 4.x line (e.g. 4.1.x), verify the group-configuration and test-support details against your pinned version — the shapes are stable, but minor behavior can drift between minors.

## The interface

This is the whole client:

```java
@HttpExchange(accept = "application/json")
public interface PaymentsClient {

    @GetExchange("/charges/{id}")
    Charge getCharge(@PathVariable String id);

    @GetExchange("/charges")
    List<Charge> listCharges(@RequestParam("status") String status);

    @PostExchange("/charges")
    Charge createCharge(@RequestBody ChargeRequest request,
                        @RequestHeader("Idempotency-Key") String idempotencyKey);
}
```

The annotations deliberately mirror Spring MVC — `@GetExchange`/`@PostExchange` declare method and path, and `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader` work the way you expect. Return types can be DTOs, `List<T>`, `ResponseEntity<T>`, or `void`. (`List<Charge>` resolves through normal HTTP message conversion — not a special HTTP Service Client feature.) Treat the resemblance as a learning aid, not an equivalence: HTTP Service Clients are generated proxies with their own rules for return types, error handling, and request adapters, so "a controller in reverse" will eventually surprise you.

Note the interface declares no base URL. That comes from the service group configuration below — one canonical place per upstream, not a placeholder scattered across annotations.

## Zero-wiring registration in Boot 4

In Boot 4, registering the interface as a bean is one annotation — no `HttpServiceProxyFactory` boilerplate:

```java
import org.springframework.web.service.registry.HttpServiceGroup.ClientType;

@Configuration(proxyBeanMethods = false)
@ImportHttpServices(
    group = "payments",
    types = PaymentsClient.class,
    clientType = ClientType.REST_CLIENT
)
class HttpClientConfig {
}
```

Inject `PaymentsClient` anywhere like a normal bean; Spring generates the implementation at startup. The `clientType` attribute declares the backing explicitly — `REST_CLIENT` here. Use `WEB_CLIENT` for a reactive client. Leaving `clientType` unspecified uses the registrar's default, which is `REST_CLIENT` unless that default has been customized — it does not perform reactive/imperative auto-selection, so don't read it as "Boot decides for me". The group configurers then customize the already-selected type: `RestClientHttpServiceGroupConfigurer` customizes `REST_CLIENT` groups, `WebClientHttpServiceGroupConfigurer` customizes `WEB_CLIENT` groups. So when the backing matters, say so in the annotation rather than inferring it from whether your app is reactive.

Base URL, timeouts, and headers come from properties, grouped per client:

```yaml
spring:
  http:
    serviceclient:
      payments:
        base-url: https://payments.internal
        connect-timeout: 2s
        read-timeout: 5s
```

Don't assume an application-level timeout unless you explicitly configure one. These timeout properties may be left unset; when they are, the effective timeout behavior depends on the underlying HTTP client implementation. Size timeouts from your end-to-end request budget. A dependency's p99 latency is useful input for sizing a timeout, but it is not itself a timeout recommendation — your timeout must also account for connection establishment, TLS, retries, backoff, and the caller's overall deadline. A five-second per-attempt read timeout does not guarantee completion within a two-second caller budget. The actual attempts may complete sooner, but the timeout configuration still permits a single attempt to consume more than the caller's entire budget:

```text
2s caller budget
├── attempt 1
├── backoff
└── attempt 2
```

Size the timeout so that this whole tree fits inside the deadline. Don't size the deadline from the observed attempt duration alone: observed latency is not an upper bound on future calls, while the configured timeout is an upper bound the client may actually wait. The problem isn't choosing a timeout in isolation; it's choosing one that violates the caller's end-to-end deadline once retries and backoff are included.

## Customize without the factory bean

The manual `HttpServiceProxyFactory` bean is not the only escape hatch. Boot 4 has HTTP Service group configurers specifically for customizing grouped declarative clients — headers, error handling, and interceptors without dropping all the way to manual proxy creation:

```java
@Bean
RestClientHttpServiceGroupConfigurer paymentsConfigurer(PaymentsErrorHandler errorHandler) {
    return groups -> groups.filterByName("payments")
        .forEachClient((group, builder) -> builder
            .defaultHeader("User-Agent", "checkout-service/2.4")
            .defaultStatusHandler(HttpStatusCode::isError, errorHandler::handle)
            .requestInterceptor((request, body, execution) -> {
                try {
                    return execution.execute(request, body);
                } catch (ResourceAccessException ex) {
                    // For a RestClient-backed HTTP Service group: normalize transport failures
                    // once, at the client boundary — the exact exception type depends on the
                    // underlying HTTP client (a WebClient-backed group has a different
                    // reactive exception model and needs its own handling).
                    throw new TransientPaymentsException("payments unreachable", ex);
                }
            }));
}
```

Need full control — for example, a custom `ClientHttpRequestFactory` with connection-pool, proxy, or TLS configuration? The manual factory bean is still fully supported:

```java
// Alternative to @ImportHttpServices — use one registration strategy for a
// given client interface, not both, or you'll end up with two PaymentsClient
// beans in the same context.
@Bean
PaymentsClient paymentsClient(RestClient.Builder builder, PaymentsErrorHandler errorHandler) {
    RestClient restClient = builder
            // In production, obtain the base URL from application configuration.
            .baseUrl("https://payments.internal")
            .defaultHeader("User-Agent", "checkout-service/2.4")
            .defaultStatusHandler(HttpStatusCode::isError, errorHandler::handle)
            // Deliberately replaces the request factory Boot selected and configured
            // for the injected builder — including any global spring.http.clients.*
            // settings. If you take this escape hatch, configure timeouts, pooling,
            // proxy, and TLS on the resulting factory yourself.
            .requestFactory(new JdkClientHttpRequestFactory())
            .build();
    return HttpServiceProxyFactory.builderFor(RestClientAdapter.create(restClient))
            .build()
            .createClient(PaymentsClient.class);
}
```

Use the property-based registration until you have a reason not to; reach for the group configurer for behavior, and the factory bean when the client needs something neither can express. One caution on the escape hatch: calling `requestFactory(...)` replaces the request factory Boot auto-detected and configured for the injected `RestClient.Builder` — along with any global `spring.http.clients.*` timeout, SSL, and connection settings. A reader copying that line without configuring the replacement factory silently loses Boot's HTTP-client infrastructure, which is exactly the kind of accidental downgrade the migration checklist warns against.

## Handle errors as ProblemDetail

Spring provides first-class support for RFC 9457 Problem Details: error responses commonly use `application/problem+json` and can contain members such as `type`, `title`, `status`, `detail`, and `instance`. (Spring has supported `ProblemDetail` since Framework 6.) Your client should parse that envelope instead of string-matching on exception messages — but key on the right fields. In RFC 9457, `type` identifies the problem type, `status` is the HTTP status, `title` is a human-readable summary, and `detail` describes this specific occurrence. If your upstream contract guarantees stable `type` URIs, they can serve as machine-readable error categories — but stability is a property of the upstream contract, not of Problem Details itself. Treat `type` as a contract-defined identifier: don't build business logic around arbitrary URI formatting unless the upstream API documents it. `title` and `detail` are human-readable fields, not stable identifiers.

Make the mapping a real component so it lives in exactly one place — configuration and tests can reuse it rather than copy it. One Boot 4 note: this uses Jackson 3 (`tools.jackson.*`) — Boot 4 auto-configures a `JsonMapper` bean, and Jackson 2's `com.fasterxml.jackson.databind.ObjectMapper` is deprecated there, so don't reach for the old package in new Boot 4 code:

```java
import tools.jackson.databind.json.JsonMapper;

@Component
class PaymentsErrorHandler {

    private final JsonMapper jsonMapper;

    PaymentsErrorHandler(JsonMapper jsonMapper) {
        this.jsonMapper = jsonMapper;
    }

    void handle(HttpRequest request, ClientHttpResponse response) throws IOException {
        MediaType contentType = response.getHeaders().getContentType();
        if (contentType != null && contentType.isCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON)) {
            // Production clients should impose an appropriate maximum response-body size
            // for untrusted upstreams — readValue here is unbounded. Enforce the limit
            // at the HTTP/client buffering layer so oversized bodies are rejected
            // before they are fully materialized for JSON parsing.
            final ProblemDetail problem;
            try {
                problem = jsonMapper.readValue(response.getBody(), ProblemDetail.class);
            } catch (JacksonException ex) {
                // A malformed problem body is its own signal: the upstream broke its
                // error contract. Keep it distinct from a valid typed problem and from
                // a non-problem error body.
                throw new MalformedPaymentsErrorResponseException(response.getStatusCode(), ex);
            }
            // RFC 9457: type defaults to about:blank when the member is absent
            java.net.URI type = problem.getType() != null ? problem.getType() : java.net.URI.create("about:blank");
            // Use the actual HTTP response status for transport semantics.
            // ProblemDetail.status is informational and may be absent or inconsistent.
            throw new PaymentsException(type, problem.getDetail(), response.getStatusCode());
        }
        throw new PaymentsException(null, "payments call failed",
                response.getStatusCode().value());
    }
}
```

The point: give the caller a typed exception keyed on the upstream's `type`, not a raw `RestClientResponseException` whose message is a JSON blob. When the payments team edits their error text, your logs and alerts keep working because you keyed on the category, not the prose. And the full pipeline is explicit — problem+json that parses cleanly → typed domain exception; problem+json that doesn't parse → a distinct malformed-response exception carrying the parse failure; anything else → generic upstream exception. (`MalformedPaymentsErrorResponseException` is a small type you define yourself — a `RuntimeException` carrying the status code and the Jackson failure — so an upstream contract regression shows up in your alerts as its own category.)

## Retry with Spring Framework 7: no spring-retry

Retry is now built into Spring Framework 7, so this approach does not require the separate `spring-retry` dependency. Enable it once:

```java
@Configuration
@EnableResilientMethods
class ResilienceConfig {
}
```

Then layer `@Retryable` on the calling service method. For this design, put it on the calling service rather than the HTTP interface: retry policy belongs at the operation boundary, where you know whether the call is safe to replay. (Nothing in the framework forbids annotating the interface — this is an architectural choice, not a requirement.)

```java
@Service
class CheckoutService {

    private final PaymentsClient payments;

    CheckoutService(PaymentsClient payments) {
        this.payments = payments;
    }

    @Retryable(includes = TransientPaymentsException.class,
               maxRetries = 2, delay = 200, multiplier = 2.0, maxDelay = 2000, jitter = 50)
    public Charge chargeWithRetry(ChargeRequest request) {
        return payments.createCharge(request);
    }
}
```

Four things to get right, because the attribute names differ from the old spring-retry project:

- **`maxRetries` counts retries after the first call.** `maxRetries = 2` means three total attempts. (The old annotation's `maxAttempts` counted the initial call too.)
- **`includes` is the retry trigger list — name your exception, not the transport's.** Retry only when the failure is plausibly transient **and** the operation is safe to replay under the upstream API's contract — "transient" and "safe to retry" are different properties. A POST that timed out tells you nothing about whether the server committed it. The exception a timeout surfaces as depends on the underlying HTTP client (JDK, Apache, Netty all differ) — `SocketTimeoutException` is not a portable "HTTP timeout" catch-all. Normalize once at the client boundary, as the interceptor above does, and retry your own `TransientPaymentsException`.
- **Don't use the status code alone as the retry policy.** A `400 Bad Request` generally should not be replayed, but `429 Too Many Requests` (back off and retry) or a documented 409-conflict-retry contract can be. For 429s, honor `Retry-After` when the upstream provides it, subject to your overall request deadline and maximum backoff policy — a fixed exponential delay can otherwise conflict with the upstream's rate-limit policy. Base retryability on the operation's semantics and the upstream contract, not the status family.
- **Think before retrying POSTs.** Retrying a charge creation is only safe if the endpoint is idempotent — this is why the interface sends an `Idempotency-Key` header. And the header only helps if the upstream actually implements idempotency semantics for it; sending it alone doesn't make a non-idempotent endpoint safe. Blindly retrying a non-idempotent POST can result in duplicate side effects — for example, charging a customer twice.

Before any retry, run this decision: is the failure plausibly transient? Is the operation replay-safe under the upstream contract? Does the upstream contract permit retry (including `Retry-After` for 429s)? Is there enough remaining request budget? A "no" anywhere means propagate rather than retry — or use an idempotency mechanism where the contract supports one.

And one honest limitation: Framework 7's core `@Retryable` does not provide Spring Retry's `@Recover` mechanism — when attempts are exhausted, the last exception propagates. Catching it at the orchestration boundary and deciding the degraded path there is the common pattern. It isn't the only recovery mechanism: `@Retryable` publishes `MethodRetryEvent` instances for failures during retry processing, so exhausted retries don't have to be the only signal you retain. The programmatic `RetryTemplate` API additionally provides `RetryListener` hooks when you need more control. Like all proxy-based advice, self-invocation bypasses it: the `@Retryable` method must be called through the bean.

### Test retry at the service layer

The `PaymentsClient` test in **Test it** below covers mapping; retry belongs a layer up. Test `CheckoutService` with the client mocked, so the test exercises the retry policy — and so a `MockRestServiceServer` expectation never has to reason about retries:

```java
@SpringBootTest(classes = {ResilienceConfig.class, CheckoutService.class})
class CheckoutServiceRetryTest {

    @Autowired CheckoutService checkout;

    @MockitoBean PaymentsClient payments;

    @Test
    void retriesTransientFailureThenSucceeds() {
        when(payments.createCharge(any()))
            .thenThrow(new TransientPaymentsException("timeout", new IOException("read timed out")))
            .thenReturn(new Charge("ch_123", "succeeded"));

        Charge charge = checkout.chargeWithRetry(new ChargeRequest(100, "usd"));

        assertThat(charge.id()).isEqualTo("ch_123");
        verify(payments, times(2)).createCharge(any());
    }

    @Test
    void propagatesAfterExhaustedRetries() {
        when(payments.createCharge(any()))
            .thenThrow(new TransientPaymentsException("down", new IOException("connection refused")));

        assertThatThrownBy(() -> checkout.chargeWithRetry(new ChargeRequest(100, "usd")))
            .isInstanceOf(TransientPaymentsException.class);
        // 1 initial attempt + 2 retries
        verify(payments, times(3)).createCharge(any());
    }
}
```

This also reinforces the architecture: retry lives on the calling service, where replay safety is known, not on the HTTP interface. (`@MockitoBean` is Spring Boot's Mockito integration; the retry delays make the exhausted-retry test take roughly 600ms of backoff — 200ms then 400ms with the 2.0 multiplier — plus jitter. Size the delays appropriately for your test suite in real code.)

## Test it

Test the declarative interface by building the proxy yourself over a `MockRestServiceServer`-bound `RestClient.Builder` — in-process, no real port, using Spring's mock-server test support:

```java
import tools.jackson.databind.json.JsonMapper;

class PaymentsClientTest {

    MockRestServiceServer server;
    PaymentsClient client;

    @BeforeEach
    void setUp() {
        // Reuse the production error handler so the test exercises production
        // mapping, not a second copy. Use a minimal JsonMapper here because this
        // is a focused unit test, not a Spring wiring test — Boot's configured
        // JsonMapper should be covered separately by an application-context/
        // integration test.
        PaymentsErrorHandler errorHandler =
                new PaymentsErrorHandler(JsonMapper.builder().build());
        RestClient.Builder builder = RestClient.builder()
                .baseUrl("https://payments.internal")
                .defaultStatusHandler(HttpStatusCode::isError, errorHandler::handle);
        server = MockRestServiceServer.bindTo(builder).build();
        client = HttpServiceProxyFactory
                .builderFor(RestClientAdapter.create(builder.build()))
                .build()
                .createClient(PaymentsClient.class);
    }

    @Test
    void fetchesCharge() {
        server.expect(requestTo("https://payments.internal/charges/ch_123"))
              .andRespond(withSuccess(
                  "{\"id\":\"ch_123\",\"status\":\"succeeded\"}",
                  MediaType.APPLICATION_JSON));

        assertThat(client.getCharge("ch_123").status()).isEqualTo("succeeded");
        server.verify();
    }

    @Test
    void serializesChargeRequest() {
        server.expect(requestTo("https://payments.internal/charges"))
              .andExpect(method(HttpMethod.POST))
              .andExpect(content().json("""
                  {"amount":100,"currency":"usd"}
              """))
              .andRespond(withSuccess(
                  "{\"id\":\"ch_123\",\"status\":\"succeeded\"}",
                  MediaType.APPLICATION_JSON));

        Charge charge = client.createCharge(new ChargeRequest(100, "usd"), "key-123");

        assertThat(charge.id()).isEqualTo("ch_123");
        server.verify();
    }

    @Test
    void mapsProblemDetail() {
        server.expect(requestTo("https://payments.internal/charges/missing"))
              .andRespond(withStatus(HttpStatus.NOT_FOUND)
                  .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                  .body("{\"type\":\"https://payments.internal/problems/charge-not-found\"," +
                        "\"title\":\"Charge not found\",\"status\":404}"));

        assertThatThrownBy(() -> client.getCharge("missing"))
              .isInstanceOf(PaymentsException.class)
              .hasMessageContaining("charge-not-found");
        server.verify();
    }
}
```

This verifies URI templates, request serialization, response deserialization, and your error mapping against the real proxy — the four things this focused test exercises particularly well.

One honest caveat: this builds the proxy explicitly. `@RestClientTest` is centered on the `RestClient.Builder` configured by the test slice, so if your HTTP Service proxy is created through Boot's HTTP Service registry, verify that the proxy actually uses the instrumented builder before relying on the auto-configured `MockRestServiceServer`. The concrete way to check: a small `@SpringBootTest` that imports the actual HTTP Service registration is often the simplest way to verify the complete Boot wiring.

A practical progression:

- `MockRestServiceServer` → focused HTTP Service Client test
- `@SpringBootTest` + mock server → Boot wiring test
- WireMock → real HTTP boundary

Escalate to WireMock when you need a real port: real timeouts, TLS, latency/fault injection, or a stub shared across a full `@SpringBootTest`. Integration tests against an actual server also catch URI, serialization, header, and networking behavior that a client-level mock never exercises. A practical rule of thumb: use a mock server for focused HTTP Service Client tests, and a real HTTP test server such as WireMock when you need to exercise the HTTP boundary more realistically.

## Migration checklist

If you're staring at a codebase full of `RestTemplate`:

1. **Inventory first.** Grep for `RestTemplate` construction sites and inspect each underlying request factory for timeout, pooling, TLS, proxy, and error-handler configuration. A `new RestTemplate(customRequestFactory)` can be perfectly well-configured, and a Spring-managed `RestTemplate` can have no explicit timeouts at all — the constructor tells you nothing by itself.
2. **Keep interfaces cohesive.** Recommended design guideline, not a framework rule: one interface per consumer-facing capability (`PaymentsClient`, `FraudClient`), not one forty-method `ExternalApiClient`. Splits like `PaymentsClient` vs `PaymentsAdminClient` against the same upstream are legitimate when the capabilities differ.
3. **Move base URLs to service groups.** The canonical Boot 4 approach is `@ImportHttpServices(group = "payments", types = PaymentsClient.class, clientType = ClientType.REST_CLIENT)` plus `spring.http.serviceclient.payments.base-url` — keep the environment-specific base URL out of the Java interface and centralize it in configuration.
4. **Centralize the common error mapping, allow endpoint-specific handling.** One `defaultStatusHandler` producing one typed exception per client — but endpoint-specific handling is legitimate: a 404 on `GET /charges/{id}` becoming `Optional.empty()` while a 500 on `POST /charges` becomes an exception is a reasonable design, not a violation.
5. **Port the tests honestly.** The declarative interface tests well with `MockRestServiceServer` bound to the `RestClient.Builder` you build the proxy from; for registry-generated clients, verify the proxy uses `@RestClientTest`'s instrumented builder before relying on its auto-configured mock server. WireMock when a real port is needed.
6. **Leave working code alone until you touch it.** `RestTemplate` remains available in Boot 4, although Spring Framework 7 deprecates it in favor of `RestClient`. Migrate clients you're already changing; don't do a flag-day rewrite.

## The interview one-liner

> Spring Framework 7 deprecates `RestTemplate` in favor of `RestClient`, and provides declarative HTTP Service Clients through `@HttpExchange` — while Spring Boot 4 adds grouped registration and configuration so those interfaces can be backed by `RestClient` or `WebClient` without manually creating `HttpServiceProxyFactory`. Normalize errors to RFC 9457 `ProblemDetail` keyed on `type`, retry transient failures with Framework 7's built-in `@Retryable` (`maxRetries` counts retries after the first call; no `@Recover` — after retries are exhausted, the last exception propagates), and test the declarative interface with a `MockRestServiceServer`-bound `RestClient.Builder` — or WireMock when a real port is needed.
