---
title: 'Structured Logging in Spring Boot 4: JSON Logs Your ELK Stack Will Actually Parse'
date: 2026-10-05T12:45:00-04:00
draft: false
ShowToc: true
description: >-
  It's 3 AM, a payment flow is failing, and you're grepping 40,000 lines of
  timestamp-soup for one orderId. Structured logging turns your logs into
  queryable data. One property switches Spring Boot 4 to ECS JSON, MDC and
  trace IDs can ride along automatically once Micrometer Tracing is configured,
  and you can shape every field without touching logback XML.
tags:
  - spring-boot
  - java
  - backend
  - observability
categories: article
keywords:
  - spring boot structured logging
  - logging.structured.format.console ecs
  - spring boot 4 json logs
  - mdc traceid logging
---

It's 3 AM. A payment flow is failing for one customer out of thousands, and you're grepping 40,000 lines of timestamp-soup for a single `orderId` — a value buried inside a formatted message, different on every line because someone wrote `"order " + id + " failed"` in one place and `"failed order=" + id` in another. This post kills that workflow. Structured logging turns every log line into a JSON document with named, typed fields your ELK stack can filter, aggregate, and alert on. Spring Boot has supported it natively since 3.4, and on Boot 4 it's a one-property switch — no logstash-logback-encoder, no XML wrestling, no code changes.

## The one property

```yaml
logging:
  structured:
    format:
      console: ecs
```

That's the whole migration for the default Logback setup. Boot supports three JSON formats out of the box: `ecs` (Elastic Common Schema), `gelf` (Graylog), and `logstash` (Logstash JSON). A log line goes from this:

```
2026-10-05T12:40:01.123 INFO  [http-nio-8080-exec-3] c.e.payments.OrderService : Order placed orderId=ORD-9918 amount=149.99
```

to this:

```json
{"@timestamp":"2026-10-05T12:40:01.123Z","log":{"level":"INFO","logger":"com.example.payments.OrderService"},"process":{"pid":39599,"thread":{"name":"http-nio-8080-exec-3"}},"service":{"name":"payments-api"},"message":"Order placed orderId=ORD-9918 amount=149.99","ecs":{"version":"8.11"}}
```

Same line, now a document. Kibana can filter `service.name:payments-api AND log.level:ERROR` without a single regex.

One thing to notice: `orderId` and `amount` are still just message *text* here — ECS wraps the log event, it doesn't parse fields out of the message. Structured fields only appear when they're supplied as MDC or SLF4J key-value data, which is exactly what the next two sections set up. Don't read this example as "ECS extracts my values automatically." The `service.name` comes from `spring.application.name` for free (version defaults to `spring.application.version`); override either under `logging.structured.ecs.service`.

## MDC rides along for free

The real win isn't the JSON wrapper — it's that every key-value pair in the SLF4J MDC is added to the JSON object automatically. One filter puts a request ID on every line of every request:

```java
@Component
public class RequestIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String requestId = Optional.ofNullable(request.getHeader("X-Request-Id"))
                .filter(s -> !s.isBlank())
                .orElseGet(() -> UUID.randomUUID().toString());
        MDC.put("requestId", requestId);
        response.setHeader("X-Request-Id", requestId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("requestId"); // remove only the key this filter added —
                                     // clear() would also wipe traceId/spanId and
                                     // any context other filters established
        }
    }
}
```

The `finally` block is the part people forget, and it's the part that matters: Tomcat recycles threads, so the `requestId` you set would leak into the next request on the same thread unless you remove it at the same boundary where you set it. Note the deliberate `MDC.remove("requestId")` instead of `MDC.clear()` — this filter can run alongside other MDC context, including Micrometer Tracing's `traceId`/`spanId`, and clearing the whole map would erase those too.

Two production caveats before you copy this. First, `X-Request-Id` is untrusted input: this example reflects the header value into the MDC and back out in the response header. In production, validate its length and character set — or generate a fresh ID when the incoming value doesn't match your expected format — so arbitrary client input can't pollute your logs or grow MDC values without bound. Second, "every log line in the request" means every log line emitted *on the request thread*. MDC is thread-local, so `@Async` methods, executor tasks, and reactive pipelines won't see `requestId` unless you propagate the context (e.g. a `TaskDecorator` that copies the MDC map).

What belongs in MDC: request-scoped identifiers — `requestId`, `userId`, `tenantId`. What doesn't: full domain objects, values that change on every log line (raw timestamps, per-event UUIDs), or anything sensitive (more on that below).

## Per-event fields: stop burying data in messages

