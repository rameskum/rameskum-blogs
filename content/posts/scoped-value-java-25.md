---
title: 'ScopedValue: The ThreadLocal Replacement Java 25 Finalized'
date: 2026-10-23T19:40:00+05:30
draft: false
ShowToc: true
description: >-
  JDK 25 made ScopedValue final. Here is the practical migration from
  ThreadLocal for request context: dynamic-scope bindings, safe defaults,
  structured-concurrency inheritance, the MDC bridge, the memory difference,
  and the cases where ThreadLocal is still the right tool.
tags:
  - java
  - backend
  - concurrency
  - interview
categories: article
keywords:
  - ScopedValue Java 25
  - JEP 506
  - ScopedValue vs ThreadLocal
  - Java scoped values tutorial
  - ThreadLocal replacement
  - virtual threads request context
---

In the [ThreadLocal leak post](https://blogs.rameskum.com/posts/threadlocal-leak-spring-boot/), `ScopedValue` was the third fix — two paragraphs and a filter snippet. This is the full treatment, because JDK 25 changed its status: after an incubator round in JDK 20 and previews in JDK 21–24, [JEP 506](https://openjdk.org/jeps/506) made `java.lang.ScopedValue` final in JDK 25. No preview flag. No incubator module. It is now part of the platform.

First, the honest framing. "Replacement" comes with an asterisk:

- JEP 506 explicitly does **not** deprecate `ThreadLocal`, and migration is **not** a goal for every use.
- `ScopedValue` replaces the dominant request-context pattern: bind an immutable value at an entry point, let callees read it, and guarantee it disappears when that operation ends.
- It does not replace per-thread mutable caches, accumulators that callees update for a caller, or values that must outlive the method that created them.

That distinction is the whole article. Get it right and `ScopedValue` removes a bug class. Treat it as a nicer global variable and you will recreate the `ThreadLocal` mess with newer syntax.

## The mental model: bind, run, disappear

A `ThreadLocal` is a per-thread variable you mutate:

```java
CONTEXT.set(value);
// ... later, somewhere else ...
CONTEXT.remove();
```

A `ScopedValue` is a key whose binding exists only for a dynamic extent — the execution of one `run(...)` or `call(...)` operation and everything it calls, directly or indirectly:

```java
private static final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

// At the boundary:
ScopedValue.where(REQUEST_ID, "req-42")
           .run(() -> handleRequest());

// Anywhere downstream during handleRequest():
String requestId = REQUEST_ID.get();

// Back at the boundary, after run() returns:
// REQUEST_ID.get() throws NoSuchElementException — the binding is gone.
```

Three details carry most of the design:

1. **`where(...)` does not bind anything yet.** It returns an immutable `ScopedValue.Carrier` describing one or more key/value mappings.
2. **`run(...)` or `call(...)` performs the binding.** For the duration of that operation, the current thread sees those mappings. When the operation returns or throws, the previous state is restored: bound values revert, and previously unbound values become unbound again.
3. **There is no `set()` or `remove()`.** Code cannot mutate an existing binding from somewhere deep in the stack. A callee can create a *nested* binding for its own callees, but it cannot change what its caller sees.

"Dynamic scope" is the key phrase. The binding does not follow the text of the lambda; it follows execution. If code inside `run(...)` calls five layers of services, all five can read the value. If it returns a `CompletableFuture` that completes after `run(...)` has exited, the completion callback is outside the extent and cannot rely on the binding still being there.

## The JDK 25 API, without the stale examples

The final API is small enough to memorize. From the [JDK 25 `ScopedValue` documentation](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/ScopedValue.html):

| API | What it does |
|---|---|
| `ScopedValue.newInstance()` | Creates a scoped value that starts unbound for every thread |
| `ScopedValue.where(key, value)` | Creates a `Carrier` containing one mapping; does not bind yet |
| `Carrier.where(key, value)` | Returns a **new** carrier with an additional or replacement mapping |
| `Carrier.run(Runnable)` | Binds the mappings and runs an operation that returns nothing |
| `Carrier.call(CallableOp)` | Binds the mappings and runs an operation that returns a value and may throw |
| `Carrier.get(key)` | Reads a value from the carrier's own mapping — not from the current thread |
| `get()` | Returns the current thread's binding; throws `NoSuchElementException` if unbound |
| `isBound()` | Whether the current thread has a binding |
| `orElse(other)` | Returns the binding, or `other` when unbound |
| `orElseThrow(supplier)` | Returns the binding, or throws the supplied exception when unbound |

Bind several values fluently:

```java
ScopedValue.where(REQUEST_ID, requestId)
           .where(TENANT_ID, tenantId)
           .run(() -> handleRequest());
```

That is not the only style you will find online. Examples from the earlier preview rounds may use `ScopedValue.runWhere(...)` or `ScopedValue.callWhere(...)`. Those methods were removed in JDK 24's fourth preview, leaving the fluent `where(...).run(...)` / `where(...).call(...)` form as the final API. If an example does not name its JDK version, do not trust its `ScopedValue` syntax.

One JDK 25 tightening worth knowing: `orElse(...)` no longer accepts `null` as the fallback argument. Use an explicit sentinel, an `Optional`-returning accessor, or `orElseThrow(...)` when "missing" is an error.

## Rebinding means shadowing, not mutation

A nested `where(...)` temporarily shadows the outer value:

```java
private static final ScopedValue<String> TENANT = ScopedValue.newInstance();

ScopedValue.where(TENANT, "acme").run(() -> {
    System.out.println(TENANT.get()); // acme

    ScopedValue.where(TENANT, "globex").run(() -> {
        System.out.println(TENANT.get()); // globex
    });

    System.out.println(TENANT.get()); // acme again
});
```

This is deliberately different from `ThreadLocal.set()`:

- `bar()` cannot change the value `foo()` will see after `bar()` returns.
- The nested value applies only to the nested operation and its callees.
- When the outermost operation exits, the scoped value is unbound again.

For tenant and principal context, that is a security property as well as a memory property. Downstream code can establish a narrower context for a sub-operation, but it cannot silently rewrite the caller's context in place.

It does **not** make the object you bind immutable. `ScopedValue<List<String>>` still points to whichever `List` implementation you supplied. If that list is mutable and several child threads mutate it, you have a concurrent-mutation problem with better packaging. Bind records and deeply immutable values; defensively copy collections at the boundary.

## Why JEP 506 says ThreadLocal has three design flaws

JEP 506 names the problems directly:

### 1. Unconstrained mutability

Any code with access to a `ThreadLocal` can call `set()` at any time. Data can flow back up the stack, sideways through shared code, or change halfway through a request. With a scoped value, transmission is one-way: caller to callees, for one bounded operation.

### 2. Unbounded lifetime

A `ThreadLocal` value survives until `remove()` is called or the thread dies. Forget `remove()` on a pooled servlet thread and the value leaks into later work — the retention and cross-request contamination covered in the leak post. A scoped binding's lifetime is visible in the code structure: it ends when `run(...)` or `call(...)` ends, normally or by exception.

### 3. Expensive inheritance

`InheritableThreadLocal` copies the parent's mappings into each child thread's own map. The child gets separate storage because either thread may later call `set()` independently. Immutable scoped bindings do not need that copy-per-child model: structured child threads can share the parent's immutable value safely.

Notice what is *not* on the list: "virtual threads leak ThreadLocals." They do not leak in the pooled-thread sense, because virtual threads are not normally pooled and reused for unrelated requests. The virtual-thread argument is about aggregate footprint and propagation, not the old servlet-pool leak.

## A request-context holder you can actually migrate to

Do not scatter public `ScopedValue` fields across the codebase. The scoped value itself is a capability: code that can access it can read the binding, and code that can access the binding operation can establish one. Keep the key private and expose a small, deliberate API.

Start with an immutable payload:

```java
public record RequestContext(
        String requestId,
        String tenantId,
        String userId) {

    static RequestContext from(HttpServletRequest request) {
        return new RequestContext(
                request.getHeader("X-Request-Id"),
                request.getHeader("X-Tenant-Id"),
                request.getRemoteUser());
    }
}
```

Then one owner for the key:

```java
import java.util.Objects;
import java.util.Optional;

public final class RequestContexts {

    private static final ScopedValue<RequestContext> CURRENT =
            ScopedValue.newInstance();

    private RequestContexts() {
    }

    public static ScopedValue.Carrier where(RequestContext context) {
        return ScopedValue.where(CURRENT, Objects.requireNonNull(context));
    }

    public static RequestContext current() {
        return CURRENT.orElseThrow(() ->
                new IllegalStateException("No request context is bound"));
    }

    public static Optional<RequestContext> currentIfBound() {
        return CURRENT.isBound()
                ? Optional.of(CURRENT.get())
                : Optional.empty();
    }
}
```

The two read styles are intentional:

- Use `current()` for tenant, principal, and other values that business logic **requires**. Failing fast is better than quietly processing under a default tenant.
- Use `currentIfBound()` or `orElse(...)` only for optional decoration, such as adding a correlation ID when one happens to exist.

For a servlet filter, the old shape:

```java
CURRENT.set(context);
try {
    chain.doFilter(request, response);
} finally {
    CURRENT.remove();
}
```

becomes a binding around the chain:

```java
RequestContext context = RequestContext.from(request);

try {
    RequestContexts.where(context).call(() -> {
        chain.doFilter(request, response);
        return null;
    });
} catch (IOException | ServletException | RuntimeException e) {
    throw e;
} catch (Exception e) {
    // Carrier.call can surface the operation's checked exception type;
    // the filter narrows it back to the exceptions it is allowed to throw.
    throw new IllegalStateException(
            "Unexpected checked exception from the filter chain", e);
}
```

Downstream code no longer needs the request object just to recover context:

```java
public OrderView loadOrder(long orderId) {
    RequestContext context = RequestContexts.current();

    audit.record(context.requestId(), context.tenantId(), orderId);
    return orders.findForTenant(context.tenantId(), orderId);
}
```

That is the proper use case: framework and cross-cutting context that would otherwise be added to every method signature only so the fifth layer can use it. If a value is a real business argument that the next method needs to do its job, pass the parameter. Do not hide ordinary data flow inside a scoped value merely because you can.

## Inheritance: structured scopes only

Scoped bindings cross threads only in the structured case. The JDK 25 API documentation says bindings are captured when a `StructuredTaskScope` is created and inherited by all threads started through that scope's `fork(...)` method.

```java
// JDK 25 preview API (JEP 505). On JDK 27 (JEP 533) this shape changes:
// join() also throws ExecutionException — see the linked post below.
Dashboard loadDashboard(String userId) throws InterruptedException {
    // Called while the request context is bound.
    try (var scope = StructuredTaskScope.open()) {
        var user = scope.fork(() -> userClient.find(userId));
        var orders = scope.fork(() -> orderClient.countFor(userId));

        scope.join();

        return new Dashboard(user.get(), orders.get());
    }
}
```

If `userClient.find(...)` calls `RequestContexts.current()`, it sees the same request context as the parent. JEP 506 is explicit that the parent's bindings are not copied into the child thread. The child reads the shared immutable context.

Two caveats keep this honest:

1. **`StructuredTaskScope` is still a preview API in JDK 25** (JEP 505). `ScopedValue` is final; using it with structured scopes requires `--enable-preview` and acceptance of preview-API churn. My [structured concurrency post](https://blogs.rameskum.com/posts/structured-concurrency-java-jep533/) tracks how that API changed again in later rounds. Failure shape, pinned to this post's version: on JDK 25 a failed subtask surfaces from `join()` as the unchecked `StructuredTaskScope.FailedException`, which is why the signature above declares only `InterruptedException`. On JDK 27, `join()` throws `ExecutionException` instead.
2. **Other thread handoffs do not inherit the binding.** A plain `new Thread`, an arbitrary `ExecutorService` task, a `CompletableFuture` continuation, and `ForkJoinPool` work do not get structured-scope inheritance. If the work can execute after the parent operation exits, pass the context explicitly in the task or message. That is not a limitation to work around with a static field; it is the lifetime rule doing its job.

A child subtask may establish a nested binding for its own callees. That changes only that subtask's dynamic extent, not the parent's binding or a sibling's view.

Also remember the cross-thread rule from the API documentation: a value shared across threads must be immutable, or every access to it must be appropriately synchronized. A scoped reference does not make a JDBC `Connection`, a `StringBuilder`, or a mutable cache safe to share across forks.

## The memory math, without the folklore

You will see claims that `ScopedValue` uses "no memory" or that `ThreadLocal` always leaks on virtual threads. Both are wrong.

### Platform threads in a pool

The dangerous case remains the familiar one: a value is set, `remove()` is missed, and the thread returns to a pool instead of dying. Retention lasts as long as the pooled thread and can contaminate the next request. Scoped bindings remove that failure mode because there is no post-operation state to forget to clear.

### Virtual threads

Virtual threads are cheap, not free, and they are normally dedicated to one task rather than pooled. A `ThreadLocal` entry still occupies storage while the virtual thread is alive, and millions of *concurrently live* virtual threads can each carry their own map entries.

Fan-out makes inheritance matter. Suppose 10,000 requests are in flight and each fans out to four child threads. With inheritable thread locals, each child needs its own map entries for the inherited variables. The values may point at the same objects — inheritance copies the map structure and references, not a deep copy of your object graph — but the per-thread storage multiplies.

With structured concurrency and scoped values, the child threads share the immutable binding instead of receiving a copied map. The context object itself still exists, of course. What disappears is the need to duplicate binding storage per child and to clean it up afterward.

The correct summary is the one in JEP 506: scoped values have lower space and time costs, especially with virtual threads and structured concurrency. It is not "zero cost," and there is no universal percentage saving to quote without measuring your own objects, concurrency, and access pattern.

## Migrating an MDC-style holder

Here is the migration trap: changing your own holder to `ScopedValue` does not teach a ThreadLocal-based library to read it.

SLF4J's MDC (backed by Logback's ThreadLocal adapter) is the common example. If your application code moves to `RequestContexts.current()`, but the logging encoder still reads MDC, you need a bridge at the boundary where the library takes over.

For code running in the request thread, that may mean populating MDC once around the operation and restoring whatever was there before. For a structured subtask, install the values it needs inside that subtask — saving and restoring the previous values, never just removing the keys, because a tracing filter or an outer bridge may have put them there first:

```java
static void runWithRequestMdc(Runnable operation) {
    RequestContext context = RequestContexts.current();

    String previousRequestId = MDC.get("requestId");
    String previousTenantId = MDC.get("tenantId");

    MDC.put("requestId", context.requestId());
    MDC.put("tenantId", context.tenantId());
    try {
        operation.run();
    } finally {
        restore("requestId", previousRequestId);
        restore("tenantId", previousTenantId);
    }
}

private static void restore(String key, String previousValue) {
    if (previousValue == null) {
        MDC.remove(key);
    } else {
        MDC.put(key, previousValue);
    }
}
```

Treat that as an adapter, not as the new application model:

- Application and domain code read `RequestContexts`.
- Only the logging boundary translates the immutable context into MDC's ThreadLocal representation.
- Every bridge saves and restores the previous MDC values in a `finally` block — the same discipline from the leak post, now isolated in one place.

The same rule applies to framework holders. Do not assume `RequestContextHolder`, a security-context holder, a tracing library, or a database framework has adopted `ScopedValue` just because your code has. Check its actual propagation mechanism, and bridge or pass values explicitly where it has not.

## Test the semantics, not just the happy path

Because the lifetime rules are the feature, test them directly. These tests use only the final JDK 25 `ScopedValue` API — no preview flag:

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.NoSuchElementException;
import org.junit.jupiter.api.Test;

class ScopedValueTest {

    private static final ScopedValue<String> REQUEST_ID =
            ScopedValue.newInstance();

    private static final ScopedValue<String> TENANT =
            ScopedValue.newInstance();

    @Test
    void bindingEndsWhenTheScopeEnds() {
        assertFalse(REQUEST_ID.isBound());

        ScopedValue.where(REQUEST_ID, "req-1").run(() -> {
            assertTrue(REQUEST_ID.isBound());
            assertEquals("req-1", REQUEST_ID.get());
        });

        assertFalse(REQUEST_ID.isBound());
        assertThrows(NoSuchElementException.class,
                () -> REQUEST_ID.get());
    }

    @Test
    void nestedBindingShadowsAndThenRestoresTheOuterValue() {
        ScopedValue.where(TENANT, "acme").run(() -> {
            assertEquals("acme", TENANT.get());

            ScopedValue.where(TENANT, "globex").run(() ->
                    assertEquals("globex", TENANT.get()));

            assertEquals("acme", TENANT.get());
        });
    }

    @Test
    void bindingIsGoneEvenWhenTheOperationThrows() {
        assertThrows(IllegalStateException.class, () ->
                ScopedValue.where(REQUEST_ID, "req-1").run(() -> {
                    throw new IllegalStateException("boom");
                }));

        assertFalse(REQUEST_ID.isBound());
    }

    @Test
    void unboundReadsCanUseAFallbackOrFailExplicitly() {
        assertEquals("no-request", REQUEST_ID.orElse("no-request"));

        IllegalStateException exception = assertThrows(
                IllegalStateException.class,
                () -> REQUEST_ID.orElseThrow(() ->
                        new IllegalStateException("No request context")));

        assertEquals("No request context", exception.getMessage());
    }
}
```

Add an integration test for structured fan-out separately if you use it, and run that test with the preview settings required by your JDK. The important cases are: both subtasks see the parent's context, neither sibling can change the parent's value, and code after the request operation finds the value unbound.

## When to keep ThreadLocal

JEP 506's own migration advice is refreshingly un-dogmatic: migrate when the thread local is being used for one-way transmission of unchanging data. Keep `ThreadLocal` — or choose another design entirely — when it is not.

| Pattern | Better fit |
|---|---|
| Immutable request, tenant, principal, or tracing context passed from caller to callees | `ScopedValue` |
| Context that must be visible to structured-concurrency subtasks | `ScopedValue` + `StructuredTaskScope` |
| Mutable per-thread cache of an expensive object | `ThreadLocal` |
| A builder or accumulator that callees mutate and the caller reads later | Pass a parameter or return a value; neither hidden mechanism is ideal |
| Context for detached async work or a message consumer on another process | Put it in the task/message explicitly |
| A library API that is itself ThreadLocal-based, such as MDC | Keep the library mechanism and bridge at the boundary |
| Data that must survive after the creating method returns | Do not use `ScopedValue` |

The old `SimpleDateFormat` example still illustrates the cache case: a mutable, expensive, thread-confined formatter was commonly stored per thread. Modern code should usually prefer immutable `DateTimeFormatter` in a shared field — but the category is real. Per-thread mutable state did not become illegitimate because a better context mechanism arrived.

## The migration checklist

1. **Inventory every `ThreadLocal`.** Classify it as immutable context, mutable cache, two-way communication, or "nobody remembers."
2. **Migrate only the immutable, one-way context first.** Request IDs, tenant IDs, principals, and feature context are the clean wins.
3. **Make the payload deeply immutable.** Prefer a record; copy maps and collections before binding.
4. **Bind at one entry point.** A filter, interceptor, RPC handler, or message consumer owns the `where(...).run(...)` / `call(...)` call.
5. **Fail explicitly for required context.** Use `orElseThrow(...)` for tenant and security data. Reserve defaults for optional decoration.
6. **Audit every thread hop.** Structured scopes inherit; executors, futures, reactive chains, and detached callbacks do not magically do so.
7. **Bridge ThreadLocal-based libraries deliberately.** MDC and framework holders need an adapter or their own supported propagation.
8. **Test the boundaries.** Bound inside, unbound outside, nested shadowing, exceptions, and structured subtasks.

## The 2-minute interview answer

> "`ScopedValue` is JDK 25's final replacement for the request-context use of `ThreadLocal`, standardized by JEP 506. You declare a `ScopedValue` key, bind it with `ScopedValue.where(key, value)`, and execute code with `run` or `call`. Everything called inside that dynamic extent can read the value with `get`; when the operation exits, the binding disappears automatically. There is no `set` or `remove`, and a nested binding only shadows the outer one for its own callees. That fixes ThreadLocal's unconstrained mutability and unbounded lifetime. For concurrency, bindings are captured by a `StructuredTaskScope` and inherited by its forked subtasks without copying a ThreadLocal map into every child thread; arbitrary executors and plain threads do not inherit them. It is not a universal ThreadLocal replacement — mutable per-thread caches and two-way communication still do not fit — and the value itself must be immutable or synchronized if child threads share it."

## Cheat sheet

| Fact | Detail |
|---|---|
| Status | Final in JDK 25, JEP 506; incubator in 20, preview in 21–24 |
| Declare | `static final ScopedValue<T> X = ScopedValue.newInstance()` |
| Bind + run | `ScopedValue.where(X, value).run(() -> work())` |
| Bind + return/throw | `ScopedValue.where(X, value).call(() -> work())` |
| Read | `X.get()`, `X.isBound()`, `X.orElse(fallback)`, `X.orElseThrow(supplier)` |
| Unbound `get()` | Throws `NoSuchElementException` |
| Rebinding | Nested `where(...)` shadows only inside the nested extent |
| Multiple values | Chain `.where(...)` calls on the carrier |
| Cross-thread rule | Inherited by `StructuredTaskScope` forks; not by arbitrary threads or executors |
| Inheritance cost | Bindings are shared, not copied from parent to child |
| Mutable payload | Still your problem — bind immutable data or synchronize access |
| Stale syntax | `runWhere` / `callWhere` were removed before final |
| Keep `ThreadLocal` for | Mutable per-thread caches and genuinely thread-confined state |

`ThreadLocal` asked every developer to remember two things forever: pair every `set()` with a `remove()`, and never let a value wander into the wrong thread. `ScopedValue` turns both into structure. On JDK 25, for request context, that is the migration worth making.
