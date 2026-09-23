---
title: 'Consistent Hashing, Actually Working: From Modulo to a Java Ring with Virtual Nodes'
date: 2026-09-23T08:24:00-04:00
draft: false
ShowToc: true
description: >-
  Senior interviews keep drifting from LeetCode toward scoped system design,
  and consistent hashing shows up in both. Build a working hash ring in Java —
  modulo's failure mode, the ring fix, virtual nodes, tests — plus the
  Dynamo/memcached lineage and the 2-minute answer.
tags:
  - interview
  - algorithms
  - java
  - system-design
categories: article
keywords:
  - consistent hashing java
  - hash ring virtual nodes
  - system design interview
  - distributed caching interview
---

"Design a distributed cache." You say `hash(key) % N`. The interviewer nods, then asks: "A node dies at 3 AM. What happens?" If your answer is "we rehash everything," you've just told them your cache has a planned outage every time the cluster changes shape. This post builds the real answer — a working consistent hash ring in Java, with virtual nodes and tests — and the lineage and follow-ups that turn it into a senior-level answer.

## The problem: modulo-n falls over on membership change

Three cache nodes. Keys land via `hash(key) % 3`:

```java
String nodeFor(String key, List<String> nodes) {
    return nodes.get(Math.floorMod(key.hashCode(), nodes.size()));
}
```

Works fine — until a node dies and `N` becomes 2. Now `hash(key) % 2` instead of `% 3`, and **nearly every key remaps**, including keys whose node is perfectly healthy. Your surviving nodes get hammered with cache misses all at once: the thundering herd, at 3 AM, exactly when you didn't need it. Adding a node for capacity is the same story in reverse.

The property we want: when the cluster changes, only the keys that *must* move should move — roughly `K/N` of them, not all `K`.

## The ring: hash the nodes too

Consistent hashing (Karger et al., 1997 — built for web caches, the idea Akamai was founded on) puts keys **and** nodes on the same ring:

```
        node-A ●
       /        \
  ● key-1    key-2 ●
  |                |
  ● key-4    key-3 ●
       \        /
        node-B ●──── node-C ●
```

Hash each node to a point on the ring (say, a 64-bit space). Hash each key the same way. A key belongs to the **first node clockwise** from it. Now remove node-B: only the keys in node-B's arc — the ones between node-A and node-B — remap, and they go to node-C. Keys in every other arc don't move at all. Removing 1 of N nodes moves ~`K/N` keys instead of ~all of them. That's the whole trick.

