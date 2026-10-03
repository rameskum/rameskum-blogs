---
title: 'Top-K Everything: Heap Questions That Actually Show Up in Java Interviews'
date: 2026-10-02T12:45:00-04:00
draft: false
ShowToc: true
description: >-
  "Find the k largest elements." If your answer starts with "sort the array,"
  you're leaving a faster, streaming-friendly solution on the table when k is
  small relative to n. Four working Java patterns —
  k largest, top-k frequent, kth largest in a stream, sliding-window median —
  the complexity argument that separates a correct answer from a strong one, and the PriorityQueue
  traps that sink candidates.
tags:
  - interview
  - algorithms
  - java
categories: article
keywords:
  - top k elements java priorityqueue
  - kth largest stream java
  - sliding window median two heaps
  - java heap interview questions
---

"Find the k largest elements in this array."

Most candidates reach for `Arrays.sort()` and stop. It works. It's O(n log n). A stronger answer reaches for the heap: sorting processes all n elements in O(n log n) time, while a bounded heap takes O(n log k) and maintains only k candidates — preferable when k is small relative to n, or when the input arrives as a stream. For 1 ≤ k ≤ n, the heap uses O(k) auxiliary space, and the returned result uses O(k) output space. This post builds four common heap patterns that cover a wide range of top-k questions asked in Java interviews, with working code, the complexity math, and the `PriorityQueue` traps that quietly sink candidates.

## The one idea that unlocks all four patterns

Java's `PriorityQueue` is a **min-heap by default**: the head of the queue is always the *least* element under the ordering. That single fact gives you the universal top-k pattern:

> Keep a min-heap of at most k elements, assuming 1 ≤ k ≤ n. For each candidate, add it; if the heap grows past k, evict the head. When you're done, the heap holds the k *largest* elements — because whenever the heap exceeds `k`, the smallest retained candidate is evicted.

```java
// The shape every top-k solution takes:
PriorityQueue<T> minHeap = new PriorityQueue<>(/* ordered so the "worst keeper" is the head */);
for (T x : candidates) {
    minHeap.offer(x);
    if (minHeap.size() > k) {
        minHeap.poll(); // evict the smallest — it can never be in the top k
    }
}
```

Everything below is this pattern with a different definition of "worst keeper."

## Know your tool: six PriorityQueue facts worth knowing cold

Straight from the `java.util.PriorityQueue` Javadoc — worth knowing cold, because they make natural follow-up questions:

1. **Min-heap by default.** Head = least element. Pass a `Comparator` (e.g. `Comparator.reverseOrder()`) for a max-heap.
2. **The iterator is not sorted.** Iterating a `PriorityQueue` does *not* visit elements in priority order — the heap is only partially ordered internally. To drain in order, `poll()` repeatedly. Candidates who return "the heap's contents" as a sorted result fail here.
3. **`offer`/`poll` are O(log n); `contains`/`remove(Object)` are linear; `peek`/`element`/`size` are O(1).** The heap gives you cheap head access, not cheap search. If your design calls `contains()` in a loop, rethink it.
4. **Not thread-safe.** The JDK wording is precise: multiple threads should not access a `PriorityQueue` concurrently *if any of them modifies it*. If multiple threads need concurrent access with mutation, use appropriate synchronization or a thread-safe alternative such as `PriorityBlockingQueue` — which adds blocking queue semantics, so reach for it when those semantics are what you want, not as a blanket replacement.
5. **No nulls.** `offer(null)` throws `NullPointerException`.
6. **Mutating an element's priority after insertion does not re-heapify.** Changing a field used by the comparator after insertion does not cause the `PriorityQueue` to re-heapify — remove and reinsert the element if its priority changes. Heap entries should be effectively immutable while they're in the queue.

## Pattern 1: k largest in an unsorted array

The canonical opener. n up to 10^6, k = 10.

```java
import java.util.PriorityQueue;

public class TopK {

    public static int[] kLargest(int[] nums, int k) {
        if (k <= 0) {
            return new int[0];
        }
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        for (int n : nums) {
            minHeap.offer(n);
            if (minHeap.size() > k) {
                minHeap.poll(); // drop the smallest survivor
            }
        }
        // Draining with poll() yields them in ascending order.
        // Iterating the queue directly would NOT be sorted.
        int[] out = new int[minHeap.size()];
        int i = 0;
        while (!minHeap.isEmpty()) {
            out[i++] = minHeap.poll();
        }
        return out;
    }
}
```

Edge cases to say out loud in the interview: k ≤ 0 returns empty; k ≥ n returns everything (the heap just never evicts). For **1 ≤ k ≤ n**: time is **O(n log k)** — each element performs an `offer()` and, once the heap is full, at most one `poll()`; the heap holds at most k + 1 elements during processing. Auxiliary heap space is **O(k)**. The implementation additionally handles k ≤ 0 (returns empty) and k > n (returns everything), as documented above.

