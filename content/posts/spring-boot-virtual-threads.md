---
title: 'Flipping spring.threads.virtual.enabled=true? Read This First'
date: 2026-09-20T10:00:00-04:00
draft: false
ShowToc: true
description: >-
  One property flips Spring Boot to virtual threads per request — but it doesn't
  make queries faster, remove the need for backpressure, or fix ThreadLocal
  caches. What the flag actually changes, how to verify it, and a production
  checklist before you enable it.
tags:
  - java
  - spring-boot
  - virtual-threads
  - interview
categories: article
keywords:
  - java
  - spring boot
  - virtual threads
  - project loom
---

Virtual threads can be one of the easiest throughput wins in a blocking Spring Boot application. The switch is one line:

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

But that line changes the concurrency model of the application. It does not make database queries faster, reduce API latency, or create more database connections. It makes waiting cheaper by allowing many request tasks to share a smaller number of carrier threads.

That is excellent for I/O-heavy services. It also means an old Tomcat worker-thread limit may no longer provide the backpressure you thought it did.

This guide builds a small endpoint, verifies that requests really use virtual threads, adds an explicit downstream concurrency limit, and finishes with a production checklist.

## What virtual threads improve

A platform thread normally occupies an operating-system thread while it exists. A virtual thread is scheduled by the JVM and can unmount from its carrier while waiting on supported blocking I/O. The carrier can then run another virtual thread.

The important distinction is:

- **Virtual threads improve scale, not speed.** One database query does not complete faster merely because it runs on a virtual thread.
- **They fit blocking, I/O-heavy code.** Spring MVC, JDBC, JPA, blocking HTTP calls, and file or socket I/O are natural candidates.
- **They do not accelerate CPU-heavy work.** More runnable threads cannot create more CPU cores.
- **They should not be pooled.** The intended model is one new virtual thread per task.

OpenJDK describes virtual threads as a way to raise throughput while preserving the familiar thread-per-request programming model. They are not "faster threads."

## Prerequisites

Use:

- Java 21 or newer
- Spring Boot with virtual-thread support (`spring.threads.virtual.enabled` was introduced in the Spring Boot 3.2 generation)
- Spring MVC with an embedded servlet container for this example

The sample needs Spring Web and, optionally, Actuator:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Then enable virtual threads:

```yaml
spring:
  threads:
    virtual:
      enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,metrics
```

For a normal web application, the embedded server keeps the JVM alive. For a non-web application that relies only on virtual-thread scheduled work, remember that virtual threads are daemon threads. Review your application's lifecycle and keep-alive behavior before deploying.

---

## Prove the switch is active

Do not trust configuration alone. Expose a temporary diagnostic endpoint and inspect the thread handling a real HTTP request.

```java
package com.example.loomdemo;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/demo")
class VirtualThreadController {

    @GetMapping("/thread")
    ThreadReport thread() {
        Thread current = Thread.currentThread();
        return new ThreadReport(current.toString(), current.isVirtual());
    }

    @GetMapping("/io")
    ThreadReport simulatedIo(
            @RequestParam(defaultValue = "250") long delayMs
    ) throws InterruptedException {
        if (delayMs < 0 || delayMs > 5_000) {
            throw new IllegalArgumentException("delayMs must be between 0 and 5000");
        }

        Thread.sleep(delayMs);
        Thread current = Thread.currentThread();
        return new ThreadReport(current.toString(), current.isVirtual());
    }

    record ThreadReport(String thread, boolean virtual) {}
}
```

Run the application and call it:

```bash
curl -s http://localhost:8080/demo/thread
```

The response should contain:

```json
{"thread":"VirtualThread[...]","virtual":true}
```

The exact thread text is JVM-specific. The stable assertion is `virtual: true`.

You can also verify it through an integration test that reaches the actual embedded server. `MockMvc` is not suitable for this particular check because the test can execute controller code on the test thread rather than through Tomcat.

```java
package com.example.loomdemo;

import static org.assertj.core.api.Assertions.assertThat;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class VirtualThreadSmokeTest {

    @LocalServerPort
    int port;

    @Test
    void tomcatHandlesTheRequestOnAVirtualThread() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("http://localhost:" + port + "/demo/thread"))
                .GET()
                .build();

        HttpResponse<String> response = HttpClient.newHttpClient().send(
                request,
                HttpResponse.BodyHandlers.ofString()
        );

        assertThat(response.statusCode()).isEqualTo(200);
        assertThat(response.body()).contains("\"virtual\":true");
    }
}
```

Remove the public diagnostic endpoint after validation, or protect it as an internal-only endpoint.

---

## The `server.tomcat.threads.max` trap

A traditional Tomcat connector uses a bounded worker pool. In that model, this property is meaningful:

```yaml
server:
  tomcat:
    threads:
      max: 200
```

With virtual request threads enabled, Tomcat executes tasks using a virtual-thread-per-task executor. There is no reusable request-worker pool with 200 virtual threads waiting inside it. As a result, `server.tomcat.threads.max` is not a cap on request concurrency in this mode.

