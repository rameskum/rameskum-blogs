---
title: 'The ThreadLocal Leak: Why Your Spring Boot Heap Grows Until the 3 A.M. Restart'
date: 2026-10-16T08:30:00+05:30
draft: false
ShowToc: true
description: >-
  A ThreadLocal you set but never remove doesn't die with the request — it
  lives as long as the pooled thread, holding your object graph hostage. How
  the weak-key/strong-value trap works, how to find it in a heap dump, and
  the three fixes: remove() discipline, framework-managed holders, and
  ScopedValue.
tags:
  - java
  - spring-boot
  - backend
  - interview
categories: article
keywords:
  - threadlocal memory leak java
  - threadlocalmap stale entry
  - spring boot heap leak
  - scopedvalue vs threadlocal
  - java memory leak heap dump
---

The symptom is always the same: heap usage climbs a little every day, GC reclaims less and less, and eventually someone sets a cron to restart the service at 3 AM. The heap dump shows nothing obviously wrong — no giant cache, no runaway collection. Just thousands of ordinary objects that should have died with their requests, all reachable through one path: `Thread → ThreadLocalMap → Entry`.

This is the ThreadLocal leak, and it's the quietest memory leak in Java.

## The mechanism: weak key, strong value

Every thread carries its own `ThreadLocalMap` — a hash table where the key is your `ThreadLocal` and the value is whatever you `set()`. Here's the trap, straight from the JDK's design:

- The **key** is a **weak reference** to the `ThreadLocal` instance.
- The **value** is a **strong reference** to your object.

If the `ThreadLocal` itself gets garbage collected (say it was a local variable that went out of scope), the key becomes null — a *stale entry*. But the value stays strongly reachable from the live thread. The map has cleanup code for stale entries, but it is opportunistic: it runs only when the map is touched — `set()` and `remove()` clean up around the slot they touch, and even a `get()` that misses its key can expunge stale entries it encounters. Cleanup is opportunistic and only runs when the map is touched; a stale entry on a map nobody touches is never cleaned.

Now add the thread pool, and the leak becomes structural. Tomcat's worker threads, your `ExecutorService` threads — they never die. They get reused for request after request. A `ThreadLocal` you `set()` at the start of a request and never `remove()` stays attached to that pooled thread **forever**, along with everything your value references. Two hundred worker threads each holding the last user session graph they handled is 200 leaked object graphs that never shrink.

In production this stops being theoretical: the retained values are reachable from live threads, so GC can never reclaim them, old-gen occupancy ratchets up, and a bigger heap only delays the OOM — GC can't collect reachable objects. The fix is always the same shape: `remove()` on that path, or stop holding the object in a `ThreadLocal` at all.

## The second leak: cross-request contamination

The memory leak has an evil twin. That pooled thread holding the last request's `UserSession`? The *next* request on the same thread can read it. A missing `remove()` doesn't just leak memory — it leaks data across requests. Tenant A's context visible to tenant B's request is a security incident wearing a memory leak's clothes.

This is also why the "just use MDC" advice needs the fine print: Logback's MDC is ThreadLocal-backed. Setting `requestId` in a filter without removing it in a `finally` (or after the chain) leaks it into the next request's logs *and* memory. `MDC.remove("requestId")`, not `MDC.clear()` — clear would also nuke tracing context other filters set.

## Reproducing it: the filter that forgets

The classic shape in Spring Boot — a filter stashing request context:

```java
@Component
public class TenantFilter extends OncePerRequestFilter {

    private static final ThreadLocal<TenantContext> CURRENT = new ThreadLocal<>();

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        CURRENT.set(TenantContext.from(req));  // set...
        chain.doFilter(req, res);
        // ...but never removed. The pooled thread keeps the latest one forever.
    }

    public static TenantContext current() { return CURRENT.get(); }
}
```

Be precise about what this filter costs, because it's less than the horror stories claim and worse than it looks. The `static` ThreadLocal's key is strongly held by the class, so the entry is never stale, and the next request on the same thread simply overwrites the value: retention is bounded at one `TenantContext` per worker thread. The damage is (1) a fixed, permanent tax — N worker threads × the whole context graph (user, tenant config, maybe a session), with the retained values also pinning the webapp classloader across redeploys — and (2) cross-request contamination: any code path that reads `current()` before this filter's `set()` runs — an earlier filter, an error dispatch, an async handoff — sees the *previous* tenant's context.

The unbounded version — the one that really does climb until the 3 AM restart — needs a different shape: `ThreadLocal` instances created per request or per object, whose keys get garbage-collected while their values stay strongly referenced in a pooled thread's map, accumulating faster than the opportunistic cleanup reaches them; or a value that itself accumulates, like a `List` held in a ThreadLocal that every request appends to. If the heap dump shows thousands of entries per thread rather than one context per thread, you're looking at one of those.

## Diagnosing it: reading the heap dump

When the heap is climbing and nothing obvious is growing:

1. **Take the dump:** `jmap -dump:live,format=b,file=heap.hprof <pid>` (or let `-XX:+HeapDumpOnOutOfMemoryError` do it for you — you want this flag on in production anyway).
2. **Open it in MAT** (Eclipse Memory Analyzer) and look at the dominator tree. Sort by retained heap. Threads near the top that shouldn't be there are your suspects.
3. **Follow the path:** right-click a suspect thread → *Path to GC Roots* → *exclude weak references*. You'll see `Thread → threadLocals → ThreadLocalMap → Entry[] → your leaked object`. The `Entry` with a null key and a fat value is the stale one.
4. **Confirm the pattern:** many threads each retaining similar object graphs through `ThreadLocalMap` = the filter-that-forgets. One thread retaining something enormous = a single oversized value.

