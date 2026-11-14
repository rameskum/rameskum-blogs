---
title: 'Cyclic Sort: The O(n) Trick Behind "Find the Missing Number"'
date: 2026-11-13T23:00:00-05:00
draft: false
ShowToc: true
description: >-
  When the values live in a known range — 0..n or 1..n — the array is its own
  hash table: every value belongs at its own index. One placement loop, three
  interview problems (Missing Number, Find All Duplicates, First Missing
  Positive), each in O(n) time and O(1) extra space, with the worked traces and
  the duplicate guard that keeps the loop from spinning forever. All 12 tests
  run twice on JDK 25.
tags:
  - java
  - algorithms
  - interview
  - backend
categories: article
keywords:
  - cyclic sort java
  - first missing positive java
  - find all duplicates in array
  - missing number leetcode 268
  - in-place array interview pattern
---

Cyclic sort is the highest-ROI pattern you've never been taught by name. The entire technique is one invariant: **when the values come from a known range, every value belongs at its own index — so put it there.** Value `v` goes to index `v` (for ranges starting at 0) or index `v - 1` (for ranges starting at 1). After one placement pass, whatever sits at the wrong index *is* the answer: the missing number, the duplicate, the first gap.

The tell is the problem statement. Values in `[0, n]` or `[1, n]`, plus a demand for O(n) time and O(1) extra space, means the intended solution uses the array as its own hash table. Sorting is O(n log n) and a `HashSet` is O(n) space — both fail the constraints. Placement passes both. Three LeetCode staples fall to the same loop: Missing Number (268, Easy), Find All Duplicates (442, Medium), and First Missing Positive (41, Hard). Learn the loop once, and the Hard one is the same code with a range check bolted on.

## The pattern: place, don't sort

```java
static void swap(int[] nums, int i, int j) {
    int tmp = nums[i];
    nums[i] = nums[j];
    nums[j] = tmp;
}
```

The loop walks index `i` from left to right. At each step it looks at `nums[i]`, computes that value's home index, and swaps it home — *without* advancing `i`, because the swap just brought a new, unchecked value into position `i`. It advances only when the current value is already home, has no home (out of range), or its home is already occupied by an equal value. Then a final scan reads the answer off the mismatched positions.

**Why is this O(n) when there's a nested-looking swap loop?** Because every swap puts at least one element in its final home, and no swap ever moves a correctly placed element: a swap only targets a home slot whose current value differs from the incoming value, and a value already home is never swapped away. So there are at most `n` swaps and at most `n` advances — at most `2n` placement steps — plus the O(n) scan. I checked the bound empirically: across 2,000 random arrays, the swap count never exceeded `n` (worst observed: 25 swaps at `n = 46`). The cost is mutation: cyclic sort rearranges the input. If the caller needs the original order, this isn't the technique — say so in the interview before you write it.

## Problem 1: Missing Number (LeetCode 268)

`n` distinct numbers from `[0, n]`, one missing. Values start at 0, so value `v` belongs at index `v` — except `n` itself, which has no index in a length-`n` array. That exception is the whole problem: skip `n` during placement, and the scan finds the gap.

```java
static int missingNumber(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        int v = nums[i];
        if (v < nums.length && nums[i] != nums[v]) {
            swap(nums, i, v); // value v belongs at index v
        } else {
            i++;
        }
    }
    for (i = 0; i < nums.length; i++) {
        if (nums[i] != i) {
            return i;
        }
    }
    return nums.length; // every index matched: n itself is missing
}
```

Trace on `[3, 0, 1]`, exactly as it runs:

- `i=0`, `v=3`: no home index (`3 == length`) → advance. `[3, 0, 1]`
- `i=1`, `v=0`: swap with index 0 → `[0, 3, 1]`, `i` stays.
- `i=1`, `v=3`: no home → advance. `[0, 3, 1]`
- `i=2`, `v=1`: swap with index 1 → `[0, 1, 3]`, `i` stays.
- `i=2`, `v=3`: no home → advance. Scan: index 2 holds `3`, not `2` → answer **2**.

The edge case interviewers probe is `[0, 1]` → `2`: every index matches, so the answer is the value with no home, returned by the final line. The other probe: "what if you can't modify the input?" Then XOR is the answer — same O(n)/O(1), no mutation:

```java
static int missingNumberXor(int[] nums) {
    int missing = nums.length;
    for (int i = 0; i < nums.length; i++) {
        missing ^= i ^ nums[i];
    }
    return missing;
}
```