### The "why not sort?" answer

This is the part interviewers actually score. Say it like this:

> Sorting processes all n elements and takes O(n log n) time. A bounded heap takes O(n log k) time and maintains only k candidates, making it preferable when k is small relative to n or when the input arrives as a stream. At the rough asymptotic scale, n log₂ n ≈ 20M versus n log₂ k ≈ 3.3M — roughly 6× apart on this theoretical scale (actual comparison counts depend on the sort and heap implementations), and it never holds more than 11 elements. When k is small relative to n, sorting is doing work the problem never asked for.

That paragraph is the difference between "correct" and "strong hire" on this question.

## Pattern 2: top-k frequent elements

Now "worst keeper" means *lowest frequency*. Count first, then heap on the counts (LeetCode 347 shape).

This method assumes **1 ≤ k ≤ number of distinct elements** — the usual interview-problem constraint — and enforces it, because without it the output array would be the wrong size when k exceeds the distinct count:

```java
import java.util.*;

public class TopKFrequent {

    public static int[] topKFrequent(int[] nums, int k) {
        Map<Integer, Integer> freq = new HashMap<>();
        for (int n : nums) {
            freq.merge(n, 1, Integer::sum);
        }
        if (k <= 0 || k > freq.size()) {
            throw new IllegalArgumentException(
                    "k must satisfy 1 <= k <= number of distinct elements, got " + k);
        }
        // Min-heap ordered by frequency: the head is the least frequent keeper.
        PriorityQueue<Integer> minHeap =
                new PriorityQueue<>(Comparator.comparingInt(freq::get));
        for (int key : freq.keySet()) {
            minHeap.offer(key);
            if (minHeap.size() > k) {
                minHeap.poll();
            }
        }
        int[] out = new int[k];
        int i = 0;
        while (!minHeap.isEmpty()) {
            out[i++] = minHeap.poll();
        }
        return out;
    }
}
```

Complexity: let m be the number of distinct values. Counting takes O(n), the heap takes O(m log k), for **O(n + m log k)** total time. Space is **O(m + k)** for the map and heap, which is O(n) in the worst case. The follow-up to expect: "what if frequencies tie?" — answer honestly: ties break arbitrarily. For deterministic tie-breaking, add a secondary comparator that matches the problem's required tie rule — for example, `Comparator.comparingInt(freq::get).thenComparingInt(x -> x)` makes larger values survive ties (the smaller value becomes the evictable head), so choose the secondary ordering deliberately rather than copying this one.

## Pattern 3: kth largest in a stream

The stream variant (LeetCode 703): numbers arrive one at a time, and `add()` must return the kth largest so far. Same pattern, but the heap persists between calls:

```java
import java.util.PriorityQueue;

public class KthLargest {
    private final int k;
    private final PriorityQueue<Integer> minHeap = new PriorityQueue<>();

    public KthLargest(int k, int[] nums) {
        if (k < 1) {
            throw new IllegalArgumentException("k must be >= 1, got " + k);
        }
        if (nums.length < k) {
            throw new IllegalArgumentException(
                    "nums must contain at least k elements, got " + nums.length);
        }
        this.k = k;
        for (int n : nums) {
            add(n);
        }
    }

    public int add(int val) {
        minHeap.offer(val);
        if (minHeap.size() > k) {
            minHeap.poll();
        }
        return minHeap.peek(); // the kth largest is the smallest of the top k
    }
}
```

This class assumes **k ≥ 1** and that the initial array holds at least k elements — both enforced in the constructor. Without the second check, `new KthLargest(5, new int[]{1, 2})` followed by `add(3)` would return 1: a "5th largest" that doesn't exist. This matches the LeetCode 703 contract, where the kth largest is always defined when `add()` is called. `add()` is **O(log k)**. The constructor is O(n log k) as written.

A good bonus talking point, if you know it well: `new PriorityQueue<>(collection)` bulk-loads with a linear-time heapify, O(n) instead of n individual inserts — but note it's a *different* initialization strategy, not a drop-in optimization here. Heapifying the whole input takes O(n) memory, which breaks the bounded O(k)-space property this streaming design exists to provide.

And the trap to name proactively: `peek()` returns `null` on an empty queue (whereas `element()` throws). Here the queue is non-empty after every `add` for k ≥ 1, but saying it shows you've read the Javadoc, not just the tutorial.

## Pattern 4: sliding-window median — the two-heap technique

This is the "hard" one interviewers graduate to (LeetCode 480). One heap can't do it, because a median needs the middle, not an extreme. The technique: **two heaps** — a max-heap `lo` for the lower half, a min-heap `hi` for the upper half — kept balanced so the median is always at the tops.

