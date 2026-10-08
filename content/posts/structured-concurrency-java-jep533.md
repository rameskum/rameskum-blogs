---
title: 'Structured Concurrency in Java: What Virtual Threads Didn''t Fix (JEP 533, JDK 27)'
date: 2026-10-08T10:00:00-04:00
draft: false
ShowToc: true
description: >-
  Virtual threads gave Java cheap concurrency; structured concurrency gives it
  correctness — no leaked threads when a subtask fails. But the API is still in
  preview (JEP 533, seventh round, JDK 27) and most examples online show the
  obsolete pre-JDK-25 API. Here is the current StructuredTaskScope/Joiner API
  verified against the JEP, plus ScopedValue (final since JDK 25) done honestly.
tags:
  - java
  - backend
  - concurrency
  - system-design
  - interview
categories: article
keywords:
  - structured concurrency java
  - StructuredTaskScope Joiner API
  - JEP 533
  - ScopedValue vs ThreadLocal
  - virtual threads error handling
---

Virtual threads (final since JDK 21) made blocking I/O cheap. But cheap threads didn't fix the hard part: when you fan out to five subtasks and one fails, who cancels the other four? With `ExecutorService` + `Future`, the answer is "you, manually, and you'll get it wrong." That failure mode — leaked threads, swallowed errors, unobserved work — is what structured concurrency exists to kill.

One warning before the code: this API is still moving. Everything below is verified against **JEP 533, the seventh preview, targeting JDK 27** — and most blog examples you'll find online show the *obsolete* API. Get the status table right first, then the code.

## Status check: where Project Loom actually stands

| Feature | Status | Since |
|---|---|---|
| Virtual threads (JEP 444) | **Final** | JDK 21 |
| `ScopedValue` (JEP 506) | **Final** | JDK 25 |
| `StructuredTaskScope` (JEP 533) | **Preview** — 7th round, `--enable-preview` required | JDK 27 |

The structured concurrency API was substantially reworked in JDK 25 (JEP 505): the public constructors (`new StructuredTaskScope.ShutdownOnFailure()`) were replaced with static factories (`StructuredTaskScope.open(...)`) and a `Joiner` policy interface, with further refinements in JDK 26 (JEP 525) and JDK 27 (JEP 533). If you copy a 2023-era example onto JDK 27, it will not compile. When you read any structured-concurrency example, check which JDK it targets — the pre-25 and post-25 APIs are different languages.

## The problem virtual threads don't solve

The JEP's canonical example: a request handler fans out to `findUser()` and `fetchOrder()` via an `ExecutorService`, then joins on the two futures. Three failure modes, all silent:

1. `findUser()` throws → `handle()` fails on `user.get()`, but `fetchOrder()` keeps running in its thread. **Thread leak.**
2. The handler thread is interrupted → interruption doesn't propagate; both subtasks leak.
3. `fetchOrder()` fails fast while `findUser()` is slow → `handle()` blocks on `user.get()` anyway, waiting for work whose result it will throw away.

The root cause: the task-subtask relationship exists only in the programmer's head. `Future` lets *any* thread join *any* subtask; nothing enforces that subtasks die with their parent. Structured concurrency makes the relationship real: subtasks cannot outlive the code block that forked them.

## The current API: open, fork, join

The shape in JDK 27 (preview):

```java
UserProfile loadProfile(String userId) throws ExecutionException, InterruptedException {
    try (var scope = StructuredTaskScope.open(Joiner.allSuccessfulOrThrow())) {
        Subtask<User> user = scope.fork(() -> userService.find(userId));
        Subtask<List<Order>> orders = scope.fork(() -> orderService.list(userId));

        scope.join(); // throws ExecutionException if any subtask failed
        return new UserProfile(user.get(), orders.get());
    }
}
```

Four things to notice:

- **`open(...)` takes a `Joiner`, not a constructor call.** The joiner is the completion policy: what `join()` returns and what it throws. Create a fresh joiner per scope — the JEP is explicit that joiners must never be reused across scopes.
- **`fork()` returns a `Subtask`, not a `Future`.** The old API returned `Future`, which invited the familiar-but-wrong `get()`-anywhere pattern. `Subtask` exposes `state()` and `get()`, meant to be read *after* `join()` returns.
- **`join()` throws `ExecutionException`** with the failed subtask's exception as the cause when the outcome is a failure — one place where all errors surface, instead of N `future.get()` calls each throwing.
- **The try-with-resources is the structure.** When the block exits, every subtask is done — no orphans, by construction. That guarantee is the entire point.

Error handling stays readable with pattern matching on the cause:

```java
try (var scope = StructuredTaskScope.open()) {
    // ... fork subtasks, join
} catch (ExecutionException e) {
    switch (e.getCause()) {
        case IOException ioe -> log.warn("downstream I/O failed", ioe);
        case null, default -> throw e;
    }
}
```

(The zero-arg `open()` uses the same completion policy as `allSuccessfulOrThrow()` but returns `null` from `join()` instead of a result list — handy when your subtasks have heterogeneous types and you read them via `Subtask.get()`.)

## Joiners: pick the completion policy

Two factory methods cover most cases:

- **`Joiner.allSuccessfulOrThrow()`** — wait for everything; fail the scope if anything failed. The default fan-out.
- **`Joiner.anySuccessfulOrThrow()`** — first success wins; the scope is cancelled and remaining subtasks are interrupted. The redundant-service race, straight from the JEP:

