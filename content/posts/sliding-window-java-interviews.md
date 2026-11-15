---
title: 'Sliding Window: The Pattern Behind Every Substring Question'
date: 2026-11-15T08:30:00-05:00
draft: false
ShowToc: true
description: >-
  One window, two pointers, and a constraint: expand the right end every step,
  shrink the left end only to restore the rule. That loop solves the fixed-size
  maximum-average problem, the longest substring without repeats, and the Hard
  minimum-window problem — each in O(n) time, each with a worked trace and
  brute-force-checked tests. All 9 tests run twice on JDK 25.
tags:
  - java
  - algorithms
  - interview
  - backend
categories: article
keywords:
  - sliding window java
  - longest substring without repeating characters
  - minimum window substring leetcode 76
  - maximum average subarray leetcode 643
  - fixed variable sliding window interview
---

The [two-pointer playbook](/posts/two-pointer-playbook-java/) introduced the sliding window as one pattern among five. This post gives it the full treatment, because it is the pattern interviewers escalate through: the Easy fixed-size version, the Medium variable-size version, and the Hard frequency-constrained version are the *same loop* with a different constraint. Learn the loop once and the escalation is just a harder rule to maintain.

The loop is: **a window `[l, r]` where `r` expands every step and `l` shrinks only until the window satisfies the constraint again.** The window is always valid when the shrink loop finishes — that invariant is the whole technique. And it is O(n) for a reason you can state in one sentence: `l` and `r` only ever move right, so each index enters the window once and leaves it once. No index is ever revisited, no matter how nested the code looks. I checked the bound empirically: across 2,000 random strings, the total shrink steps in the longest-substring loop never exceeded `n`.

Two shapes cover everything below. A **fixed-size** window keeps its length at `k` and slides by swapping one element out for one element in. A **variable-size** window grows and shrinks to fit a constraint — "no repeats", "contains all of `t`" — and the answer is the longest or shortest valid window seen. The tell in a problem statement is a contiguous stretch — subarray, substring — plus a rule about what's inside it.

One standing caveat, stated once: the string solutions below use `int[128]` tables, which is fine because LeetCode 3 allows any ASCII character and LeetCode 76 uses English letters. For arbitrary Unicode input, swap the tables for `HashMap`s and the space becomes O(alphabet) instead of fixed-size. The algorithm doesn't change.

## Problem 1: Fixed window — maximum sum, maximum average (LeetCode 643)

Contiguous subarray of length exactly `k`, maximum sum. The brute force re-sums every window: O(n·k). The window version sums the first window once, then slides: each step adds the newcomer and drops the leaver, O(1) per step.

```java
static long maxSumSubarray(int[] nums, int k) {
    if (k < 1 || k > nums.length) {
        throw new IllegalArgumentException("k must satisfy 1 <= k <= nums.length");
    }
    long window = 0;
    for (int i = 0; i < k; i++) {
        window += nums[i];
    }
    long best = window;
    for (int r = k; r < nums.length; r++) {
        window += (long) nums[r] - nums[r - k]; // slide: add the newcomer, drop the leaver
        best = Math.max(best, window);
    }
    return best;
}
```

Trace on the LeetCode 643 example, `[1, 12, -5, -6, 50, 3]`, `k = 4`:

- First window `[1, 12, -5, -6]`, sum `2`. Best `2`.
- Slide: `+50`, `-1` → `[12, -5, -6, 50]`, sum `51`. Best `51`.
- Slide: `+3`, `-12` → `[-5, -6, 50, 3]`, sum `42`. Best stays `51`.

LeetCode 643 asks for the maximum *average*, which is the same window: every window has length `k`, so the largest sum is the largest average — compare sums, divide once at the end (`51 / 4 = 12.75`). Dividing inside the loop would work but invites floating-point noise for no benefit. Two details worth naming: the accumulator is a `long`, and the `(long)` cast widens `nums[r]` *before* the subtraction, so large inputs can't wrap either the running sum or the newcomer-minus-leaver difference the way `int` arithmetic can; and the `k` validation matches the problem's own constraint (`1 <= k <= n`), so `k = 0` and `k > n` fail loudly instead of returning a nonsense answer. Time O(n), space O(1).

## Problem 2: Variable window — longest substring without repeats (LeetCode 3)

Longest substring with no character appearing twice. The window's constraint is "all distinct": expand `r`, and whenever the new character duplicates one already in the window, shrink `l` until the duplicate leaves. Written in the general template form, so the shape matches Problem 3:

```java
static int lengthOfLongestSubstring(String s) {
    int[] count = new int[128];
    int best = 0, l = 0;
    for (int r = 0; r < s.length(); r++) {
        count[s.charAt(r)]++;
        while (count[s.charAt(r)] > 1) { // duplicate: shrink until it leaves
            count[s.charAt(l)]--;
            l++;
        }
        best = Math.max(best, r - l + 1);
    }
    return best;
}
```

Trace on `"abcabcbb"`, exactly as it runs:

- `r=0..2`: `"a"`, `"ab"`, `"abc"` — best `3`.
- `r=3`, `'a'` duplicates: drop index 0 → `"bca"`. `r=4`, `'b'` duplicates: drop index 1 → `"cab"`. `r=5`, `'c'` duplicates: drop index 2 → `"abc"`. Best stays `3`.
- `r=6`, `'b'` duplicates: drop `'a'`, then `'b'` → `"cb"`. `r=7`, `'b'` duplicates: drop `'c'`, then `'b'` → `"b"`. Answer **3**.

Note the double shrink at `r=6`: the loop drops characters one at a time until the *new* character's count is back to 1 — it doesn't stop at the first drop. That's the template working as designed, and it's why the shrink loop's total work stays O(n): `l` only moves right.

**The refinement interviewers expect next is the jump.** Instead of shrinking one character at a time, remember each character's last index and leap `l` past the previous occurrence in one move. The playbook has this version with a `HashMap`; the array form is identical in structure:

```java
static int lengthOfLongestSubstringJump(String s) {
    int[] last = new int[128];
    Arrays.fill(last, -1);
    int best = 0, l = 0;
    for (int r = 0; r < s.length(); r++) {
        char c = s.charAt(r);
        l = Math.max(l, last[c] + 1); // never move l backwards
        last[c] = r;
        best = Math.max(best, r - l + 1);
    }
    return best;
}
```

**The trap is the `Math.max`.** Without it, a duplicate whose previous occurrence sits *behind* the current window start drags `l` backwards, and the window silently includes characters it already passed. I executed the buggy version: on `"abba"` it reports `3` instead of the correct `2` — at the final `'a'`, `l` jumps back from `2` to `1` and the window `"bba"` gets counted as length 3. With `Math.max`, `l` is monotonic and the bug is impossible. Both versions are in the test run below, checked against each other and a brute force on 500 random strings.

Space, stated carefully: the `int[128]` tables are fixed-size for ASCII input. The `HashMap` version from the playbook is O(min(n, alphabet)) — the alphabet bound is the honest one when the input isn't ASCII.

## Problem 3: Variable window, frequency constraint — minimum window (LeetCode 76)

Shortest substring of `s` containing every character of `t`, with multiplicity. The window's constraint is "covers `t`": expand until it does, then shrink while it still does, recording the shortest valid window seen. The engineering problem is checking coverage in O(1): recounting the window's frequencies on every step would make the whole thing O(n × alphabet). The fix is a `formed` counter — how many distinct characters have reached their required count — updated incrementally as characters enter and leave:

```java
static String minWindow(String s, String t) {
    if (t.isEmpty()) {
        return "";
    }
    int[] need = new int[128];
    for (int i = 0; i < t.length(); i++) {
        need[t.charAt(i)]++;
    }
    int required = 0;
    for (int n : need) {
        if (n > 0) {
            required++;
        }
    }
    int[] have = new int[128];
    int formed = 0, l = 0, bestL = 0, bestLen = Integer.MAX_VALUE;
    for (int r = 0; r < s.length(); r++) {
        char c = s.charAt(r);
        if (need[c] > 0) {
            have[c]++;
            if (have[c] == need[c]) {
                formed++;
            }
        }
        while (formed == required) { // valid: shrink while it stays valid
            if (r - l + 1 < bestLen) {
                bestLen = r - l + 1;
                bestL = l;
            }
            char out = s.charAt(l);
            if (need[out] > 0) {
                if (have[out] == need[out]) {
                    formed--;
                }
                have[out]--;
            }
            l++;
        }
    }
    return bestLen == Integer.MAX_VALUE ? "" : s.substring(bestL, bestL + bestLen);
}
```

Trace on the classic example, `s = "ADOBECODEBANC"`, `t = "ABC"`:

- `r=5` (`'C'`): all three letters present — window `[0..5]`, `"ADOBEC"`, length 6, first best. Shrink drops `'A'` and validity breaks.
- `r=10` (`'A'`): valid again at `[1..10]`, length 10 — no improvement. Shrink walks `l` through `D`, `O`, `B`, `E` (still valid), then drops `'C'` and validity breaks.
- `r=12` (`'C'`): valid at `[6..12]`. Shrink: `[8..12]`, `"EBANC"`, length 5 — new best. Shrink once more: `[9..12]`, `"BANC"`, length 4 — new best. Dropping `'B'` breaks validity. Answer **"BANC"**.