Every present value appears twice across the indices and the array and cancels (`x ^ x == 0`); only the missing value survives. Prefer XOR over the sum formula `n(n+1)/2 - sum`: the sum overflows an `int` once `n` passes 65,535, and XOR has no such ceiling. Both versions are in the test run below, checked against each other on 500 random inputs.

## Problem 2: Find All Duplicates (LeetCode 442)

Values in `[1, n]`, each appearing once or twice. Values start at 1, so value `v` belongs at index `v - 1`. Place everything, then scan: a value sitting at the wrong index is a duplicate, because its home is already occupied by its twin.

```java
static List<Integer> findDuplicates(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        int correct = nums[i] - 1; // value v belongs at index v - 1
        if (nums[i] != nums[correct]) {
            swap(nums, i, correct);
        } else {
            i++;
        }
    }
    List<Integer> duplicates = new ArrayList<>();
    for (i = 0; i < nums.length; i++) {
        if (nums[i] != i + 1) {
            duplicates.add(nums[i]);
        }
    }
    return duplicates;
}
```

Trace on `[4, 3, 2, 7, 8, 2, 3, 1]`:

- `i=0` swaps four times, following the chain home: `[7, 3, 2, 4, 8, 2, 3, 1]` → `[3, 3, 2, 4, 8, 2, 7, 1]` → `[2, 3, 3, 4, 8, 2, 7, 1]` → `[3, 2, 3, 4, 8, 2, 7, 1]`.
- `i=0` now holds `3`, and index 2 already holds `3` — the second `3` has nowhere to go → advance. Indices 1–3 are home, advance through them.
- `i=4` swaps twice: `[3, 2, 3, 4, 1, 2, 7, 8]` → `[1, 2, 3, 4, 3, 2, 7, 8]`.
- `i=4` holds `3` (home occupied), `i=5` holds `2` (home occupied) → advance. Scan: index 4 holds `3 ≠ 5`, index 5 holds `2 ≠ 6` → duplicates **3 and 2** (any order is accepted).

**The trap that infinite-loops this problem is the `!=` guard.** Without `nums[i] != nums[correct]`, the second `3` at `i=0` swaps with the `3` at index 2 forever — swapping equal values changes nothing, `i` never advances. I verified it: the guard-less loop on `[1, 1]` was still running at a 10,000-step cap. With the guard, a duplicate is detected exactly when its home is already occupied by an equal value, and the loop moves on. State this in the interview; it's the detail that separates "memorized the pattern" from "understands the termination argument."

Two precision notes. First, each duplicate is reported exactly once: with at most two copies, one sits at home and the other sits at exactly one mismatched slot. Second, space is O(1) *auxiliary* — the returned list is output space, which every solution must allocate. The placement itself uses no extra array.

## Problem 3: First Missing Positive (LeetCode 41)

Unsorted array, arbitrary integers — negatives, zero, duplicates, values far outside the range. Return the smallest missing positive integer, in O(n) time and O(1) auxiliary space. This is rated Hard, but it's Problem 2 with a range check: only values in `[1, n]` have a home, and everything else is skipped.

Why is the answer always in `[1, n+1]`? Pigeonhole: there are only `n` slots, so the array can hold at most `n` distinct positives from `1..n`. If all of `1..n` are present, the answer is `n+1`; otherwise the first gap is at most `n`.

```java
static int firstMissingPositive(int[] nums) {
    int i = 0;
    while (i < nums.length) {
        int v = nums[i];
        if (v >= 1 && v <= nums.length && nums[v - 1] != v) {
            swap(nums, i, v - 1); // value v belongs at index v - 1
        } else {
            i++;
        }
    }
    for (i = 0; i < nums.length; i++) {
        if (nums[i] != i + 1) {
            return i + 1;
        }
    }
    return nums.length + 1;
}
```

Trace on `[3, 4, -1, 1]`:

- `i=0`, `v=3`: swap with index 2 → `[-1, 4, 3, 1]`, `i` stays.
- `i=0`, `v=-1`: out of range → advance.
- `i=1`, `v=4`: swap with index 3 → `[-1, 1, 3, 4]`; then `v=1`: swap with index 0 → `[1, -1, 3, 4]`.
- `i=1`, `v=-1`: out of range → advance. Indices 2–3 are home. Scan: index 1 holds `-1`, not `2` → answer **2**.

**The trap here is the order of the conditions.** The range check `v >= 1 && v <= nums.length` must come *before* `nums[v - 1]`, or an input like `[7, 8, 9, 11, 12]` throws `ArrayIndexOutOfBoundsException` on the first element. Java's `&&` short-circuits left to right, so the ordering is the protection — get it backwards and the "Hard" problem fails on its own third example. One more edge the tests cover: no `Math.abs` anywhere in this code, so `Integer.MIN_VALUE` is safe (unlike sign-marking solutions, where `Math.abs(Integer.MIN_VALUE)` overflows back to negative). `[Integer.MIN_VALUE, 1]` correctly returns `2`.

