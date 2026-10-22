---
title: 'HashMap Internals: The Questions That Expose Shallow Knowledge'
date: 2026-10-13T07:00:00-04:00
draft: false
ShowToc: true
description: >-
  "How does HashMap work?" is the first question on every backend interview
  list — and most answers stop at "array of linked lists." The follow-ups that
  separate deep from shallow: the hash spread, treeify thresholds, the resize
  split trick, and why ConcurrentHashMap counts with cells instead of a long.
tags:
  - java
  - interview
  - backend
  - algorithms
categories: article
keywords:
  - hashmap internals java
  - java hashmap treeify
  - hashmap resize
  - concurrenthashmap counter cells
  - java interview hashmap
---

"How does HashMap work?" opens half of all Java interviews. Most candidates answer "array of linked lists, O(1) lookup" — and then the follow-ups start, and the room gets quiet. The interviewers aren't testing whether you've memorized the structure. They're testing whether you know *why* it's built the way it is. Here are the questions that expose shallow knowledge, with the answers that don't.

## Question 1: Why does HashMap mangle your hashCode?

Everyone knows `hashCode()` feeds the map. Almost nobody asks what HashMap does to it first:

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

That `^ (h >>> 16)` is the **hash spread** — it XORs the top 16 bits down into the bottom 16. Why? Because the bucket index is computed as `(n - 1) & hash`, and with a power-of-two table size, that expression only looks at the **low bits** of the hash. If your keys differ only in their high bits — entirely plausible with auto-generated IDs or poorly distributed `hashCode()` implementations — they'd collide in every small table. Spreading mixes the high bits down where they actually get used.

The interview answer: *"The index uses only low bits, so HashMap perturbs the hash to make high-bit differences affect bucket placement. It's defense against bad hash functions, not a replacement for writing a good one."*

Bonus follow-up: what if the key is `null`? HashMap allows exactly one null key — and there is no special case in the lookup path. `hash(null)` simply returns 0, so the null key hashes to 0 and lives in bucket 0 like any other key (which is also why only one fits). The only null-specific behavior anywhere is inside `hash()` itself. `ConcurrentHashMap` throws `NullPointerException` instead — no nulls, period.

## Question 2: Why is the capacity always a power of two?

Because `(n - 1) & hash` is a fast modulo — but only when `n` is a power of two. With `n = 16`, `n - 1 = 15 = 0b1111`, and AND-ing masks out everything but the low 4 bits: exactly `hash % 16` for non-negative hashes, in one CPU cycle instead of a division.

This is why the constructor rounds your requested capacity up to the next power of two (`tableSizeFor`), and why resize doubles. The power-of-two invariant isn't trivia — it's load-bearing for the resize trick in Question 4.

## Question 3: When does a bucket become a tree — and when doesn't it?

Java 8+ converts a bucket's linked list into a red-black tree once the chain grows **past** `TREEIFY_THRESHOLD` (8) — and the precise trigger is a classic follow-up trap. In `putVal`, the new node is appended first, and only then does the check run: `if (binCount >= TREEIFY_THRESHOLD - 1) treeifyBin(tab, hash)`. Since `binCount` counted the nodes already in the chain, the check fires on the insert that makes the chain **9 nodes** long. It doesn't treeify when the chain "hits 8" — it treeifies when the chain exceeds it. But the number everyone forgets is `MIN_TREEIFY_CAPACITY` (64):

> If the table has fewer than 64 buckets, HashMap **resizes instead of treeifying**.

The reasoning is sound: one crowded bucket in a tiny table probably means the table is too small, not that the keys are pathological. Spread the entries across more buckets first; pay for tree nodes only when crowding survives a 64-bucket table. On the way back down, keep two different mechanisms separate — conflating them is the shallow answer. `UNTREEIFY_THRESHOLD` (6) is documented as the threshold for untreeifying a bin **during a resize split**: when a resize splits a tree bin into halves of 6 or fewer, each half converts back to a plain list. Removal is not a count check at all: `removeTreeNode` decides from the tree's shape — a structural heuristic on the root and its children, not "6 nodes left". Trees cost memory; the thresholds exist to reclaim it without flip-flopping.

