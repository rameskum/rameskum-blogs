---
title: "Floyd's Tortoise and Hare: Linked List Cycle Detection in Java, With the Proof"
date: 2026-10-17T18:30:00+05:30
draft: false
ShowToc: true
description: >-
  Everyone can recite "slow moves one, fast moves two." The interview
  follow-ups are where it ends: why can't fast skip slow, why resetting one
  pointer to head finds the entry, and the null-check traps. The full Floyd's
  algorithm in Java — with the proof, not the memorization.
tags:
  - java
  - algorithms
  - interview
categories: article
keywords:
  - floyd cycle detection java
  - linked list cycle entry proof
  - tortoise and hare algorithm
  - leetcode 141 142
---

"Detect a cycle in a linked list." Everyone recites it: slow moves one step, fast moves two, if they meet there's a cycle. Then the interviewer says "why can't fast skip over slow?" and the room goes quiet. Then "find where the cycle starts" — and the memorized answer runs out.

Floyd's tortoise and hare is a two-minute algorithm with a twenty-minute understanding. Here's the understanding.

## The shallow answer (and why interviewers hate it)

The HashSet version works and every candidate knows it:

```java
public boolean hasCycleHashSet(ListNode head) {
    Set<ListNode> seen = new HashSet<>();
    for (ListNode cur = head; cur != null; cur = cur.next) {
        if (!seen.add(cur)) return true;  // seen this node before → cycle
    }
    return false;
}
```

O(n) time, O(n) space. The interviewer nods and says "now do it in O(1) space." That's the actual question — the HashSet version is just the warmup to see if you know what a linked list is.

## Phase 1: detection — and why fast can't skip slow

```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;          // one step
        fast = fast.next.next;     // two steps
        if (slow == fast) return true;
    }
    return false;
}
```

Two things to get right. First, the loop condition checks `fast` **and** `fast.next` — because `fast.next.next` on a null `fast.next` is a NullPointerException, and that's the most common bug candidates write on the whiteboard. If fast reaches the end, there is no cycle; that's the only exit.

Second, the question that ends memorization: *why can't fast jump over slow without landing on it?* Because the gap between them shrinks by exactly one each iteration. Fast gains one step per round — the gap goes from d to d−1 to d−2 … to 1, then 0. It can never jump from 2 to 0 because the net gain per iteration is exactly 1. Once both pointers are inside the cycle, they *must* meet within one lap. No luck involved.

## Phase 2: finding the entry — the reset trick and its proof

This is LeetCode 142, and it's where "I memorized the code" dies. After slow and fast meet at some point inside the cycle: reset one pointer to the head, keep the other at the meeting point, and advance both **one step at a time**. They meet again at the cycle's entry.

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) break;
    }
    if (fast == null || fast.next == null) return null;  // no cycle

    slow = head;                       // reset to head
    while (slow != fast) {             // walk both at speed 1
        slow = slow.next;
        fast = fast.next;
    }
    return slow;                       // the cycle entry
}
```

Why does this work? Name the distances: let **x** = head → cycle entry, **y** = entry → meeting point, and the cycle length = **y + z** (z = meeting point → entry along the cycle). When they first meet, slow has traveled x + y, and fast has traveled twice that: 2(x + y). But fast also did k full extra laps around the cycle, so 2(x + y) = (x + y) + k(y + z), which simplifies to:

**x = (k−1)(y + z) + z**

In words: walking x steps from the head reaches the entry, and walking x steps from the meeting point *also* reaches the entry — because from the meeting point, z steps complete the current lap back to the entry, and the remaining (k−1) full laps bring you right back there. Both pointers walk x steps at the same speed, so they arrive together. The meeting point was arbitrary; the entry is determined.

The interview answer: *"The distance from head to entry equals the distance from the meeting point to the entry, modulo the cycle length. So two pointers walking at the same speed from those two spots must converge at the entry."*

## Phase 3: the length — the free follow-up

Once you have the meeting point, the cycle length is one more lap:

```java
public int cycleLength(ListNode meetingPoint) {
    int len = 0;
    ListNode cur = meetingPoint;
    do {
        cur = cur.next;
        len++;
    } while (cur != meetingPoint);
    return len;
}
```

The `do-while` matters — a plain `while` would exit immediately since `cur` starts *at* the meeting point. Small detail, classic whiteboard bug.

## The traps: what breaks on the board

- **Empty list / single node:** `head == null` → the while condition fails immediately, returns false/null. A single node with `next == null` does the same. A single node pointing to *itself* is a cycle of length 1 — slow and fast meet on the first iteration. Make sure your code handles all three; interviewers check.
- **The null checks:** `fast != null && fast.next != null`, in that order. Checking `fast.next` first NPEs. This is the #1 whiteboard bug.
- **Comparing values instead of references:** `slow.val == fast.val` is wrong — two distinct nodes can hold the same value. Compare node *identity* (`==` on references).
- **Resetting the wrong pointer:** after the meeting, reset to head and walk at speed 1. Keeping both at speed 2, or not resetting at all, finds nothing.
- **"Why not just use the HashSet?"** — because the question is testing whether you can trade time for space. Say it explicitly: the HashSet is O(n) space and perfectly fine in production; Floyd's is O(1) space and what the interview asks for.

## Where else this pattern lives

Floyd's isn't just a linked-list trick. The same two-pointer convergence shows up in: **find the duplicate number** (model the array as a linked list via indices — the duplicate is the cycle entry), **middle of the linked list** (fast reaches the end when slow reaches the middle), and **palindrome linked list** (find the middle, reverse the second half, compare). Recognizing "this is a cycle problem in disguise" is the skill the interview is actually testing.

## The 2-minute interview answer

> "For O(1) space I use Floyd's tortoise and hare: slow moves one step, fast moves two. If fast hits null there's no cycle. If there's a cycle, the gap between them shrinks by exactly one per iteration once both are inside, so they must meet — fast can't skip slow. To find the entry, I reset one pointer to the head and walk both at speed one; they meet at the entry because the head-to-entry distance equals the meeting-point-to-entry distance modulo the cycle length. Cycle length is one more lap from the meeting point. Edge cases: empty list, single node, self-loop, and the fast/fast.next null checks in the right order — and I compare node references, not values."

## Cheat sheet

| Step | Code shape |
|---|---|
| Detect | `slow=head, fast=head`; loop `fast!=null && fast.next!=null`; `slow=slow.next`, `fast=fast.next.next`; meet → cycle |
| Why they meet | Gap shrinks by exactly 1 per iteration — can't jump from 2 to 0 |
| Find entry | Reset one to `head`, walk both at speed 1 → meet at entry |
| Why reset works | `x = (k−1)(y+z) + z`: equal remaining distances to the entry |
| Cycle length | `do-while` lap from the meeting point |
| Complexity | O(n) time, O(1) space |
| Traps | Null-check order, self-loop, reference (not value) comparison, `do-while` for length |

Memorize the code and you'll pass the first question. Understand the proof and you'll pass the follow-ups — which is where the interview is actually decided.