## When cyclic sort does not apply

The pattern needs a value-to-index mapping. No mapping, no pattern:

- **Values with no bounded range relationship** — arbitrary strings, objects, or integers where the value says nothing about position. Sort or hash instead.
- **Input must not be mutated** — the placement pass rearranges the array. For Missing Number, XOR keeps O(n)/O(1) without touching the input; for the other two, the honest fallback is a `HashSet` at O(n) space. Name the tradeoff; don't pretend the constraint away.
- **Duplicates with no guard rule** — if your swap condition can fire on equal values, the loop doesn't terminate (see Problem 2). Before trusting any placement loop, check the invariant: *every swap must put a value in its final home, and a value already home must never move.* If either half fails, the O(n) bound fails with it.

The natural follow-up exercise is LeetCode 448 (Find All Numbers Disappeared): same placement loop as Problem 2, but the scan collects the *missing* values (`i + 1` at each mismatched index) instead of the misplaced ones. And 287 (Find the Duplicate Number) is the deliberate counterexample — it forbids modifying the array, which rules cyclic sort out and pushes you to Floyd's cycle detection instead. Knowing which constraint kills which technique is the actual interview skill.

## Prove it: 12 tests, three brute-force checks

Every method above is the code that ran — no simplified-for-print versions. 12 JUnit tests, run twice on Temurin JDK 25.0.5 (JUnit 6.0.3, `javac --release 25`), all passing both runs:

- **Examples from the problem statements**, including all three First Missing Positive examples, the `[9, 6, 4, 2, 3, 5, 7, 0, 1] → 8` case, and single-element edges (`[0] → 1`, `[1] → 0`).
- **Three randomized brute-force checks**, 500 trials each with fixed seeds: Missing Number (both the placement and XOR versions) against the known missing value; Find All Duplicates against a frequency count; First Missing Positive against a `HashSet` scan, over arrays mixing negatives, zero, duplicates, and out-of-range values.
- **The traps, executed**: the guard-less loop confirmed non-terminating at a 10,000-step cap on `[1, 1]`; `Integer.MIN_VALUE`/`Integer.MAX_VALUE` inputs; the swap-count bound checked over 2,000 random arrays (never exceeded `n`).

The most convincing test is the property check for Problem 3 — random chaos in, brute-force truth out:

```java
@Test
void firstMissingPositiveMatchesBruteForceOnRandomInputs() {
    Random rng = new Random(43);
    for (int trial = 0; trial < 500; trial++) {
        int n = rng.nextInt(30);
        int[] nums = new int[n];
        Set<Integer> present = new HashSet<>();
        for (int i = 0; i < n; i++) {
            nums[i] = rng.nextInt(2 * n + 5) - n; // negatives, zero, in/out of range
            present.add(nums[i]);
        }
        int expected = 1;
        while (present.contains(expected)) expected++;
        assertEquals(expected, CyclicSort.firstMissingPositive(nums), "trial " + trial);
    }
}
```

## The 2-minute interview answer

"When values come from a known range, the array is its own hash table: value `v` belongs at index `v`, or `v - 1` for 1-based ranges. I walk the array swapping each value home, staying on the same index after a swap because a new unchecked value just arrived. Every swap fixes one slot permanently, so there are at most `n` swaps — O(n) time, O(1) extra space — and then one scan reads the answer off the mismatched positions: the missing number, the duplicates, or the first gap. Two guards matter: skip values with no home before indexing, and never swap when the home already holds an equal value, or duplicates loop forever. The tradeoff is mutation — if the input must stay untouched, I'd use XOR for Missing Number or a hash set for the rest."

## Cheat sheet

| Problem | Range | Home index | Skip / guard | Scan reads |
|---|---|---|---|---|
| Missing Number (268) | `[0, n]`, distinct | `v` | `v == n` has no home | first `nums[i] != i`, else `n` |
| Find All Duplicates (442) | `[1, n]`, ≤ twice | `v - 1` | home holds equal value → advance | misplaced values = duplicates |
| First Missing Positive (41) | any `int` | `v - 1` | `v < 1` or `v > n`, or home holds `v` | first `nums[i] != i + 1`, else `n + 1` |

One loop, three problems, and the Hard one is just the Medium one with a range check. Pair it with the [two-pointer playbook](/posts/two-pointer-playbook-java/) and you have the two patterns that cover a surprising share of array interviews.
