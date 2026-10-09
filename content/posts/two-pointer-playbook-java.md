---
title: 'The Two-Pointer Playbook: 5 Patterns, One Technique, Zero Extra Space'
date: 2026-10-09T10:00:00-04:00
draft: false
ShowToc: true
description: >-
  Two pointers is the highest-ROI technique in coding interviews: two indices,
  one decision rule, O(1) extra space. This playbook covers the five patterns
  interviewers actually test — opposite ends, greedy shrink, fast and slow,
  sliding window, and fix-one-plus-two-pointers — each with Java code, the
  complexity math, and the "why does this work" argument they always ask for.
tags:
  - java
  - algorithms
  - interview
  - backend
categories: article
keywords:
  - two pointer technique java
  - two sum ii container with most water
  - floyd cycle detection
  - sliding window longest substring
  - 3sum java interview
---

Two pointers is the highest-ROI technique in coding interviews. The entire playbook is one idea: **two indices walk through the data, and at each step a decision rule tells you which one moves.** No extra arrays — O(1) extra space in nearly every pattern, usually O(n) time. Interviewers love it because it tests whether you can *discard search space*, not just whether you can code.

Five patterns cover nearly every two-pointer question you'll meet. Learn the decision rule for each, not just the code. (One honest exception up front: the sliding-window pattern trades O(alphabet) space for its single pass — the cheat sheet says so. Everything else is O(1).)

## Pattern 1: Opposite ends — Two Sum II

Sorted array, find two numbers summing to a target. The brute force is O(n²); the two-pointer version is O(n).

```java
int[] twoSum(int[] numbers, int target) {
    int l = 0, r = numbers.length - 1;
    while (l < r) {
        int sum = numbers[l] + numbers[r];
        if (sum == target) return new int[]{l + 1, r + 1}; // 1-indexed
        else if (sum < target) l++;  // need a bigger sum → move left up
        else r--;                    // need a smaller sum → move right down
    }
    throw new IllegalArgumentException("no solution");
}
```

The decision rule: the array is sorted, so the sum tells you exactly which side is wrong. Too small? The left pointer is the smallest unused value — advancing it is the *only* way to grow the sum. That "only way" reasoning is what makes it O(n) instead of O(n²): each step eliminates one index permanently.

## Pattern 2: Greedy shrink — Container With Most Water

Two lines, maximize `min(height[l], height[r]) × (r - l)`. Same opposite-ends setup, subtler decision rule:

```java
int maxArea(int[] height) {
    int l = 0, r = height.length - 1, best = 0;
    while (l < r) {
        best = Math.max(best, Math.min(height[l], height[r]) * (r - l));
        if (height[l] < height[r]) l++;
        else r--;
    }
    return best;
}
```

**The question interviewers always ask: why move the shorter one?** Because the width shrinks every step no matter what, so the area can only grow via height — and height is capped by the *shorter* line. Moving the taller line keeps the same cap with less width: strictly worse. Moving the shorter line is the only move that can find a taller cap. Two sentences, and you've shown you understand the proof, not just the code.

## Pattern 3: Fast and slow — Linked list cycle

Floyd's tortoise and hare. One pointer moves one step, the other moves two; if there's a cycle, they meet.