That difference is easy to miss because the property still looks valid in configuration. The application starts, but that number no longer limits concurrency the way a bounded platform-thread pool used to.

Do not replace it with "a bigger virtual-thread pool." Virtual threads are intentionally not pooled. Put limits around the scarce resource instead:

- database connections
- outbound calls to a vendor API
- Kafka producer or consumer work
- filesystem or object-storage operations
- memory-heavy request processing

Transport settings such as connection limits and accept queues solve a different problem. They do not express how much concurrent work a database or downstream service can safely handle.

## Add explicit backpressure

Assume a payment provider allows only 40 calls from this service at once. A semaphore makes that limit visible and independent of the web-server thread model.

```java
package com.example.loomdemo;

import static java.util.concurrent.TimeUnit.MILLISECONDS;
import static org.springframework.http.HttpStatus.BAD_GATEWAY;
import static org.springframework.http.HttpStatus.SERVICE_UNAVAILABLE;
import static org.springframework.http.HttpStatus.TOO_MANY_REQUESTS;

import java.util.concurrent.Callable;
import java.util.concurrent.Semaphore;

import org.springframework.stereotype.Component;
import org.springframework.web.server.ResponseStatusException;

@Component
class DownstreamGuard {

    private final Semaphore permits = new Semaphore(40);

    <T> T call(Callable<T> work) {
        boolean acquired = false;

        try {
            acquired = permits.tryAcquire(250, MILLISECONDS);
            if (!acquired) {
                throw new ResponseStatusException(
                        TOO_MANY_REQUESTS,
                        "Too many concurrent downstream calls"
                );
            }

            return work.call();
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new ResponseStatusException(
                    SERVICE_UNAVAILABLE,
                    "Request interrupted",
                    exception
            );
        } catch (ResponseStatusException exception) {
            throw exception;
        } catch (Exception exception) {
            throw new ResponseStatusException(
                    BAD_GATEWAY,
                    "Downstream call failed",
                    exception
            );
        } finally {
            if (acquired) {
                permits.release();
            }
        }
    }
}
```

Use the guard at the integration boundary. This endpoint simulates a protected downstream call without introducing another dependency:

```java
package com.example.loomdemo;

import java.util.Map;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
class ProtectedIoController {

    private final DownstreamGuard guard;

    ProtectedIoController(DownstreamGuard guard) {
        this.guard = guard;
    }

    @GetMapping("/demo/protected-io")
    Map<String, Object> protectedIo(
            @RequestParam(defaultValue = "250") long delayMs
    ) {
        return guard.call(() -> {
            Thread.sleep(delayMs);
            return Map.of(
                    "virtual", Thread.currentThread().isVirtual(),
                    "delayMs", delayMs
            );
        });
    }
}
```

A `Semaphore` parks a waiting virtual thread without requiring a dedicated platform thread. Set the permit count from the real capacity of the dependency, not from the number of virtual threads the JVM can create.

For JDBC or JPA, the connection pool is already a concurrency boundary. Do not multiply the pool size merely because the application can create more request threads. Watch connection-acquisition time, pool saturation, database CPU, lock waits, and query latency first.

---

## The pinning advice depends on your JDK

Many virtual-thread guides say to replace every `synchronized` block with `ReentrantLock`. That advice needs a version label.

### Java 21 through Java 23

A virtual thread that blocks while inside a `synchronized` method or block can pin its carrier. Frequent, long-lived pinning can reduce scalability. This pattern deserves attention:

```java
synchronized (lock) {
    return remoteClient.fetch(); // blocking I/O while holding a monitor
}
```

On these JDKs, moving a hot blocking section to a `ReentrantLock`, or redesigning it so the I/O happens outside the critical section, can prevent that pinning.

### Java 24 and newer

JEP 491 changed monitor handling so a virtual thread can unmount while blocked in `synchronized` code. Do not mechanically rewrite every monitor just for virtual-thread compatibility on these JDKs.

The design rule still matters: keep critical sections narrow and avoid slow I/O while holding locks when practical. That reduces contention regardless of thread type.

Some uncommon pinning cases remain, including blocking through certain native or foreign-function call paths. On modern JDKs, use JFR instead of the old `-Djdk.tracePinnedThreads` switch. JEP 491 made that switch unnecessary but kept the `jdk.VirtualThreadPinned` event for the remaining cases.

Start a short recording against a representative workload:

```bash
PID=$(pgrep -f 'java.*app.jar' | head -1)
jcmd "$PID" JFR.start \
  name=virtual-threads \
  settings=profile \
  duration=60s \
  filename=virtual-threads.jfr
```

Open the recording in JDK Mission Control and inspect virtual-thread pinning events, socket waits, allocation pressure, and the code paths consuming CPU.

## Audit `ThreadLocal` usage

Virtual threads support `ThreadLocal`, so most existing libraries continue to work. The risk is scale: data stored once per thread can be multiplied across a much larger number of short-lived threads.

Look especially for code using a thread local as a resource cache:

```java
private static final ThreadLocal<ExpensiveClient> CLIENT =
        ThreadLocal.withInitial(ExpensiveClient::new);
```