When values live only inside the message string — string concatenation like `"order " + id + " failed"`, or inconsistent formats across call sites — they are hard to query. The SLF4J fluent API (SLF4J 2, which Boot 4 ships) puts them *next to* the message as named fields instead:

```java
log.atInfo()
   .addKeyValue("event", "order-placed")
   .addKeyValue("orderId", order.id())
   .addKeyValue("amount", order.amount())
   .addKeyValue("backordered", false)
   .log("Order placed");
```

That yields `"event":"order-placed", "orderId":"ORD-9918", "amount":149.99, "backordered":false` as top-level JSON fields — with native types, not strings. `amount > 100` now works as a numeric filter instead of a parsing exercise.

Two rules of thumb:

- Give events stable names, as the example does with `event="order-placed"`. The field survives message rewording, so dashboards don't break when someone edits the prose.
- Keep fields to durable facts about the event — business IDs, status values, attempt counts. Never serialize an entire domain object; it bloats entries and copies unrelated data into long-term storage.

## Shaping the JSON without XML

Boot lets you tune the output with properties — no custom encoder needed:

```yaml
logging:
  structured:
    format:
      console: ecs
    json:
      exclude: log.level            # drop fields you don't need
      rename:
        process.id: procid          # rename to match your ingestion schema —
                                   # note this departs from ECS naming, so do it
                                   # only when the downstream schema requires it
      add:
        environment: ${APP_ENV:local}  # add fixed fields
    ecs:
      service:
        name: payments-api
        environment: production
```

Stack traces deserve their own tuning — a full trace on every error can be expensive for your ingestion pipeline:

```yaml
logging:
  structured:
    json:
      stacktrace:
        max-length: 4096
        include-common-frames: false
        include-hashes: true   # hash lets you group identical crashes
```

For deeper surgery, implement `StructuredLoggingJsonMembersCustomizer` and point `logging.structured.json.customizer` at it.

## Your own format, when the org has a schema

If your platform team mandates a specific log schema, implement `StructuredLogFormatter` — note it doesn't even have to return JSON:

```java
import ch.qos.logback.classic.spi.ILoggingEvent;
import org.springframework.boot.logging.structured.StructuredLogFormatter;

public class OrgLogFormat implements StructuredLogFormatter<ILoggingEvent> {

    @Override
    public String format(ILoggingEvent event) {
        return "time=" + event.getInstant()
                + " level=" + event.getLevel()
                + " message=" + event.getMessage() + "\n";
    }
}
```

```yaml
logging:
  structured:
    format:
      console: com.example.logging.OrgLogFormat
```

The generic type ties the implementation to a logging system (`ILoggingEvent` for Logback, `LogEvent` for Log4j2) — that's the one coupling to be aware of. Supported constructor parameters are injected automatically — things like `Environment`, `StructuredLoggingJsonMembersCustomizer`, `StackTracePrinter`, or `ContextPairs` — but this isn't general Spring bean injection; the API documents a fixed set of supported parameter types.

One caveat: if you use a custom `logback-spring.xml`, Boot is no longer in charge of the encoder, so the `logging.structured.format.*` properties no longer control it — that's expected behavior for a custom configuration, not a silent failure. Update the encoder to respect the structured-format properties:

```xml
<encoder class="org.springframework.boot.logging.logback.StructuredLogEncoder">
    <format>${CONSOLE_LOG_STRUCTURED_FORMAT}</format>
    <charset>${CONSOLE_LOG_CHARSET}</charset>
</encoder>
```

## Trace correlation and Boot 4's OTel story

