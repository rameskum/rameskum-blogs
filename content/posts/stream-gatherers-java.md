---
title: 'Stream Gatherers in Java: The Stream API Finally Learned to Window, Scan, and Fold'
date: 2026-11-16T07:30:00-05:00
draft: false
ShowToc: true
description: >-
  Since Java 8, the Stream API's intermediate operations were a fixed set —
  map, filter, flatMap, and friends — while only terminal operations were
  extensible through collect(). Stream Gatherers (JEP 485, final in JDK 24)
  close that gap: windowFixed and windowSliding, scan and fold, bounded
  concurrent mapping, and a clean way to write your own stateful,
  short-circuiting operations. Every behavior below was run, not assumed:
  all 39 tests run twice on JDK 25.
tags:
  - java
  - streams
  - backend
categories: article
keywords:
  - stream gatherers java
  - JEP 485 stream gatherers
  - Gatherers windowFixed windowSliding
  - stream scan fold java
  - mapConcurrent virtual threads
---

Since Java 8, the Stream API has had an asymmetry: terminal operations were extensible — `collect(Collector)` lets you define any summary you like — but intermediate operations were a fixed menu. If you wanted windows, running totals, or distinct-by-key in the middle of a pipeline, you either broke out of the pipeline into a loop, or bent a `Collector` into doing an intermediate operation's job. Stream Gatherers close that gap. `Stream.gather(Gatherer)` is to intermediate operations what `collect(Collector)` is to terminal operations — that framing is JEP 485's own, and it is the fastest way to understand the feature.

The history, because it decides whether you can use this today: previewed as JEP 461 in JDK 22, re-previewed as JEP 473 in JDK 23, and finalized as JEP 485 in JDK 24 *without change*. On JDK 24 and later there is no `--enable-preview` flag to set. Everything below was compiled and run on Temurin JDK 25.0.5 with `--release 25` and no preview flag. If your floor is JDK 21, gatherers need an upgrade to 24+ — there is no backport.

A gatherer is a `Gatherer<T, A, R>`: it consumes elements of type `T`, keeps optional state of type `A`, and produces elements of type `R`. It can transform one-to-one, one-to-many, many-to-one, or many-to-many; it can hold state across elements; it can short-circuit; and, with a combiner, it can run in parallel. One scope note, verified against the JDK 25 API itself: `gather` exists only on `Stream<T>`. There is no `gather` on `IntStream`, `LongStream`, or `DoubleStream` — box first with `.boxed()` if your source is primitive.

Yesterday's [sliding-window post](/posts/sliding-window-java-interviews/) covered windows over arrays and strings with indices. This post is the stream version of the same idea, plus the operations that only make sense on a stream: running scans, mid-pipeline folds, and bounded concurrent mapping.

## The problem gatherers solve, in one example

JEP 485 motivates the feature with a task that sounds trivial: take an infinite stream of integers, group it into fixed windows of three, keep the first two windows. The old workaround put the grouping in a terminal `Collector`, which cannot signal "I'm done" while elements keep arriving — so the `limit` had to move *before* the grouping, sized by hand, and the combiner had to throw because ordered grouping cannot parallelize:

```java
// The pre-gatherer workaround, condensed from JEP 485. Note limit(3 * 2):
// the collector needs the caller to pre-compute how many elements to take.
var result = Stream.iterate(0, i -> i + 1)
        .limit(3 * 2)
        .collect(Collector.of(
                () -> new ArrayList<ArrayList<Integer>>(),
                (groups, element) -> {
                    if (groups.isEmpty() || groups.getLast().size() == 3) {
                        var current = new ArrayList<Integer>();
                        current.add(element);
                        groups.addLast(current);
                    } else {
                        groups.getLast().add(element);
                    }
                },
                (left, right) -> {
                    throw new UnsupportedOperationException("Cannot be parallelized");
                }));
// result ==> [[0, 1, 2], [3, 4, 5]]
```

With gatherers, the window is an intermediate operation again, and `limit` goes back where it belongs — after the operation it limits:

```java
var result = Stream.iterate(0, i -> i + 1)
        .gather(Gatherers.windowFixed(3))
        .limit(2)
        .toList();
// result ==> [[0, 1, 2], [3, 4, 5]]
```

This works on an infinite stream because windows are emitted as they fill, and the downstream `limit` rejecting further elements stops the pull from upstream. I measured it with a counting source: exactly 6 elements were pulled from the infinite stream, no more. That laziness-plus-short-circuit combination is the whole point — a collector can never do it, because a collector only sees the end.