```java
static class ListNode {
    int val; ListNode next;
    ListNode(int val) { this.val = val; }
}

boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

**Why they must meet:** once both pointers are inside the cycle, the fast pointer gains one node per step on the slow one — the gap shrinks by exactly 1 each iteration, so it hits 0 within one lap. No gap size can be skipped. O(n) time, O(1) space — the "no extra space" constraint is the whole point; a `HashSet` of visited nodes is the O(n)-space answer they don't want.

The same fast/slow idea finds the middle of a list in one pass (when fast hits the end, slow is at the middle) and removes the nth node from the end. One technique, three questions.

## Pattern 4: Sliding window — longest substring without repeating characters

The window `[l, r]` is always valid; `r` expands every step, `l` jumps only when a duplicate appears:

```java
int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> last = new HashMap<>();
    int best = 0, l = 0;
    for (int r = 0; r < s.length(); r++) {
        char c = s.charAt(r);
        if (last.containsKey(c)) l = Math.max(l, last.get(c) + 1);
        last.put(c, r);
        best = Math.max(best, r - l + 1);
    }
    return best;
}
```

Two details that separate a clean answer from a buggy one:

- `l = Math.max(l, last.get(c) + 1)` — the `Math.max` matters. Without it, a duplicate whose previous occurrence is *behind* the current window start would drag `l` backwards. (Trace `"abba"` by hand if you don't believe it.)
- Each index is visited at most twice (once by `r`, once by `l`), so it's O(n) time despite the nested-looking structure. Space is O(min(n, alphabet)).

## Pattern 5: Fix one, two-pointer the rest — 3Sum

Find all unique triplets summing to zero. Sort (O(n log n)), fix the first element, run Pattern 1 on the remainder — with deduplication:

```java
List<List<Integer>> threeSum(int[] nums) {
    Arrays.sort(nums);
    List<List<Integer>> res = new ArrayList<>();
    for (int i = 0; i < nums.length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue; // skip duplicate first elements
        int l = i + 1, r = nums.length - 1;
        while (l < r) {
            int sum = nums[i] + nums[l] + nums[r];
            if (sum == 0) {
                res.add(List.of(nums[i], nums[l], nums[r]));
                while (l < r && nums[l] == nums[l + 1]) l++;
                while (l < r && nums[r] == nums[r - 1]) r--;
                l++; r--;
            } else if (sum < 0) l++;
            else r--;
        }
    }
    return res;
}
```

The decision rule is Pattern 1's, but the interview signal is the dedup: skip duplicate `i` values *and* duplicate `l`/`r` values after a hit, or you'll return the same triplet five times. Total: **O(n²)** time — the nested two-pointer search dominates the **O(n log n)** sort — and **O(1)** extra space besides the output.

## Prove it: one test per pattern

```java
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class TwoPointerTest {

    @Test
    void twoSumSorted() {
        assertArrayEquals(new int[]{1, 2}, twoSum(new int[]{2, 7, 11, 15}, 9));
    }

    @Test
    void containerExample() {
        assertEquals(49, maxArea(new int[]{1, 8, 6, 2, 5, 4, 8, 3, 7}));
    }

    @Test
    void cycleDetected() {
        ListNode a = new ListNode(1), b = new ListNode(2), c = new ListNode(3);
        a.next = b; b.next = c; c.next = b; // cycle
        assertTrue(hasCycle(a));
        c.next = null;
        assertFalse(hasCycle(a));
    }

    @Test
    void slidingWindow() {
        assertEquals(3, lengthOfLongestSubstring("abcabcbb"));
        assertEquals(1, lengthOfLongestSubstring("bbbbb"));
        assertEquals(3, lengthOfLongestSubstring("pwwkew"));
    }

    @Test
    void threeSumExample() {
        var res = threeSum(new int[]{-1, 0, 1, 2, -1, -4});
        assertEquals(2, res.size()); // [-1,-1,2] and [-1,0,1]
    }
}
```

(The `"abba"` case from Pattern 4 makes a good extra assertion — it's the input that catches the missing `Math.max`.)

## The 2-minute interview answer

> "Two pointers is one idea: two indices, one decision rule, and mostly O(1) extra space. Opposite ends on a sorted array gives Two Sum II — the sum tells you which pointer is wrong. Container With Most Water is the same setup with a greedy rule: always move the shorter line, because width only shrinks so height is the only thing that can grow. Fast and slow gives Floyd's cycle detection — the fast pointer gains exactly one node per lap, so they must meet. Sliding window keeps the window valid while the right pointer expands — longest substring without repeats, jumping the left pointer past the last duplicate. And 3Sum is sort plus fix-one-and-two-pointer with dedup at all three positions. Every one is O(n) or O(n²) time, with O(1) extra space except the sliding window's O(alphabet) last-seen map."

## Cheat sheet

| Pattern | Decision rule | Example | Complexity |
|---|---|---|---|
| Opposite ends | Sum tells you which side is wrong | Two Sum II (sorted) | O(n) time, O(1) space |
| Greedy shrink | Move the shorter line — width only shrinks | Container With Most Water | O(n) time, O(1) space |
| Fast and slow | Fast gains 1/lap → must meet in a cycle | Linked list cycle | O(n) time, O(1) space |
| Sliding window | Expand right always; jump left past violations | Longest substring, no repeats | O(n) time, O(alphabet) space |
| Fix one + two pointers | Sort, fix first, two-pointer the rest, dedup all three | 3Sum | O(n²) time, O(1) space |

Five patterns, one idea: at each step, know exactly which pointer moves and why. If you can state the "why" — the shorter line, the shrinking gap, the jumping left pointer — you've already passed.