Two details carry the correctness argument. First, `formed` increments only on *equality* (`have[c] == need[c]`): a surplus third `'B'` must not count as forming anything, and on the way out `formed` decrements only when a character drops *from* its required count. Get either comparison wrong and the window claims validity it doesn't have. Second, multiplicity is real: with `t = "AABC"` the answer on the same string is `"ADOBECODEBA"` — the window must span both `'A'`s — and the equality checks are what enforce the doubled requirement. Both the equality rule and the duplicate-`t` case are covered by the brute-force check below, which compares against every substring on 500 random inputs.

## When the window doesn't slide

The technique needs one property: **shrinking the window from the left must be able to restore the constraint.** If that fails, the loop is unsound no matter how clean it looks:

- **"Subarray sum exactly `k`" with negative numbers.** Growing the window can *shrink* the sum, so "too big, shrink left" no longer points the right way — a window that's over target might need to grow, not shrink. The right tool is prefix sums plus a hash map, not a window. (With all-non-negative numbers the window works fine, because sums only grow as the window grows.)
- **Subsequences, not substrings.** A window is contiguous by definition. If the problem lets you skip characters, you're in subsequence/DP territory and no window formulation applies.
- **"Find all windows" or order statistics over windows.** The loop finds the best window in one pass; enumerating every valid window or ranking them is a different job, usually with a different data structure (a deque for sliding maximums, for example).

Saying which constraint kills the technique — and naming the replacement — is the part interviewers actually score. The code is the easy half.

## Prove it: 9 tests, three brute-force checks

Every method above is the code that ran — no simplified-for-print versions. 9 JUnit tests, run twice on Temurin JDK 25.0.5 (JUnit 6.0.3, `javac --release 25`), all passing both runs:

- **Examples from the problem statements**, including all of LeetCode 3's (`"abcabcbb"` → 3, `"bbbbb"` → 1, `"pwwkew"` → 3), LeetCode 643's (`[1, 12, -5, -6, 50, 3]`, `k = 4` → 12.75), and LeetCode 76's (`"BANC"`, `"a"`, and the no-window `""` cases) — plus edges: empty strings, `k = 1`, `k = n`, all-negative arrays, invalid `k` rejected, and a `long` accumulator check with three `Integer.MAX_VALUE` values.
- **Three randomized brute-force checks**, 500 trials each with fixed seeds: max-sum against the O(n·k) recomputation; both longest-substring forms against the O(n²) check of every start position; minimum window against every substring, comparing length and coverage (ties mean the exact string can differ, so the check compares the length and verifies the returned window covers `t`).
- **The traps, executed**: the `Math.max`-less jump confirmed wrong on `"abba"` (3 instead of 2); the shrink-step bound checked over 2,000 random strings (never exceeded `n`).
- One honest note: the brute-force check caught a wrong hand-computed expectation of mine during development — for `t = "AABC"` I first wrote down the whole 13-character string, but the algorithm and the brute force agreed the answer is the 11-character `"ADOBECODEBA"`. The test now asserts the verified value.

## The 2-minute interview answer

"A window `[l, r]` where `r` expands every step and `l` shrinks only until the constraint holds again. Fixed windows keep length `k` and swap one element per step; variable windows grow and shrink to fit a rule like 'no repeats' or 'covers all of `t`'. It's O(n) because both pointers only move right — each index enters and leaves once. The engineering is in maintaining the constraint incrementally: counts for duplicates, a `formed` counter for coverage, so validity is O(1) per step instead of a recount. Two traps: `l` must never move backwards when jumping past a previous occurrence, and the technique fails when shrinking can't restore the constraint — exact-sum with negatives is the classic case, and that's prefix sums instead."

## Cheat sheet

| Problem | Window | Constraint | Shrink trigger | Answer read |
|---|---|---|---|---|
| Max sum / average, size `k` (643) | fixed, length `k` | none — length is the rule | never; slide by one | largest window sum (÷ `k` for average) |
| Longest substring, no repeats (3) | variable | all chars distinct | new char duplicates one in the window | longest valid window |
| Minimum window substring (76) | variable | covers every char of `t` | window is valid — shrink while it stays valid | shortest valid window |

One loop, three difficulties, and the Hard one is the Medium one with a frequency counter. Pair it with the [two-pointer playbook](/posts/two-pointer-playbook-java/) for the opposite-ends family, and [cyclic sort](/posts/cyclic-sort-java-interviews/) for the other O(n), O(1)-space pattern that shows up wherever values come from a known range.