## Built-in 1: windows — windowFixed and windowSliding

The `Gatherers` class ships five built-in gatherers. Start with the two windows, because they answer the most common "streams can't do that" complaint: batching.

`windowFixed` groups elements into non-overlapping, encounter-ordered lists. The last window may be partial — the gatherer's finisher flushes whatever is left when the stream ends:

```java
List<List<Integer>> windows = Stream.of(1, 2, 3, 4, 5, 6, 7, 8)
        .gather(Gatherers.windowFixed(3))
        .toList();
// [[1, 2, 3], [4, 5, 6], [7, 8]]
```

`windowSliding` shifts by one element at a time, so consecutive windows overlap. It is the moving-average primitive:

```java
List<Double> averages = Stream.of(20, 22, 24, 23, 25)
        .gather(Gatherers.windowSliding(3))
        .map(w -> w.stream().mapToInt(Integer::intValue).average().orElseThrow())
        .toList();
// [22.0, 23.0, 24.0]
```

Trace: `[20, 22, 24]` averages `22.0`; slide one, `[22, 24, 23]` averages `23.0`; slide one, `[24, 23, 25]` averages `24.0`. Three windows from five readings, each a full window of three. Use fixed windows for "flush every N messages" batching; use sliding windows for moving averages, consecutive-pair diffs (`windowSliding(2)`), and n-gram-style views.

Now the edge cases, because they are where the two windows differ in a way that will surprise you once:

- **Empty stream:** both produce no windows at all. `Stream.empty().gather(Gatherers.windowFixed(3))` is `[]`.
- **Short stream, sliding window:** if the stream is *shorter* than the window, `windowSliding` still produces exactly one window holding every element. `Stream.of(1, 2).gather(Gatherers.windowSliding(5))` is `[[1, 2]]`, not `[]`. This is specified in the javadoc and I verified it by running it — do not assume a short stream yields nothing.
- **No trailing partials:** once a full sliding window has been emitted, the stream ends without partial windows. `Stream.of(1, 2, 3, 4, 5).gather(Gatherers.windowSliding(3))` is exactly `[[1, 2, 3], [2, 3, 4], [3, 4, 5]]` — there is no `[4, 5]` or `[5]` at the end. The single-partial-window rule above applies only when the stream never filled even one window.
- **Windows are unmodifiable.** Each emitted `List` throws `UnsupportedOperationException` on any mutator call. Copy it (`new ArrayList<>(window)`) if you need to mutate.
- **Validation is eager:** `windowFixed(0)` and `windowSliding(0)` throw `IllegalArgumentException` at the factory call, before any stream runs.

One performance note from the javadoc, stated as documentation rather than measured: windows may be allocated contiguously and eagerly, so a very large window size on a small stream can use excessive memory for the duration of the operation. Size windows to the data you actually have.

I property-checked both windows against hand-written subList logic over 200 seeded random inputs each: fixed windows always concatenate back to the input with every window but the last at full size; sliding windows always match `input.subList(i, i + k)` with the short-stream rule above. No exceptions.

## Built-in 2: scan and fold — the same signature, different timing

`scan` and `fold` take the same two arguments — a `Supplier` for the initial value and a `BiFunction` that combines the current accumulation with the next element — and differ only in *when* they emit. `fold` emits once, at the end. `scan` emits after every element. Same inputs, same function, two different streams:

```java
List<Integer> folded = Stream.of(1, 2, 3, 4, 5)
        .gather(Gatherers.fold(() -> 0, Integer::sum))
        .toList();
// [15]

List<Integer> scanned = Stream.of(1, 2, 3, 4, 5)
        .gather(Gatherers.scan(() -> 0, Integer::sum))
        .toList();
// [1, 3, 6, 10, 15]
```

`fold` is `reduce` as an intermediate operation: the result stays a `Stream`, so the pipeline continues. That matters when a later stage wants to transform the single result, or when the fold is one step in a composed gatherer (composition is covered below):

```java
String label = Stream.of(1, 2, 3, 4, 5)
        .gather(Gatherers.fold(() -> 0, Integer::sum))
        .map(sum -> "total=" + sum)
        .findFirst().orElseThrow();
// "total=15"
```

The javadoc positions `fold` for order-dependent reductions where no combiner can be written — it is explicitly sequential, and that is a feature, not a limitation: some reductions are only correct in encounter order.

Three behaviors to internalize, all verified by running them:

1. **On an empty stream, `fold` still emits once — the initial value.** `Stream.<Integer>empty().gather(Gatherers.fold(() -> 0, Integer::sum))` is `[0]`. `scan` on the same empty stream is `[]`. If your code treats "one element out" as "input was non-empty", `fold` will mislead you.
2. **`scan` never emits the initial value itself.** `Stream.of(1, 2).gather(Gatherers.scan(() -> 100, Integer::sum))` is `[101, 103]`, not `[100, 101, 103]`. Output count always equals input count.
3. **`fold` consumes the entire stream before emitting anything.** I put `fold` behind `limit(1)` over 100 elements and counted: all 100 were pulled. `limit` cannot save a `fold`, because the single result only exists at end-of-stream. The corollary is absolute: never put `fold` on an infinite stream — its only push happens in the finisher, which runs at end-of-stream, so on an infinite stream it never emits. `scan` and the windows, by contrast, emit incrementally and compose fine with `limit` on infinite sources, as the first example showed.

One trap lives in `scan` specifically. The scanner function receives the current accumulation and returns the next one — and `scan` pushes *whatever object you return*. If your function mutates a shared collection and returns the same instance every time, every downstream element is the same object, and by the time you read the results they all show the final state:

```java
// BUG: every emitted element is the same ArrayList instance.
var aliased = Stream.of(1, 2, 3)
        .gather(Gatherers.scan(ArrayList<Integer>::new,
                (list, e) -> { list.add(e); return list; }))
        .toList();
// [[1, 2, 3], [1, 2, 3], [1, 2, 3]] — three references, one object
```

I verified with identity checks that all three elements are the same instance. The fix is to return a new value at each step:

```java
var fixed = Stream.of(1, 2, 3)
        .gather(Gatherers.scan(ArrayList<Integer>::new, (list, e) -> {
            var next = new ArrayList<>(list);
            next.add(e);
            return next;
        }))
        .toList();
// [[1], [1, 2], [1, 2, 3]] — three distinct lists
```

With immutable accumulations (integers, strings, records) this cannot happen, which is one more reason to prefer them as scan state. `fold` has the same aliasing shape in principle, but since it emits only once, at the end, the shared-mutation hazard rarely bites there.

## Built-in 3: mapConcurrent — bounded concurrency that keeps order

`mapConcurrent(maxConcurrency, mapper)` applies a function to each element concurrently on virtual threads, with at most `maxConcurrency` tasks in flight, and preserves encounter order in the output. It is the gatherer for the "call a slow service for each element, but don't fire all thousand at once" shape:

```java
// download() is a blocking per-URL fetch; at most 8 run at once.
List<String> pages = urls.stream()
        .gather(Gatherers.mapConcurrent(8, this::download))
        .toList();
```

Two properties matter, and both held in my runs. First, **ordering**: with 12 inputs and a 20ms blocking mapper, the results came back in exact input order every time — concurrency is internal; the output stream is ordered. Second, **the bound is real**: with `maxConcurrency` set to 3, the peak in-flight count stayed at or below 3 while remaining above 1 — genuinely concurrent, genuinely bounded. The concurrency comes from the gatherer itself, on a plain sequential stream — you do not call `.parallel()`.

Exceptions propagate out of the terminal operation. In my run, a mapper throwing `IllegalStateException("boom")` surfaced as that same `IllegalStateException`, unwrapped — the JDK implementation rethrows unchecked causes as-is and wraps checked causes in a `RuntimeException`, cancelling the remaining tasks. `mapConcurrent(0, ...)` throws `IllegalArgumentException` at the factory call, and a `null` mapper throws `NullPointerException`.

When to reach for it: per-element blocking I/O with an explicit in-flight bound, where order matters. It is not a general replacement for `parallel()`: parallel streams split work across the shared common pool, while `mapConcurrent` bounds the work and runs it on virtual threads, which is the right shape when the mapper blocks on I/O. I ran no throughput benchmark for this post, so take no performance claim from it — choose on semantics (bounded, ordered, blocking-friendly) and measure your own workload.

## Writing your own: the four functions

Built-ins cover the common cases. The custom API is for everything else, and it has exactly four pieces. Only one is mandatory:

| Piece | Type | Role |
|---|---|---|
| Initializer | `Supplier<A>` | Creates fresh mutable state for one evaluation. |
| Integrator | `Integrator<A, T, R>` | Consumes one element; may push zero or more downstream. **Required.** |
| Combiner | `BinaryOperator<A>` | Merges two states for parallel evaluation. Omit it and the gatherer runs sequentially. |
| Finisher | `BiConsumer<A, Downstream<R>>` | Runs at end-of-stream; may flush remaining state. |