In Java, the ring is a `TreeMap` — `tailMap` gives us "first node clockwise" in one call, including the wrap-around:

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHashRing<T> {

    private final SortedMap<Long, T> ring = new TreeMap<>();
    private final int virtualNodes;

    public ConsistentHashRing(int virtualNodes) {
        if (virtualNodes < 1) throw new IllegalArgumentException("virtualNodes must be >= 1");
        this.virtualNodes = virtualNodes;
    }

    public void addNode(T node) {
        for (int i = 0; i < virtualNodes; i++) {
            ring.put(hash(node + "#" + i), node);   // one point per virtual node
        }
    }

    public void removeNode(T node) {
        for (int i = 0; i < virtualNodes; i++) {
            ring.remove(hash(node + "#" + i));
        }
    }

    public T getNode(String key) {
        if (ring.isEmpty()) throw new IllegalStateException("no nodes in ring");
        SortedMap<Long, T> tail = ring.tailMap(hash(key));   // clockwise from key
        long nodeHash = tail.isEmpty() ? ring.firstKey() : tail.firstKey(); // wrap-around
        return ring.get(nodeHash);
    }

    public int points() { return ring.size(); }

    private static long hash(String s) {
        try {
            // fresh digest per call: MessageDigest instances are NOT thread-safe
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] d = md.digest(s.getBytes(StandardCharsets.UTF_8));
            long h = 0;
            for (int i = 0; i < 8; i++) h = (h << 8) | (d[i] & 0xFF);
            return h;
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

That's the complete data structure. ~50 lines.

## Why virtual nodes exist

With one point per physical node, three nodes land at three random spots — and one of them can easily own half the ring while another owns a sliver. Real clusters don't tolerate a node holding 50% of the data.

Virtual nodes fix it: each physical node gets, say, 150 points spread around the ring (`node-A#0` … `node-A#149`). The arcs interleave, so each physical node owns ~`1/N` of the keyspace with low variance. More vnodes = smoother distribution, at the cost of a bigger ring map and more gossip on membership change — 100–200 per node is the usual range. (Fun detail for interviews: Cassandra defaulted to 256 vnodes per node for years, then dropped it to 16 in 4.x once they decided the metadata wasn't worth it.)

## Seeing it work: the 3 AM test

```java
import java.util.*;

public class RingDemo {

    static Map<String, String> assign(ConsistentHashRing<String> ring, int keys) {
        Map<String, String> m = new HashMap<>();
        for (int i = 0; i < keys; i++) m.put("key-" + i, ring.getNode("key-" + i));
        return m;
    }

    static long moved(Map<String, String> before, Map<String, String> after) {
        return before.entrySet().stream()
                .filter(e -> !e.getValue().equals(after.get(e.getKey())))
                .count();
    }

    public static void main(String[] args) {
        List<String> nodes = new ArrayList<>(List.of("node-A", "node-B", "node-C"));

        // --- consistent hashing ---
        ConsistentHashRing<String> ring = new ConsistentHashRing<>(150);
        nodes.forEach(ring::addNode);
        Map<String, String> before = assign(ring, 100_000);
        ring.removeNode("node-B");
        Map<String, String> after = assign(ring, 100_000);
        System.out.printf("ring:    removed 1 of 3 nodes -> %,d of 100,000 keys moved%n",
                moved(before, after));

        // --- modulo-n, for comparison ---
        Map<String, String> mBefore = new HashMap<>();
        for (int i = 0; i < 100_000; i++)
            mBefore.put("key-" + i, nodes.get(Math.floorMod(("key-" + i).hashCode(), 3)));
        nodes.remove("node-B");
        long mMoved = 0;
        for (int i = 0; i < 100_000; i++)
            if (!mBefore.get("key-" + i).equals(nodes.get(Math.floorMod(("key-" + i).hashCode(), 2)))) mMoved++;
        System.out.printf("modulo:  removed 1 of 3 nodes -> %,d of 100,000 keys moved%n", mMoved);
    }
}
```

Typical output (exact numbers vary with the hash, the shape doesn't):

```
ring:    removed 1 of 3 nodes -> ~33,000 of 100,000 keys moved
modulo:  removed 1 of 3 nodes -> ~99,000 of 100,000 keys moved
```

One third versus nearly everything. That gap is the entire interview answer.

## Tests that prove the properties (not the statistics)

The nice thing about testing a hash ring: the important properties are **structural**, not statistical — they hold for any decent hash function, so the tests are deterministic:

```java
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class ConsistentHashRingTest {

    private ConsistentHashRing<String> ringOf(String... nodes) {
        ConsistentHashRing<String> ring = new ConsistentHashRing<>(150);
        Arrays.stream(nodes).forEach(ring::addNode);
        return ring;
    }

    @Test
    void emptyRingThrows() {
        assertThrows(IllegalStateException.class,
                () -> new ConsistentHashRing<String>(150).getNode("k"));
    }

    @Test
    void removingNodeOnlyMovesThatNodesKeys() {
        ConsistentHashRing<String> ring = ringOf("A", "B", "C");
        Map<String, String> before = new HashMap<>();
        for (int i = 0; i < 20_000; i++) before.put("k" + i, ring.getNode("k" + i));

        ring.removeNode("B");

        long movedFromElsewhere = 0, movedTotal = 0;
        for (int i = 0; i < 20_000; i++) {
            String b = before.get("k" + i), a = ring.getNode("k" + i);
            if (!b.equals(a)) {
                movedTotal++;
                if (!b.equals("B")) movedFromElsewhere++;
            }
        }
        assertEquals(0, movedFromElsewhere, "keys on surviving nodes must not move");
        assertTrue(movedTotal > 0, "B's keys must go somewhere");
    }

    @Test
    void addingNodeOnlyStealsKeysNeverReshuffles() {
        ConsistentHashRing<String> ring = ringOf("A", "B", "C");
        Map<String, String> before = new HashMap<>();
        for (int i = 0; i < 20_000; i++) before.put("k" + i, ring.getNode("k" + i));

        ring.addNode("D");

        for (int i = 0; i < 20_000; i++) {
            String b = before.get("k" + i), a = ring.getNode("k" + i);
            assertTrue(b.equals(a) || a.equals("D"),
                    "a key may stay or move to D, never A->B->C reshuffle");
        }
    }

    @Test
    void virtualNodesEvenOutLoad() {
        // structural version: every physical node must own at least one arc
        ConsistentHashRing<String> ring = ringOf("A", "B", "C");
        Set<String> owners = new HashSet<>();
        for (int i = 0; i < 20_000; i++) owners.add(ring.getNode("k" + i));
        assertEquals(Set.of("A", "B", "C"), owners);
        assertEquals(450, ring.points()); // 3 nodes x 150 vnodes
    }
}
```

`removingNodeOnlyMovesThatNodesKeys` is the test I'd walk through on a whiteboard — it encodes the minimal-disruption guarantee directly.

## The lineage (for "where is this used in real life?")

Drop two of these and the answer goes from "textbook" to "has operated systems":

- **1997 — Karger et al.** Consistent hashing invented for web caches; the idea Akamai was built on.
- **2001 — Chord** (MIT). Distributed hash table over a ring; every P2P system since is a footnote.
- **2007 — Amazon Dynamo** ("Dynamo: Amazon's Highly Available Key-value Store"). Virtual nodes, preference lists (each key replicated on the next N nodes clockwise), sloppy quorums. This paper is the reason the technique is in every system-design interview.
- **2007 — ketama** (Richard Jones, last.fm). Brought consistent hashing to memcached clients; still the algorithm behind most cache client libraries.
- **Cassandra** — token ring with vnodes (`num_tokens`; 256 per node for years, 16 since 4.x).

## The 2-minute interview answer

> "I'd partition with consistent hashing: hash keys and nodes onto a ring, each key goes to the first node clockwise. Adding or removing a node only remaps the keys in that node's arcs — about K/N — instead of nearly everything like modulo-n, so there's no thundering herd on membership change. I'd use virtual nodes, ~150 per physical node, so the arcs interleave and load stays even. For durability I'd replicate each key onto the next R nodes clockwise, Dynamo-style, so a node failure just shifts reads to the next replica."

Then handle the follow-ups before they ask:

| Follow-up | Answer |
|---|---|
| "A node dies mid-request?" | Its arcs are picked up by the next node clockwise; with replication factor R the data is already there. |
| "Hot keys?" | Vnodes fix *node* imbalance, not *key* hotspots. Cache hot keys separately or salt them (`key#1`…`key#n` + scatter-gather). |
| "When would you NOT use it?" | Tiny clusters where a full reshuffle is cheap; range scans (you want ordered partitioning, like Bigtable/HBase, or hash slots like Redis Cluster); when membership basically never changes. |
| "How many virtual nodes?" | Trade-off: more = smoother, but a bigger ring to gossip. 100–200 is the usual band; Cassandra ships 16. |
| "Which hash function?" | Uniformity matters, crypto doesn't — MD5 is fine for placement, MurmurHash3/xxHash in production for speed. It must be stable across restarts and languages. |

## Gotchas worth knowing

- **Data still moves.** Consistent hashing minimizes movement; it doesn't eliminate it. A dying node still sheds `K/N` keys, and someone has to serve them — plan the replication, not just the ring.
- **`MessageDigest` is not thread-safe.** The implementation above creates one per hash call — correct and simple, slightly wasteful. In production use a `ThreadLocal` digest or a non-crypto hash like xxHash.
- **The ring is the easy part.** Real systems spend their complexity on membership (who's in the ring — gossip, like Cassandra/SWIM), failure detection, and data transfer. The ring just tells you *where* things belong.

Build the 50-line ring once, test the two structural properties, and "design a distributed cache" stops being a scary question — it's a story about a ring, some virtual nodes, and a 3 AM node death that nobody noticed.