Structured logs get dramatically more useful when every line carries `traceId`/`spanId`. On Boot 4, the recommended OTel/OTLP path is the official starter — one dependency, version managed by the Boot BOM:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
```

This wires the OTel SDK, the Micrometer Tracing bridge, and the OTLP exporters. Trace context then lands in the MDC automatically, so `traceId`/`spanId` flow into your JSON with zero code. (On the Brave/Zipkin path instead: Boot 4 split `spring-boot-micrometer-tracing` into dedicated `brave` and `opentelemetry` modules — pull the one matching your tracer.)

Two property renames to know when wiring OTLP export:

- tracing export: `management.tracing.export.{name}.enabled`
- logging export: `management.logging.export.{name}.enabled`
- OTLP log-export tuning: `management.opentelemetry.logging.export.otlp.*`

Concretely, on the 3.x → 4.x path: `management.otlp.tracing.export.enabled` became `management.tracing.export.otlp.enabled`, and `management.opentelemetry.logging.export.endpoint` became `management.opentelemetry.logging.export.otlp.endpoint`. If you're migrating a 3.x app that shipped logs to an OTel collector, those renames are the lines to grep for.

One conceptual distinction worth keeping straight: `logging.structured.format.console: ecs` controls the *rendered* log output, while `management.logging.export.otlp.*` configures *OTLP log export* — the exporter sends OpenTelemetry's log data model, not the ECS JSON string. Console/file JSON and OTLP export are separate paths.

## What never goes in the logs

Structured fields make data *easier* to query — including by people who shouldn't see it. Never log `Authorization`/`Cookie` headers, tokens, API keys, passwords, full PII, or card data. If you need a correlatable identifier, mask it (`email=a***@example.com`) or use a keyed hash/HMAC — a plain unsalted hash of a low-entropy value like an email address can be dictionary-attacked, so hashing alone isn't anonymization. Add a CI grep gate that fails the build on `log.*(password|secret|token|apiKey)` as an additional defense-in-depth check — a regex over source can't reliably catch dynamically constructed values, so treat it as a backstop, not a guarantee.

## Prove it: test the shape

Your log schema is a contract with your observability stack. Test it like one — with a real Logback event, not a mock, so the test compiles and runs as shown:

```java
import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.Logger;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.classic.spi.LoggingEvent;
import org.junit.jupiter.api.Test;
import org.slf4j.LoggerFactory;

import static org.junit.jupiter.api.Assertions.*;

class OrgLogFormatTest {

    private final OrgLogFormat format = new OrgLogFormat();

    private static ILoggingEvent event(Level level, String message) {
        Logger logger = (Logger) LoggerFactory.getLogger("test");
        return new LoggingEvent(
                OrgLogFormatTest.class.getName(), logger, level, message, null, null);
    }

    @Test
    void includesLevelAndMessage() {
        String out = format.format(event(Level.INFO, "Order placed"));
        assertTrue(out.contains("level=INFO"));
        assertTrue(out.contains("message=Order placed"));
        assertTrue(out.endsWith("\n"));
    }

    @Test
    void specialCharactersExposeEscapingLimits() {
        // A message containing '=' is ambiguous in this naive format:
        // "message=note=a=b" has no defined field boundaries. This test
        // documents the wire format as-is — a production schema needs real
        // escaping or structured serialization (which the ECS/JSON formats
        // give you for free).
        String out = format.format(event(Level.INFO, "note=a=b"));
        assertTrue(out.contains("message=note=a=b"));
    }
}
```

## The 2-minute interview answer

"Text logs are write-only. Structured logging emits each event as a JSON document with named, typed fields — ECS, GELF, or Logstash out of the box in Spring Boot via `logging.structured.format.console`. Boot's structured formats include MDC key/value pairs automatically, per-event fields come from the SLF4J fluent API's `addKeyValue`, and the schema is tunable with `logging.structured.json.*` properties or a custom `StructuredLogFormatter`. With Micrometer Tracing configured, trace and span IDs land in the MDC, so a log line joins directly to its trace. And I never log secrets or PII — structure makes data easier to query, which cuts both ways."

## Cheat sheet

| Need | How |
|---|---|
| JSON console output | `logging.structured.format.console: ecs` (`ecs`/`gelf`/`logstash`) |
| JSON file output | `logging.structured.format.file: ecs` |
| Request ID on every line | MDC in a filter, remove the filter-owned key in `finally` |
| Per-event fields, native types | `log.atInfo().addKeyValue("k", v).log("...")` |
| Rename/drop/add fields | `logging.structured.json.rename/exclude/add` |
| Trim stack traces | `logging.structured.json.stacktrace.max-length`, `include-hashes` |
| Custom schema | `StructuredLogFormatter<ILoggingEvent>`, FQCN in `format.console` |
| Custom logback.xml | `StructuredLogEncoder` + `${CONSOLE_LOG_STRUCTURED_FORMAT}` |
| Trace correlation | Micrometer Tracing bridge → traceId/spanId in MDC |
| OTLP log export (Boot 4) | `management.logging.export.otlp.enabled`, tuning under `management.opentelemetry.logging.export.otlp.*` |
| Secrets in logs | Never — hash or mask, plus a CI grep gate |

## Takeaway

Structured logging isn't a logging upgrade — it's an observability upgrade. One property gives you JSON, you put request context into the MDC and Boot carries those fields into the structured output, `addKeyValue` gives you typed per-event fields, and the `logging.structured.*` properties give you schema control without XML. The next time a payment fails at 3 AM, you'll filter `requestId` in Kibana instead of grepping 40,000 lines — and go back to sleep.