The integrator's signature is the whole contract:

```java
boolean integrate(A state, T element, Downstream<? super R> downstream);
```

Return `true` to keep consuming, `false` to stop — returning `false` is interpreted exactly as if the input had ended, and the finisher still runs (verified below). `downstream.push(element)` returns `false` when the next stage wants no more elements; pushing after that is silently dropped, never an exception. The idiomatic integrator returns `downstream.push(...)`'s result, so downstream rejection propagates instead of being ignored.

`Gatherer.ofSequential(...)` builds a gatherer with the default combiner, which the javadoc defines as sequential-only. `Gatherer.of(...)` with an explicit combiner builds a parallel-capable one. Start with `ofSequential` unless you have a combiner in mind — most order-dependent operations do not, and a sequential gatherer inside a parallel pipeline still produces correct, ordered results (verified in the parallel section).

### Example: distinctBy

JEP 485's motivating example: `distinct` uses object equality, but suppose you want distinctness by a key — at most one string of each length. That is a stateful one-to-many operation with no built-in, and it is about ten lines as a gatherer:

```java
public static <T, K> Gatherer<T, ?, T> distinctBy(Function<? super T, ? extends K> keyExtractor) {
    class State {
        Set<K> seen = new HashSet<>();
    }
    return Gatherer.<T, State, T>ofSequential(
            State::new,
            Gatherer.Integrator.ofGreedy((state, element, downstream) -> {
                if (state.seen.add(keyExtractor.apply(element))) {
                    return downstream.push(element);
                }
                return true;
            }));
}
```

```java
List<String> result = Stream.of("foo", "bar", "baz", "quux")
        .gather(distinctBy(String::length))
        .toList();
// [foo, quux]
```

Trace: `foo` (length 3) is new, pushed. `bar` (length 3) is already seen, dropped. `baz` (length 3) is already seen, dropped. `quux` (length 4) is new, pushed. The state lives in the initializer, so every pipeline evaluation gets a fresh `Set` — the same gatherer instance reused across two pipelines gives correct results in both (verified; the trap version that captures the `Set` outside the initializer is in the traps section).

Note `Integrator.ofGreedy`. A Greedy integrator promises it never short-circuits on its own decision: it consumes all input and only relays downstream rejection. `distinctBy` qualifies — it returns `true` when dropping an element and `downstream.push(...)`'s result when pushing. The promise exists so the library can optimize evaluation, and breaking it has observable consequences, covered in trap 1.

### Example: a short-circuiting gatherer

An integrator that returns `false` stops the pipeline early, which is how you turn an infinite stream finite or stop scanning at the first match. This gatherer emits elements until and including the first one matching a predicate:

```java
public static <T> Gatherer<T, ?, T> takeUntilInclusive(Predicate<? super T> stop) {
    return Gatherer.ofSequential(
            (unused, element, downstream) -> downstream.push(element) && !stop.test(element));
}
```

```java
List<Integer> result = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
        .gather(takeUntilInclusive(n -> n >= 5))
        .toList();
// [1, 2, 3, 4, 5]
```

Measured with a counting source: exactly 5 elements were pulled. The remaining 5 were never consumed. This is the gatherer equivalent of `takeWhile`, except inclusive — the matching element is emitted before stopping — and it works unchanged on infinite streams, where `takeWhile`-style logic over a `Collector` cannot exist at all.

### Composition: andThen

Gatherers compose with `andThen`, and the composition is semantically equivalent to chaining `.gather()` calls. A fixed window followed by a per-window sum, written both ways, gives the same `[6, 15, 15]` on `1..8` (verified):

```java
var composed = Gatherers.<Integer>windowFixed(3)
        .andThen(mapGatherer(w -> w.stream().mapToInt(Integer::intValue).sum()));

List<Integer> sums = Stream.of(1, 2, 3, 4, 5, 6, 7, 8).gather(composed).toList();
// [6, 15, 15]
```

`mapGatherer` here is the stateless map-as-gatherer from the `Gatherer` javadoc — one integrator that pushes `mapper.apply(element)`:

```java
public static <T, R> Gatherer<T, ?, R> mapGatherer(Function<? super T, ? extends R> mapper) {
    return Gatherer.of((unused, element, downstream) -> downstream.push(mapper.apply(element)));
}
```

Composition is mostly useful when you build reusable gatherer libraries; in a single pipeline, chained `.gather()` calls read just as well.

## Three traps, each measured