The numbers to have on instant recall: **8 up (fires at 9), 6 down (on resize splits), 64 minimum**. The interview answer: *"Treeify once a chain grows past 8 nodes — the `putVal` check fires on the insert that makes it 9 — for O(log n) worst case in a hostile bucket, but only if the table is already 64+ buckets; otherwise resize. Untreeify happens when a resize split leaves a half with 6 or fewer nodes; on removal the tree reverts structurally, not by count."*

## Question 4: What actually happens during resize?

The shallow answer: "capacity doubles and entries are rehashed." The deep answer: **no hash is recomputed**. Because the capacity doubles (power of two, see Question 2), each old bucket's chain splits cleanly in two based on a single bit:

```java
// during resize, for each node in old bucket j:
if ((e.hash & oldCap) == 0)
    // goes to the "low" list — same index j
else
    // goes to the "high" list — index j + oldCap
```

One bit test per node, zero recomputation, and the relative order within each resulting list is preserved. A single resize is O(n) — every entry is touched once — but amortized across the puts that triggered it, it's O(1) per put, the same argument as `ArrayList` growth.

The resize trigger: `size > capacity * loadFactor` (default 0.75), checked after each insert. The follow-up that ends interviews: *"Why 0.75?"* — it's the engineering tradeoff between space waste and collision probability that the JDK authors measured, not a law of nature. Saying "it's tunable via the constructor and 0.75 is the measured sweet spot" beats reciting the number.

## Question 5: Why is iteration order "undefined"?

Because it's a side effect of three things you don't control: the spread hash values, the power-of-two table size, and the resize history. Insert the same keys in the same order into two maps with different initial capacities and you can get different iteration orders. The interview trap is code that *accidentally* depends on HashMap order and passes tests for months — then breaks after a resize threshold is crossed in production.

The answer: *"Order is an accident of hashing, capacity, and resize history. If you need insertion order, that's what LinkedHashMap is for — it maintains a separate doubly-linked list across entries for exactly this."*

## Question 6: Why does ConcurrentHashMap count with cells instead of a long?

`ConcurrentHashMap.size()` can't just keep a `volatile long` counter — every `put` from every thread would CAS on the same cache line and serialize. Instead it borrows the `LongAdder` design: a `baseCount` plus a lazily-allocated array of `CounterCell`s. Threads hash to different cells and CAS on their own cell; `size()` sums them up. Contention spreads instead of serializing.

This is the question that separates "I know ConcurrentHashMap is thread-safe" from "I know *how* it's thread-safe without locking the world." Related: its hash spread additionally clears the sign bit (`& 0x7fffffff`) because negative hash values mark special nodes (forwarding nodes during resize, tree bins) — the sign bit is protocol, not data.

## The 2-minute interview answer

> "HashMap is an array of bins sized as a power of two, so the index is `(n-1) & hash` — one AND instead of a modulo. The key's hashCode is first spread with `h ^ (h >>> 16)` because the index only uses low bits, and high-bit differences would otherwise always collide. Collisions chain as linked lists; once a chain grows past 8 nodes (the treeify check fires on the 9th insert) the bin treeifies into a red-black tree for O(log n) worst case — but only if the table is already 64 buckets, otherwise it resizes — and a tree bin converts back to a list when a resize split leaves a half of 6 or fewer. Resize doubles the table and splits each bin by testing one bit of the stored hash — no recomputation. Default load factor 0.75 balances space against collisions. Iteration order is undefined because it depends on hashes, capacity, and resize history — use LinkedHashMap for order. And ConcurrentHashMap avoids a contended size counter with LongAdder-style counter cells."

## Cheat sheet

| Fact | Value |
|---|---|
| Default capacity / load factor | 16 / 0.75 |
| Index computation | `(n - 1) & hash` — no modulo |
| Hash spread | `(h = key.hashCode()) ^ (h >>> 16)` |
| Treeify / untreeify / min capacity | 8 (fires at 9 nodes) / 6 (resize splits) / 64 |
| Resize | Doubles; splits bins on `(e.hash & oldCap)` — no rehash |
| Null key | One allowed — `hash(null)` is 0, so it lives in bucket 0 (CHM: NPE) |
| Iteration order | Undefined — capacity + resize history dependent |
| CHM size counting | `baseCount` + `CounterCell[]`, LongAdder-style |

Know the numbers, but more importantly know the *whys*: the spread exists because the index ignores high bits, the power of two exists to make resize a bit-test, and 64 exists because treeifying a tiny table is treating the symptom. That's what the follow-ups are really asking.