The tell that separates ThreadLocal leaks from ordinary retention: the objects are reachable *only* through thread locals of pooled threads, and they correspond to per-request data that should have died.

## Fix 1: `remove()` discipline — the non-negotiable rule

Every `set()` gets a `remove()` in a `finally`. No exceptions, no "I'll do it at the end of the method":

```java
@Override
protected void doFilterInternal(HttpServletRequest req,
                                HttpServletResponse res,
                                FilterChain chain) throws IOException, ServletException {
    CURRENT.set(TenantContext.from(req));
    try {
        chain.doFilter(req, res);
    } finally {
        CURRENT.remove();  // the thread goes back to the pool clean
    }
}
```

Better yet, centralize it: one filter (or Spring interceptor) owns the set/remove pair for request-scoped context, and application code only calls `current()`. The bug class disappears when `set()` and `remove()` can't be separated.

## Fix 2: let the framework own the lifecycle

Prefer holders that something else cleans up. Spring's `RequestContextHolder` is reset by `RequestContextFilter` after every request — you get request-scoped storage without owning the cleanup. `SecurityContextHolder` works the same way: what clears it is the security filter chain at the end of the request, not the storage mode configured on it.

One caution from production: *too many* ThreadLocals is its own disease. Stale-entry cleanup is incremental — each `get()`/`set()` expunges only a few slots — so a thread whose map has bloated to thousands of entries pays scanning and rehashing costs on every later operation, and the stale values sit there until the cleanup reaches them. Don't fix the leak by adding twelve more ThreadLocals.

## Fix 3: ScopedValue — the modern answer

`ScopedValue` is the designed replacement for request-scoped ThreadLocal use — final in JDK 25 (it was a preview API on 21–24, so on those versions it needs `--enable-preview`):

```java
private static final ScopedValue<TenantContext> CURRENT = ScopedValue.newInstance();

// bind for exactly one scope — no remove() exists because none is needed.
// call(), not run(): run() takes a Runnable, which cannot throw the checked
// IOException/ServletException that chain.doFilter declares; call() propagates
// them through its generic signature so the enclosing method can rethrow them.
ScopedValue.where(CURRENT, TenantContext.from(req)).call(() -> {
    chain.doFilter(req, res);
    return null;
});
// outside the scope, the binding is simply gone
```

Why it's better: the binding is **immutable** and **scope-bounded**. There's no `set()` to forget to pair with `remove()` — the value is visible only inside the dynamic scope and unreachable the moment the block exits. No stale entries, no cross-request contamination, no cleanup code to write. One precision on propagation: bindings are inherited only by threads forked inside a `StructuredTaskScope` — the designed path, where ThreadLocal's per-thread table is both expensive and semantically wrong. A thread started any other way — `Thread.ofVirtual().start(...)`, a task on a plain executor — does **not** see the binding. Scoped values follow the scope tree, not thread creation.

The migration rule: if you're on JDK 25 and the data is request-scoped, `ScopedValue` deletes the bug class. If you're stuck on 17, the `try/finally` + `remove()` discipline is your answer.

## The 2-minute interview answer

> "Each thread has a ThreadLocalMap where the key is a weak reference to the ThreadLocal but the value is a strong reference to your object. If you set a value and never remove it, the entry stays reachable from the live thread — and in a servlet container the threads are pooled and never die, so each worker thread permanently retains its last context object — and ThreadLocal instances created per request leave stale entries whose values pile up without bound. That's the memory leak, and the same mechanism leaks data across requests, which is a security problem too. Cleanup is opportunistic — it runs only when the map is touched — so entries on untouched maps are never reclaimed. Diagnose with a heap dump: follow Thread → ThreadLocalMap → Entry in MAT. Fix it with try/finally remove() in one owning filter, prefer framework-managed holders like RequestContextHolder, and on JDK 25 use ScopedValue — immutable, scope-bounded, no remove needed, and inherited by StructuredTaskScope forks rather than copied into every thread."

## Cheat sheet

| Fact | Detail |
|---|---|
| The trap | Weak key, **strong value** in `ThreadLocalMap.Entry` |
| Why pools make it fatal | Threads never die → values never released |
| Stale entry cleanup | Opportunistic — only when the map is touched (`get`/`set`/`remove`); untouched entries persist |
| Twin failure | Cross-request data contamination, not just memory |
| MDC rule | `MDC.remove("requestId")` in finally — never `clear()` |
| Diagnose | `jmap -dump` → MAT → Thread → ThreadLocalMap → Entry |
| Fix (JDK 17) | `try { set(); ... } finally { remove(); }` in one owning filter |
| Fix (JDK 25) | `ScopedValue` — scope-bounded, immutable, no cleanup (final in 25; preview on 21–24) |
| Don't | Add more ThreadLocals to fix it (linear stale-scan cost) |

The 3 AM restart is treating the symptom. Find the filter that sets without removing, pair every `set()` with a `remove()`, and on JDK 25 let `ScopedValue` make the bug impossible.