### Trap 1: declaring a short-circuiting integrator Greedy

`ofGreedy` is a promise, and the library trusts it: the javadoc says a Greedy integrator "can be assumed not to short-circuit, and the return value of invoking integrate does not need to be inspected." I ran the identical stop-after-2 logic both ways over 100 elements. Non-greedy: result `[1, 2]`, 2 elements consumed. Greedy:

```java
// BROKEN: identical logic, declared Greedy. The false return is ignored.
Gatherer.<Integer, Integer>ofSequential(
        Gatherer.Integrator.ofGreedy((unused, e, downstream) -> {
            downstream.push(e);
            return e < 2; // library may ignore this — and does
        }))
// result: all 100 elements emitted, all 100 consumed
```

The short-circuit silently did nothing. The rule is simple: use `ofGreedy` only when the integrator's `false` can only ever come from `downstream.push(...)` returning `false`. If your integrator ever decides to stop on its own — a predicate, a count, a sentinel — use the plain integrator form. A Greedy integrator behind a downstream `limit` still stops correctly (verified: 3 consumed with `limit(3)`), because downstream rejection is handled by the framework; it is only *your own* short-circuit decision that gets ignored.

### Trap 2: capturing mutable state outside the initializer

State must be created by the initializer, per evaluation. Capturing a mutable collection in the lambda instead looks identical on the first run and corrupts every run after it:

```java
// BROKEN: 'seen' is shared by every pipeline that uses this gatherer.
public static <T, K> Gatherer<T, ?, T> distinctByBuggy(Function<? super T, ? extends K> keyExtractor) {
    Set<K> seen = new HashSet<>();
    return Gatherer.ofSequential((unused, element, downstream) -> {
        if (seen.add(keyExtractor.apply(element))) {
            return downstream.push(element);
        }
        return true;
    });
}
```

First run on `["foo", "bar", "baz", "quux"]`: `[foo, quux]`, correct. Second run, same gatherer instance, same input: `[]` — every key was already seen. The correct version from the previous section returns `[foo, quux]` on both runs, because `State::new` runs fresh for each evaluation. The javadoc states the underlying rule more generally: implementations must not capture, retain, or expose state or downstream references beyond the invocation that received them. If your gatherer has mutable state, it goes in the state object, full stop. This also rules out the parallel hazard: partitions each get their own state from the initializer, but a captured collection would be shared across threads with no synchronization.

### Trap 3: assuming the finisher only runs at natural end — or forgetting it entirely

Two finisher facts, both verified. First, a gatherer that buffers elements and has no finisher silently drops its tail. This hand-rolled fixed window, integrator only:

```java
// Drops the tail: [1..8] with size 3 yields [[1, 2, 3], [4, 5, 6]] — [7, 8] lost.
public static Gatherer<Integer, ?, List<Integer>> windowFixedNoFinisher(int size) {
    class State {
        List<Integer> buffer = new ArrayList<>();
    }
    return Gatherer.<Integer, State, List<Integer>>ofSequential(
            State::new,
            Gatherer.Integrator.ofGreedy((state, element, downstream) -> {
                state.buffer.add(element);
                if (state.buffer.size() == size) {
                    List<Integer> full = List.copyOf(state.buffer);
                    state.buffer.clear();
                    return downstream.push(full);
                }
                return true;
            }));
}
```

Add the finisher flush and the tail appears: `[[1, 2, 3], [4, 5, 6], [7, 8]]`. The JDK's own `windowFixed` does exactly this — its finisher pushes the partial last window. Any gatherer that buffers must flush in the finisher, or the last partial batch is gone without a trace.

Second, the finisher runs *even when the integrator short-circuits*. I built a gatherer whose finisher pushes a `"DONE"` marker: with a match at element 2, the output was `[1, 2, DONE]` — the finisher ran after the early stop, just as it did (`[1, 2, 3, DONE]`) when the stream ran to natural end. The JEP's own evaluation sketch shows the same order: integrate until the integrator returns `false`, then invoke the finisher. So write finishers that are safe to run after an early stop: guard the flush against already-emitted or empty state, and mirror the JDK's window finishers by checking `!downstream.isRejecting()` before pushing a final element into a downstream that has already said no more:

```java
(state, downstream) -> {
    if (!state.buffer.isEmpty() && !downstream.isRejecting()) {
        downstream.push(List.copyOf(state.buffer));
    }
}
```

## Parallel evaluation: the combiner earns its keep