This method assumes **1 ≤ k ≤ nums.length** (a zero or oversized window has no meaningful median), enforced up front:

Invariants to state before coding (in terms of the *logical* sizes — the code tracks them in the `loSize` / `hiSize` fields because lazy deletion leaves stale entries in the heaps, so `heap.size()` can be larger than the logical size until `prune` runs):
- Every element in `lo` ≤ every element in `hi`.
- `lo` holds the extra element when the window size is odd: `loSize == hiSize` or `loSize == hiSize + 1`.
- Median = top of `lo` (odd window) or average of both tops (even window).

The wrinkle: removing the element that slides out of the window. Finding it in a heap is O(n), so instead we **lazily delete** — record it in a `delayed` map and prune stale tops when they surface:

```java
import java.util.*;

public class SlidingWindowMedian {

    public static double[] medianSlidingWindow(int[] nums, int k) {
        if (k < 1 || k > nums.length) {
            throw new IllegalArgumentException(
                    "k must satisfy 1 <= k <= nums.length, got " + k);
        }
        Window w = new Window(k);
        double[] out = new double[nums.length - k + 1];
        for (int i = 0; i < nums.length; i++) {
            w.add(nums[i]);
            if (i >= k) {
                w.remove(nums[i - k]);
            }
            if (i >= k - 1) {
                out[i - k + 1] = w.median();
            }
        }
        return out;
    }

    static final class Window {
        private final PriorityQueue<Integer> lo =
                new PriorityQueue<>(Comparator.reverseOrder()); // max-heap: lower half
        private final PriorityQueue<Integer> hi = new PriorityQueue<>(); // min-heap: upper half
        private final Map<Integer, Integer> delayed = new HashMap<>();
        private final int k;
        private int loSize, hiSize; // logical sizes, excluding delayed removals

        Window(int k) { this.k = k; }

        void add(int num) {
            if (lo.isEmpty() || num <= lo.peek()) {
                lo.offer(num);
                loSize++;
            } else {
                hi.offer(num);
                hiSize++;
            }
            rebalance();
        }

        void remove(int num) {
            delayed.merge(num, 1, Integer::sum);
            if (num <= lo.peek()) {
                loSize--;
            } else {
                hiSize--;
            }
            prune(lo);
            prune(hi);
            rebalance();
        }

        double median() {
            prune(lo);
            prune(hi);
            return (k % 2 == 1)
                    ? lo.peek()
                    : ((double) lo.peek() + hi.peek()) / 2.0;
        }

        private void rebalance() {
            if (loSize > hiSize + 1) {
                hi.offer(lo.poll());
                loSize--;
                hiSize++;
                prune(lo);
            } else if (loSize < hiSize) {
                lo.offer(hi.poll());
                loSize++;
                hiSize--;
                prune(hi);
            }
        }

        private void prune(PriorityQueue<Integer> heap) {
            while (!heap.isEmpty()) {
                int top = heap.peek();
                int count = delayed.getOrDefault(top, 0);
                if (count == 0) {
                    break;
                }
                if (count == 1) {
                    delayed.remove(top);
                } else {
                    delayed.put(top, count - 1);
                }
                heap.poll();
            }
        }
    }
}
```

Each add/remove is **O(log k) amortized**. `prune()` can discard several stale entries in one call, but every stale heap entry is removed at most once, so the total pruning work is amortized across the run. Overall time is **O(n log k)**.

Space is **O(n) in the worst case** because lazy deletion can leave stale entries buried inside the heaps until they reach the top. The logical window and the `delayed` map are O(k), but the physical heaps can accumulate O(n) stale entries — for example, with monotonically increasing input, evicted elements sink to the bottom of the max-heap and aren't pruned until they surface. A structure with eager deletion, such as a balanced tree or multiset, could hold O(k) space at the cost of different implementation details. The three things to narrate while writing it: (1) why two heaps — a single heap can only see one extreme; (2) why lazy deletion — `remove(Object)` on a heap is O(n), which would make the whole thing O(n·k); (3) why `lo` gets the extra element — so `median()` never has to think.

## Prove it works: tests

Interview code earns trust when it's tested. The patterns above are pure functions of their inputs, which makes them trivially testable:

```java
import java.util.Arrays;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class TopKTest {

    @Test
    void kLargestBasic() {
        assertArrayEquals(new int[]{7, 9, 10},
                TopK.kLargest(new int[]{3, 1, 10, 7, 9, 2}, 3));
    }

    @Test
    void kLargestEdges() {
        assertArrayEquals(new int[0], TopK.kLargest(new int[]{5, 1}, 0));
        // k > n returns everything, ascending
        assertArrayEquals(new int[]{1, 2, 3}, TopK.kLargest(new int[]{3, 1, 2}, 9));
    }

    @Test
    void kLargestDuplicatesAndFullRange() {
        assertArrayEquals(new int[]{5, 5, 5}, TopK.kLargest(new int[]{5, 5, 5}, 3));
        // k == n returns everything, ascending
        assertArrayEquals(new int[]{1, 2, 3}, TopK.kLargest(new int[]{3, 1, 2}, 3));
    }

    @Test
    void topKFrequentBasic() {
        int[] out = TopKFrequent.topKFrequent(new int[]{1, 1, 1, 2, 2, 3}, 2);
        Arrays.sort(out);
        assertArrayEquals(new int[]{1, 2}, out);
    }

    @Test
    void topKFrequentTies() {
        // 1 and 2 tie at frequency 2; both must be returned, order not promised
        int[] out = TopKFrequent.topKFrequent(new int[]{1, 1, 2, 2, 3}, 2);
        Arrays.sort(out);
        assertArrayEquals(new int[]{1, 2}, out);
    }

    @Test
    void kthLargestStream() {
        KthLargest kth = new KthLargest(3, new int[]{4, 5, 8, 2});
        assertEquals(4, kth.add(3));   // top 3: {8, 5, 4}
        assertEquals(5, kth.add(5));   // top 3: {8, 5, 5}
        assertEquals(5, kth.add(10));  // top 3: {10, 8, 5}
        assertEquals(8, kth.add(9));   // top 3: {10, 9, 8}
    }

    @Test
    void kthLargestDuplicates() {
        KthLargest kth = new KthLargest(2, new int[]{5, 5, 5});
        assertEquals(5, kth.add(5));   // heap: {5, 5}
        assertEquals(5, kth.add(1));   // 1 evicted immediately, heap: {5, 5}
    }

    @Test
    void slidingWindowMedian() {
        assertArrayEquals(new double[]{1.0, -1.0, -1.0, 3.0, 5.0, 6.0},
                SlidingWindowMedian.medianSlidingWindow(
                        new int[]{1, 3, -1, -3, 5, 3, 6, 7}, 3), 1e-9);
    }

    @Test
    void slidingWindowMedianEdges() {
        // k = 1: the median is each element itself
        assertArrayEquals(new double[]{3.0, 1.0, 2.0},
                SlidingWindowMedian.medianSlidingWindow(new int[]{3, 1, 2}, 1), 1e-9);
        // k = n: a single window over everything
        assertArrayEquals(new double[]{2.0},
                SlidingWindowMedian.medianSlidingWindow(new int[]{3, 1, 2}, 3), 1e-9);
        // extreme values: the (double) cast happens before addition, so no overflow
        assertArrayEquals(new double[]{2147483647.0},
                SlidingWindowMedian.medianSlidingWindow(
                        new int[]{Integer.MAX_VALUE, Integer.MAX_VALUE}, 2), 1e-9);
    }
}
```

Note the `topKFrequent` test sorts before asserting. That's deliberate: `PriorityQueue` maintains heap order according to the comparator, but its iterator does not guarantee sorted order. If priority order is required, repeatedly call `poll()`. A test that assumes iteration order is testing an accident, not a contract.

## The 2-minute interview answer

When the interviewer says "k largest," deliver this and then code pattern 1:

> "I'll keep a min-heap capped at k. Every element goes in; whenever the heap exceeds k, I evict its smallest element. Therefore, after processing all values, the heap contains the k largest values seen. That's O(n log k) time in one pass, O(k) space — versus O(n log n) for sorting, which does work the problem never asked for when k is small."

Then, unprompted, add: "and I'll drain with `poll()`, not iterate, because the iterator isn't sorted." That one sentence tells them you've been burned by — or at least read about — the real API.

## Cheat sheet

| Question | Heaps | Key idea | Complexity |
|---|---|---|---|
| k largest / smallest | 1 min-heap for largest (max-heap for smallest), size k | Evict the head past k | O(n log k), O(k) |
| Top-k frequent | 1 min-heap keyed by frequency | Count first, heap second | O(n + m log k), O(m + k) |
| kth largest in a stream | 1 persistent min-heap of size k | kth largest = heap head | O(log k) per add |
| Sliding-window median | max-heap + min-heap, lazy deletion | Balance halves; median at the tops | O(n log k) amortized time, O(n) worst-case space |
| Merge k sorted lists | 1 min-heap of size k over list heads | Always pop the global minimum | O(N log k), O(k) |

That last row is the bonus pattern — same machinery, and a natural "what would you do next?" answer when the interviewer wants more. (In the table, m = number of distinct values.)

## Takeaway

Top-k questions aren't four different problems; they're one pattern — a bounded heap whose head is the worst element you're willing to keep — applied with different orderings and different eviction triggers. Learn the shape, the O(n log k) argument, and the six `PriorityQueue` facts, and the entire family collapses into a single confident answer.