```java
<T> T race(Collection<Callable<T>> tasks) throws ExecutionException, InterruptedException {
    try (var scope = StructuredTaskScope.open(Joiner.<T>anySuccessfulOrThrow())) {
        tasks.forEach(scope::fork);
        return scope.join(); // result of the first successful subtask
    }
}
```

There are also `awaitAllSuccessfulOrThrow()` and `allUntil(Predicate)` for "wait for everything but stop early on a condition," plus overloads taking a `Function` to produce a custom exception type instead of `ExecutionException`. And if none fit, `Joiner` is an interface you implement directly (`onFork`/`onComplete`/`result`/`timeout`) — the JEP includes a collecting-joiner example. Note the deliberate non-goal, though: this is *not* a replacement for `ExecutorService`. Unrestricted task submission still has its place; structured concurrency is for work that must not outlive its parent.

Timeouts are a scope configuration, not a joiner hack:

```java
try (var scope = StructuredTaskScope.open(Joiner.allSuccessfulOrThrow(),
        cf -> cf.withTimeout(Duration.ofSeconds(2)))) {
    // ...
    scope.join();
} catch (ExecutionException e) {
    // e.getCause() is a CancelledByTimeoutException on timeout
}
```

## ScopedValue: the ThreadLocal replacement, done honestly

`ScopedValue` went final in JDK 25 (JEP 506) — no preview flag, safe to use today. It looks like the answer to "how do I pass request context through layers without ThreadLocal," and mostly it is, but let's be precise about *why* it's better, because the common claims are half-wrong:

```java
private static final ScopedValue<String> TENANT = ScopedValue.newInstance();

// at the request boundary:
ScopedValue.where(TENANT, tenantId).run(() -> handleRequest());

// anywhere downstream, including forked subtasks:
String tenant = TENANT.get();
```

What ScopedValue genuinely buys you over `ThreadLocal`:

- **Scoped lifetime.** The binding exists for the dynamic extent of the `run`/`call` block and vanishes after — no `try/finally { remove(); }` discipline, no leak-across-requests bug class.
- **Immutability within the scope.** Nothing downstream can overwrite the value. For security-flavored context (tenant, principal), that's a hardening property, not just hygiene.
- **Composes with structured scopes.** Scoped values are inherited by child threads — including subtasks forked in a `StructuredTaskScope` — which is exactly the propagation story `ThreadLocal` never had a clean answer for.

What it doesn't do: it doesn't make `ThreadLocal` leak on virtual threads — virtual threads are never pooled or reused, so that particular folk claim is wrong. The honest case is the three points above.

## Should you ship it?

The preview status is the whole decision:

- **Virtual threads + `ScopedValue`: final. Ship them.** This is the production-ready core of Loom today.
- **`StructuredTaskScope`: seventh preview.** The concepts are stable and the JEP authors expect finalization, but the API has changed in *every* round (constructors → factories in 25, joiner generics in 27). Building production code on it means accepting churn and `--enable-preview` in your build. Learn it now, prototype with it, but pin your examples to a JDK version and expect edits.

That churn is also why this article exists in this form: half the examples on the internet target the dead API. When the feature finally lands, the winners will be the engineers who learned the *concepts* (structure, joiners, scoped values) rather than memorizing one preview's syntax.

## The 2-minute interview answer

> "Virtual threads made blocking I/O cheap but didn't fix error handling across fan-out — with ExecutorService and Future, a failed subtask leaks its siblings. Structured concurrency fixes that by making subtasks unable to outlive the block that forked them: you open a StructuredTaskScope with a Joiner policy like allSuccessfulOrThrow, fork subtasks, and join() surfaces every failure as one ExecutionException. The API is still in preview — seventh round, JEP 533, JDK 27 — and it was reworked in JDK 25 from constructors to static factories, so most online examples are stale. ScopedValue, the ThreadLocal replacement, is final since JDK 25 and composes with scopes because child threads inherit the bindings. In production today I'd ship virtual threads plus ScopedValue, and prototype with StructuredTaskScope."

## Cheat sheet

| Task | How (JDK 27 preview API) |
|---|---|
| Open a scope | `StructuredTaskScope.open(Joiner.allSuccessfulOrThrow())` in try-with-resources |
| Fork a subtask | `Subtask<T> st = scope.fork(() -> ...)` — not a `Future` |
| Wait + collect errors | `scope.join()` — throws `ExecutionException`, failed subtask's exception as cause |
| Race (first success wins) | `Joiner.anySuccessfulOrThrow()` |
| Timeout | `open(joiner, cf -> cf.withTimeout(duration))` → `CancelledByTimeoutException` cause |
| Custom policy | Implement `Joiner` (`onFork`/`onComplete`/`result`/`timeout`); never reuse a joiner across scopes |
| Request context | `ScopedValue.newInstance()` + `ScopedValue.where(v, x).run(...)` — final since JDK 25 |
| Observability | `jcmd <pid> Thread.dump_to_file -format=json` shows the scope/thread hierarchy |
| What to ship today | Virtual threads + `ScopedValue` (final); prototype `StructuredTaskScope` (preview) |

Structured concurrency isn't a faster `ExecutorService` — it's the end of a bug class. Learn the concepts now, pin the syntax to your JDK, and double-check every example you copy: if it says `new StructuredTaskScope.ShutdownOnFailure()`, it's from 2023.