A gatherer with the default combiner may only be evaluated sequentially — but "sequentially" describes the gatherer, not the whole pipeline. A sequential gatherer inside a parallel pipeline still produces correct, encounter-ordered results; the library extracts parallelism from the upstream and downstream operations around it. I verified both: `distinctBy` on a parallel stream of `["foo", "bar", "baz", "quux"]` yields `[foo, quux]`, and `windowFixed(3)` on a parallel stream of `1..8` yields the same three windows in order as the sequential version. Order-dependent operations — windows, scans — should stay sequential. Do not force a combiner onto an operation whose merge is not well-defined.

When a correct merge *does* exist, `Gatherer.of(...)` with a combiner parallelizes the gatherer itself: each partition gets its own state, partitions integrate independently, states merge pairwise through the combiner, and the finisher runs once on the joined state. The maximum-element gatherer is the canonical example — the merge is just "keep the larger best":

```java
public static <T> Gatherer<T, ?, T> maxGatherer(Comparator<? super T> comparator) {
    class State {
        T best;
        boolean present;
    }
    return Gatherer.<T, State, T>of(
            State::new,
            Gatherer.Integrator.ofGreedy((state, element, downstream) -> {
                if (!state.present || comparator.compare(element, state.best) > 0) {
                    state.best = element;
                    state.present = true;
                }
                return true;
            }),
            (left, right) -> {
                if (!left.present) {
                    return right;
                }
                if (!right.present) {
                    return left;
                }
                return comparator.compare(left.best, right.best) >= 0 ? left : right;
            },
            (state, downstream) -> {
                if (state.present) {
                    downstream.push(state.best);
                }
            });
}
```

Verified three ways: sequential `[3, 1, 4, 1, 5]` yields `[5]`; parallel `1..1000` yields `[1000]`; an empty stream yields `[]` (the finisher's `present` check is what keeps the empty case from pushing a phantom `null`). The combiner's correctness is entirely on you — a combiner that merges wrong produces wrong parallel results with no error, and only a sequential-vs-parallel comparison test catches it. Write that test before you ship a parallel gatherer.

## When to reach for gatherers

- **You need an intermediate operation that does not exist.** Windows, running scans, distinct-by-key, dedupe of consecutive duplicates, chunking on a predicate. If the JDK's five built-ins cover it, use them; if not, a ten-line custom gatherer beats a loop that breaks the pipeline.
- **The stream may be infinite, or you need to stop early.** Gatherers short-circuit; collectors cannot. Any logic that must react to "enough" mid-stream belongs in a gatherer (or a plain loop), never in a collector.
- **The result must stay a stream.** `fold` exists for the case where a reduction is one step among several, not the end of the pipeline.
- **You need bounded concurrent mapping with order preserved.** `mapConcurrent` is the only built-in that does all three at once.

And when not to: if you only need a final summary value, `collect()` is simpler and better understood by every reader. If the logic is a one-off with complex early exits and side effects, a plain `for` loop is clearer than a clever gatherer — gatherers earn their keep through reuse and composition, not novelty. And if the operation is inherently a terminal aggregation with a well-defined combiner, a `Collector` is still the right tool; gatherers extend the intermediate side of the API, they do not replace the terminal side.

## What was verified

Primary sources first: the JEP history and design are from [JEP 485](https://openjdk.org/jeps/485) (previewed by JEP 461 in JDK 22, re-previewed by JEP 473 in JDK 23, finalized in JDK 24 without change); the per-operation semantics — window edge cases, unmodifiable windows, `fold`/`scan` emission rules, `mapConcurrent` ordering and virtual threads, Greedy and combiner contracts — are from the `Gatherer`/`Gatherers` javadoc and implementation in the JDK 25 source bundle, and the absence of `gather` on primitive streams was checked directly against the JDK 25 API.

Then runtime: **39 tests, run twice, all passing, on Temurin JDK 25.0.5 with JUnit 6.0.3**, compiled with `--release 25` and no preview flag. That includes the three seeded property checks (200 random inputs each for `windowFixed`, `windowSliding`, and `scan`), the consumed-element counts behind every laziness claim (6 for the infinite-window example, 100 for `fold` behind `limit`, 5 for `takeUntilInclusive`, 2 vs 100 for the Greedy trap), the concurrency bound and ordering for `mapConcurrent`, and every trace printed in this post — each trace matches an executed run. Two things were *not* run and are stated as such: no throughput benchmark for `mapConcurrent`, and no infinite-stream `fold` run (its never-emits behavior follows from the finisher-only emission in the JDK source, and running it would hang by design).
