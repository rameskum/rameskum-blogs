---
title: 'Bit Manipulation in Java Interviews: 7 Tricks That Turn O(n) Into O(1)'
date: 2026-11-23T07:47:00-05:00
draft: false
ShowToc: true
description: >-
  Every int in Java is already a 32-bit array, and seven identities about
  those bits solve seven LeetCode questions in linear time and constant
  space: 136 (XOR cancels), 191 (Kernighan's n & (n-1)), 231, 260
  (n & -n), 268 (XOR the range), 338 (drop the last bit), 461, and 190.
  Twenty tests, run twice, every number below measured.
tags:
  - java
  - algorithms
  - interview-prep
  - bit-manipulation
categories: article
keywords:
  - bit manipulation java interview
  - leetcode 136 single number xor
  - leetcode 268 missing number xor
  - leetcode 338 counting bits
  - leetcode 260 single number iii
  - kernighan algorithm java
  - n and n-1 lowest set bit
---

Every `int` in Java is already a 32-bit array, and the interview questions in this post are all the same question in different clothes: *which identity about those bits turns a counting problem into a one-pass problem?* A HashMap counts; a bit trick does not need to, because the bits already carry the information. The seven tricks below are identities, not algorithms — each one is a line of algebra with a proof you can do on a whiteboard — and each one solves a real LeetCode question in O(n) time and O(1) space. The HashMap versions all work too, and the post says exactly where they lose: one integer of state instead of a table.

The whole toolkit fits in a table. Everything after it is application.

| Trick | Identity | What it does |
| --- | --- | --- |
| 1 | `x ^ x = 0`, `x ^ 0 = x` | Duplicates cancel |
| 2 | `n & (n - 1)` | Clears the lowest set bit |
| 3 | `n & -n` | Isolates the lowest set bit |
| 4 | XOR the values against the indices | The gap survives |
| 5 | `bits[i] = bits[i >> 1] + (i & 1)` | Reuse the answer for half the number |
| 6 | `popcount(x ^ y)` | Count the positions that differ |
| 7 | `result = (result << 1) \| (n & 1)` | Build the answer one bit at a time |

XOR is commutative and associative, so a fold over an array can be rearranged at will — pairs can be imagined sitting next to each other even when they are not. That is the property every trick in this post leans on, and the first thing to say when the interviewer asks why XOR works at all.

## Trick 1: duplicates cancel — LeetCode 136, Single Number

The statement: a non-empty array where every element appears exactly twice except one. Find the single one, in linear time and constant extra space. Length up to 30,000. The HashMap version is the obvious first draft — count each value, return the one seen once — and the statement rules it out with the constant-space requirement. The XOR fold is the whole solution:

```java
public static int singleNumber(int[] nums) {
    int x = 0;
    for (int v : nums) {
        x ^= v;
    }
    return x;
}
```

XOR every element; each pair cancels to 0, the single one survives. The walk, executed:

```
0 ^4=4 ^1=5 ^2=7 ^1=6 ^2=4
```

For `[4, 1, 2, 1, 2]`, the running XOR goes 4, 5, 7, 6, 4. At step 4 the second `1` erases the first; at step 5 the second `2` erases the first. The final result is 4. The test suite checks 200 seeded random arrays against a frequency-count brute force, so the identity is verified, not just illustrated — and the example from the statement, `[2, 2, 1] → 1`, is the first test.

The trap to name in the interview: XOR never counts. It only knows which bits were set an odd number of times. If the premise breaks — a value appearing three times, or two singles — the fold returns garbage, silently. The trick is the premise, not the loop.

## Trick 2: clear the lowest set bit — LeetCode 191, Number of 1 Bits

Counting the 1-bits of an integer is usually written as a fixed 32-step scan: check every bit, count the ones. Brian Kernighan's observation is that `n & (n - 1)` clears exactly the lowest set bit and nothing else. Subtracting 1 flips the lowest set bit to 0 and sets all lower bits to 1; the AND with `n` preserves everything above and zeroes the rest. So the loop runs once per set bit, never 32 times:

```java
public static int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
        n &= (n - 1); // clear the lowest set bit
        count++;
    }
    return count;
}
```

For `n = 12` (binary `1100`): `n - 1 = 11` (`1011`), and `n & (n - 1) = 8` (`1000`) — one bit gone, one iteration. Measured, not asserted: for `0x80000001`, which has exactly two set bits, the loop runs twice while a fixed 32-bit scan runs 32 times. For `-1`, all 32 bits set, it runs 32 — the worst case is the same, the common case is far better. The test suite checks 1,000 random ints against `Integer.bitCount`, including negatives: `-3` (`0xFFFFFFFD`) has 31 set bits, and the loop counts all 31.

The one-line corollary is LeetCode 231, Power of Two: a power of two has exactly one set bit, so clearing it leaves 0.

```java
public static boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}
```

Two guards do the work, and both are interview material. The `n > 0` guard is load-bearing: 0 passes the AND test, and so does `Integer.MIN_VALUE` (`1000...0 & 0111...1 == 0`). The parentheses are load-bearing too: `==` binds tighter than `&` in Java, so `n & n - 1 == 0` does not compile — it parses as `n & ((n - 1) == 0)`, an int ANDed with a boolean. The test suite checks every value from -100 to 10,000 against a divide-by-two brute force.

## Trick 3: isolate the lowest set bit — LeetCode 260, Single Number III

Trick 1 with two singles instead of one. The statement: every element appears exactly twice except two distinct elements that appear exactly once. Find both, in linear time and constant space. Example: `[1, 2, 1, 3, 2, 5]` → `[3, 5]`, either order.

XOR everything and the pairs cancel, leaving `a ^ b` — but that is one number, not two. The two singles differ, so `a ^ b` has at least one set bit: a position where one has a 0 and the other a 1. Trick 3 isolates it: `n & -n` is the lowest set bit, because in two's complement `-n` flips all bits and adds 1, so every bit below the lowest set bit is 0 in both `n` and `-n`, the lowest set bit is 1 in both, and every bit above differs. Partition the array on that bit — the two singles land in different groups, every pair lands in the same group — and XOR each group:

```java
public static int[] singleNumberIII(int[] nums) {
    int xor = 0;
    for (int v : nums) {
        xor ^= v;
    }
    int lowbit = xor & -xor; // the lowest set bit of a ^ b
    int a = 0;
    for (int v : nums) {
        if ((v & lowbit) != 0) {
            a ^= v;
        }
    }
    return new int[]{a, xor ^ a};
}
```

The trace, executed: for `[1, 2, 1, 3, 2, 5]`, the XOR of all is 6 (`110`), the lowbit is 2 (`010`), and the partition recovers `[3, 5]`. The second single needs no second pass — `b = xor ^ a`, since `a ^ b ^ a = b`. The test suite checks 200 seeded random arrays against a frequency-count brute force. The interview point: any set bit of `a ^ b` would do for the partition; the lowbit is just the cheapest one to find, in one expression.

## Trick 4: XOR the range — LeetCode 268, Missing Number

The statement: an array of `n` distinct numbers taken from `0, 1, 2, ..., n`. Find the missing one. Example: `[0, 1, 3]` → `2`. This is the same question the [cyclic sort post](/posts/cyclic-sort-java-interviews/) solved by placement — value `v` at index `v`, and the hole is the answer. The XOR approach is the contrasting method, and the contrast is the point: placement needs the array mutated (or a second array), XOR needs one integer.

```java
public static int missingNumber(int[] nums) {
    int x = nums.length; // index n has no matching value
    for (int i = 0; i < nums.length; i++) {
        x ^= i ^ nums[i];
    }
    return x;
}
```

XOR the indices `0..n` against the values. Every number present appears twice — once as an index, once as a value — and cancels. The missing one appears once, as an index, and survives. Start the fold at `nums.length` because index `n` has no matching value in an array of length `n`.

The third approach, the sum formula `n*(n+1)/2 - sum(nums)`, is where the post earns its keep, because it carries a bug the cyclic sort series already caught once. The product `n*(n+1)` overflows a 32-bit int before the division ever happens, and the threshold is not a round number. Measured in the test suite:

```
46340*46341 = 2147441940 (fits int); 46341*46342 = 2147534622 (overflows int)
```

`46340 × 46341 = 2,147,441,940`, just under `Integer.MAX_VALUE`; `46341 × 46342 = 2,147,534,622`, just over it — and in `int` arithmetic the product wraps negative, so the sum formula returns a wrong answer for `n ≥ 46,341` with no exception and no warning. The XOR version has no such threshold: it cannot overflow, because XOR never carries. When the interviewer asks which approach to pick, that is the answer — not the complexity, which is O(n) either way, but the failure mode. The test suite checks 200 seeded random arrays against a boolean-array brute force.

## Trick 5: drop the last bit, reuse the answer — LeetCode 338, Counting Bits

The statement: for every `i` in `0..n`, count the 1-bits in `i` and return the array. Example: `n = 5` → `[0, 1, 1, 2, 1, 2]`. Counting each number from scratch is O(n log n) bit operations; the trick is that the answer for `i` is already sitting in the answer for `i >> 1`:

```java
public static int[] countBits(int n) {
    int[] bits = new int[n + 1];
    for (int i = 1; i <= n; i++) {
        bits[i] = bits[i >> 1] + (i & 1);
    }
    return bits;
}
```

Dropping the last bit of `i` gives `i >> 1`, whose popcount is already computed; the last bit adds 0 or 1. One pass, one addition per number. The sibling identity `bits[i] = bits[i & (i - 1)] + 1` is trick 2 applied to the same table — clearing the lowest set bit leaves a smaller number whose answer is known. Either recurrence is the same idea: the table answers its own subproblems.

Measured: for `n = 1000`, the DP does 1,000 additions; recounting each number from scratch (one Kernighan step per set bit) does 4,938. The test suite checks every `n` from 0 to 2,000 against `Integer.bitCount` per element — exhaustive, not sampled — so the recurrence is verified across the whole range, not just the example.

## Trick 6: XOR, then count — LeetCode 461, Hamming Distance

The statement: the Hamming distance between two integers is the number of positions at which their bits differ. Example: `x = 1` (`001`), `y = 4` (`100`) → distance 2. The positions where they differ are exactly the set bits of `x ^ y`, so the problem is trick 2 in disguise:

```java
public static int hammingDistance(int x, int y) {
    return hammingWeight(x ^ y);
}
```

One line, and the interview value is recognising the reduction rather than writing the loop. The test suite checks 1,000 random pairs against a brute force that compares the 32-character binary strings position by position. The follow-up worth naming: the total Hamming distance over all pairs (LeetCode 477) is the same idea counted per bit position instead of per pair — count the ones at each position, multiply ones by zeroes — which turns an O(n²) pair loop into O(32n).

## Trick 7: build the answer one bit at a time — LeetCode 190, Reverse Bits

The statement: reverse the bits of a 32-bit unsigned integer. Example: input `43261596` (`00000010100101000001111010011100`) → `964176192` (`00111001011110000010100101000000`). Thirty-two steps, one per bit: take the lowest bit of `n`, append it to the answer, shift `n` right:

```java
public static int reverseBits(int n) {
    int result = 0;
    for (int i = 0; i < 32; i++) {
        result = (result << 1) | (n & 1);
        n >>>= 1;
    }
    return result;
}
```

The one character that matters is `>>>`. The signed shift `>>` drags the sign bit down, so for a negative input the top bits never become 0 and the loop fills the answer with ones — `reverseBits(-3)` must return `0xBFFFFFFF` (`-1073741825`), not a wall of ones. The test suite checks 1,000 random ints against a string-reversal oracle, negatives included.

The follow-up is the real interview question: if this function is called many times, what do you optimise? Reverse each byte once into a 256-entry lookup table, then combine four bytes — four table lookups instead of 32 loop iterations. Same trick, one level up: precompute the small answers, compose the big one. It is trick 5's shape applied to a different problem, and saying so is the answer.

## What to say in the interview

The pattern to name is "counting without a table". Any problem whose HashMap solution counts frequencies — duplicates, missing values, bit populations — is a candidate for one of these seven identities, and the check is always the same: does the premise give the bits something to cancel? XOR cancels pairs; `n & (n - 1)` spends one iteration per set bit; `n & -n` finds a bit to split on. The complexities are all O(n) time and O(1) space, which is the point — the win over the HashMap is space, not time, and the win over the sum formula is the failure mode, not the speed.

The connections to the rest of the series are the memory aids. Missing Number by XOR is the contrast to Missing Number by placement in the [cyclic sort post](/posts/cyclic-sort-java-interviews/) — same question, one integer of state instead of a mutated array, and no overflow threshold to remember. The trie post ([Trie](/posts/trie-java-interviews/)) is the opposite direction: where bits give you cancellation, a trie gives you structure — and the HashMap post ([HashMap internals](/posts/hashmap-internals-java-interview/)) is the baseline all seven tricks are measured against. Know the HashMap version first, then know why the bits are better: one integer, no table, no overflow, no second pass.

## Verification note

Problems 136, 191, 231, 260, 268, 338, 461, and 190 verified against their LeetCode statements (operations, examples, and constraints) before drafting. Twenty JUnit 6 tests, run twice on Temurin JDK 25.0.5.1, 20/20 both times: the example sequences exactly as stated (136 `[2,2,1] → 1`, 260 `[1,2,1,3,2,5] → [3,5]`, 268 `[3,0,1] → 2`, 338 `n=5 → [0,1,1,2,1,2]`, 461 `1,4 → 2`, 190 `43261596 → 964176192`), and seeded property checks against brute-force references that do not use the trick — 200 random arrays for 136 and 260, 200 for 268, 1,000 random ints for 191, 461, and 190, every value from -100 to 10,000 for 231, and every `n` from 0 to 2,000 for 338. The measured numbers in the post are the executed traces: the 136 XOR walk `0 ^4=4 ^1=5 ^2=7 ^1=6 ^2=4`; Kernighan on `0x80000001` in 2 iterations, not 32; the 260 partition with xor 6 and lowbit 2; the sum-formula overflow threshold `46340×46341 = 2,147,441,940` (fits) vs `46341×46342 = 2,147,534,622` (overflows); and countBits(1000) in 1,000 additions vs 4,938 naive bit-clearing steps. One test assertion was corrected before the first run (the popcount of -3 is 31, not 3 — two's complement counts the sign bits too). No live LeetCode submission was made; the tests are the verification.