That pattern made more sense when a small pool reused the same worker threads. It is a poor fit for one-thread-per-task execution because every task can initialize another expensive object.

Prefer a properly bounded shared client or pool managed by the framework. Request-scoped metadata such as trace IDs can still use supported context propagation, but measure allocation and retained memory under realistic concurrency.

On JDK 21, this diagnostic option prints a stack trace when a virtual thread sets a thread-local value:

```bash
java -Djdk.traceVirtualThreadLocals=true -jar app.jar
```

Use it in a test environment; the output can be noisy.

---

## Measure the right thing

Do not benchmark only a `/hello` endpoint. A no-op handler measures routing and serialization, not the waiting behavior virtual threads are designed to improve.

Start with the simulated I/O endpoint:

```bash
# 2,000 requests, up to 200 in flight
seq 2000 | xargs -P200 -I{} \
  curl -s -o /dev/null -w '%{http_code}\n' \
  'http://localhost:8080/demo/io?delayMs=250' \
  | sort | uniq -c
```

Run the same workload twice:

```yaml
# Run A
spring:
  threads:
    virtual:
      enabled: false
```

```yaml
# Run B
spring:
  threads:
    virtual:
      enabled: true
```

Warm up the JVM before recording results. Keep hardware, heap settings, traffic shape, and downstream capacity identical. Then compare:

- throughput
- p50, p95, and p99 latency
- error and timeout rate
- CPU utilization
- heap usage and allocation rate
- database-pool wait time
- outbound-client pool saturation
- JFR pinning events

The expected outcome is not "every request gets faster." For a sufficiently concurrent, waiting-heavy workload, the service can keep more requests in progress without consuming one operating-system thread per request. If CPU is already saturated, or the database is the bottleneck, virtual threads may produce little improvement and can expose the downstream limit sooner.

## Review custom executors

The global switch configures eligible Spring Boot auto-configured execution paths. It does not magically replace every executor created by application code or a library.

Search the codebase for:

```text
Executors.newFixedThreadPool
Executors.newCachedThreadPool
ThreadPoolTaskExecutor
ThreadPoolTaskScheduler
@Async
CompletableFuture.supplyAsync
parallelStream
```

For each result, answer:

1. Is the work CPU-bound or mostly waiting on I/O?
2. Which executor actually runs it?
3. Is its queue bounded?
4. Was its pool size acting as backpressure?
5. What happens during shutdown and cancellation?

Do not convert CPU-bound executors to virtual threads just for consistency. A bounded platform-thread executor is often the right choice for computational work.

Spring Boot also notes that pooling properties are ignored when its virtual-thread scheduler is active. If an application depends on scheduler pool size to restrict concurrent jobs, introduce an explicit concurrency limit in the job itself.

## Production checklist

Before enabling the flag in production:

- **Confirm the runtime:** use Java 21 or newer and record the exact JDK version in deployment metadata.
- **Verify a real request:** assert `Thread.currentThread().isVirtual()` through the embedded server.
- **Inventory custom executors:** the global property does not prove every task uses the same execution model.
- **Replace accidental backpressure:** do not rely on `server.tomcat.threads.max` to limit virtual request concurrency.
- **Protect dependencies:** size semaphores, bulkheads, and client pools from actual downstream capacity.
- **Review database pressure:** inspect connection-pool waits, query latency, locks, and database CPU.
- **Audit thread locals:** remove per-thread caches of expensive resources and measure memory under load.
- **Apply version-correct pinning advice:** audit `synchronized` blocking paths on Java 21–23; on Java 24+, focus on the remaining JFR events and general lock contention.
- **Load test realistic I/O:** include representative database and HTTP latency, timeouts, and failure behavior.
- **Compare tail latency and errors:** throughput alone can hide overloaded dependencies.
- **Test shutdown:** verify scheduled tasks, async work, and graceful termination.
- **Roll out gradually:** use a canary, observe it, and keep a fast rollback path.

## Final takeaway

`spring.threads.virtual.enabled=true` is simple; operating it safely requires a clear concurrency model.

Use virtual threads when the service handles many concurrent tasks that spend substantial time waiting. Do not expect lower latency from the switch alone. Make downstream limits explicit, verify the actual execution thread, audit thread-local resource caches, and interpret pinning advice according to the JDK version you run.

The best migration is not "turn on virtual threads everywhere." It is "make waiting cheap while keeping scarce resources bounded."

---

## Sources

- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [Spring Boot: Task Execution and Scheduling](https://docs.spring.io/spring-boot/3.4/reference/features/task-execution-and-scheduling.html)
- [Spring Boot `ConditionalOnVirtualThreads` API](https://docs.spring.io/spring-boot/docs/3.2.0-M1/api/org/springframework/boot/autoconfigure/condition/ConditionalOnVirtualThreads.html)
- [Apache Tomcat `VirtualThreadExecutor`](https://github.com/apache/tomcat/blob/main/java/org/apache/tomcat/util/threads/VirtualThreadExecutor.java)
