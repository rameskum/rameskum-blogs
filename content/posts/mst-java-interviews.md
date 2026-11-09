---
title: 'Minimum Spanning Trees in Java Interviews: Kruskal vs Prim, and When the Interviewer Asks for the Second-Best'
date: 2026-11-09T11:00:00+05:30
draft: false
ShowToc: true
description: >-
  Connect every node with n-1 edges at the lowest total cost — that's the
  whole problem, and two algorithms solve it from opposite directions.
  Kruskal with Union-Find, Prim with a priority queue, the two properties
  that make both correct, the visited-set trap that silently inflates
  Prim's answer, and the second-best MST follow-up, verified against a
  brute-force enumeration.
tags:
  - interview
  - algorithms
  - java
  - graphs
categories: article
keywords:
  - minimum spanning tree java
  - kruskal vs prim
  - second best mst
  - union find path compression
  - prim visited set bug
---

The setup is four cities and six possible roads, each with a cost. Connect all four cities so that every city is reachable from every other, and spend as little as possible. The answer is always exactly n-1 roads — fewer can't connect everything, more must contain a cycle, and a cycle means you're paying for a road you don't need. A minimum spanning tree is the cheapest such set.

It's a small problem with a long tail in interviews. The basic version ("compute the MST weight") is a warm-up. The follow-ups are where candidates are separated: disconnected input, equal weights, a second-best MST, and the one Prim bug that produces a plausible-looking wrong number. This post builds all of it with running code — Kruskal, Prim, the second-best MST — and every number below is measured against a brute-force enumeration of every spanning tree, not asserted.

The running example, a 4-node graph:

```
0--1 (1)   1--2 (2)   2--3 (1)
0--2 (4)   1--3 (3)   0--3 (5)
```

The MST is edges 0-1, 1-2, 2-3, weight 1 + 2 + 1 = **4**. The second-best MST weighs **5**. Keep those two numbers in mind — everything below is checked against them.

## Two properties run the whole topic

Both algorithms are just these two facts, applied greedily.

**Cut property.** Split the nodes into any two groups. The cheapest edge crossing between them belongs to *some* MST. That's Prim's engine: grow one tree, and at every step add the cheapest edge crossing the cut between the tree and everything else.

**Cycle property.** Take any cycle. Its heaviest edge belongs to *no* MST — removing it keeps the graph connected and strictly lowers the cost. That's Kruskal's engine: consider edges cheapest-first, and skip any edge that closes a cycle.

If you remember nothing else, remember this: Kruskal rejects edges that form cycles, Prim accepts edges that cross the cut. Same answer, opposite directions. (Both from CLRS chapter 23, where the proofs live.)

## Kruskal: sort, then Union-Find

Kruskal is the easier one to write correctly, because the hard part — "does this edge form a cycle?" — is exactly what Union-Find answers. Two nodes are in the same set if and only if they're already connected by chosen edges, so an edge forms a cycle exactly when its endpoints are already in the same set.

```java
public record Edge(int u, int v, int w) {
    public Edge {
        if (u < 0 || v < 0 || u == v) throw new IllegalArgumentException("bad edge: " + u + "-" + v);
    }
}

class UnionFind {
    private final int[] parent;
    private final int[] rank;

    UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]); // path compression
        return parent[x];
    }

    /** @return true if a union happened (x and y were in different sets). */
    boolean union(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;
        if (rank[rx] < rank[ry]) {
            parent[rx] = ry;
        } else if (rank[rx] > rank[ry]) {
            parent[ry] = rx;
        } else {
            parent[ry] = rx;
            rank[rx]++;
        }
        return true;
    }
}
```

```java
public static Result kruskal(int n, List<Edge> edges) {
    List<Edge> sorted = new ArrayList<>(edges);
    sorted.sort(Comparator.comparingInt(Edge::w));
    UnionFind uf = new UnionFind(n);
    List<Edge> taken = new ArrayList<>();
    int weight = 0;
    for (Edge e : sorted) {
        if (uf.union(e.u(), e.v())) {
            taken.add(e);
            weight += e.w();
        }
    }
    return new Result(taken, weight, n - taken.size());
}
```

Complexity is O(E log E): the sort dominates, and Union-Find with path compression plus union by rank is effectively O(1) per operation — amortized inverse Ackermann, α(V), which is below 5 for any input you'll ever see. The naive alternative — a DFS per edge to check for a cycle — is O(E·V) and is the version that times out. If the interviewer asks why you didn't just DFS, that's the answer: sorting once beats searching per edge.

One detail worth noticing: `n - taken.size()` is the number of connected components. On a connected graph that's 1 and `taken` has n-1 edges. On a disconnected graph Kruskal doesn't fail — it returns a minimum spanning *forest*, one tree per component. Whether that's the right answer depends on the problem: if the question guarantees a connected graph, a forest is your signal that the input violated the guarantee.

## Prim: grow one tree, and mind the stale entries

Prim starts from one node and grows a single tree, always taking the cheapest edge that crosses the cut to the outside. The natural data structure is a min-heap of candidate edges, keyed by weight.

```java
public static Result prim(int n, List<List<Edge>> adj) {
    boolean[] visited = new boolean[n];
    List<Edge> taken = new ArrayList<>();
    int weight = 0;
    int components = 0;
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    for (int start = 0; start < n; start++) {
        if (visited[start]) continue;
        components++;
        visited[start] = true;
        for (Edge e : adj.get(start)) {
            int other = e.u() == start ? e.v() : e.u();
            pq.offer(new int[]{e.w(), other, start});
        }
        while (!pq.isEmpty()) {
            int[] top = pq.poll();
            int w = top[0], node = top[1], from = top[2];
            if (visited[node]) continue; // stale entry — node already in the tree
            visited[node] = true;
            taken.add(new Edge(from, node, w));
            weight += w;
            for (Edge e : adj.get(node)) {
                int other = e.u() == node ? e.v() : e.u();
                if (!visited[other]) pq.offer(new int[]{e.w(), other, node});
            }
        }
    }
    return new Result(taken, weight, components);
}
```

Complexity is O(E log V) with a binary heap: every edge is pushed once and popped at most once. (With a Fibonacci heap it's O(E + V log V), which is the answer if the interviewer pushes on complexity — and the reason nobody implements it in an interview.)

The `if (visited[node]) continue;` line is the whole trap, and it's worth understanding exactly why. The heap is *lazy*: when a node is first reached over an expensive edge, an entry for it is pushed; when it's later reached over a cheaper edge, a second entry is pushed, and the cheaper one pops first and finalizes the node. The expensive entry is still sitting in the heap. Without the visited check, it pops later and its weight gets added a second time.

Here's the concrete failure, on the diamond graph:

```
0--1 (1)   0--2 (5)   1--2 (2)   1--3 (4)   2--3 (1)
```

Correct Prim gives 4 (edges 0-1:1, 1-2:2, 2-3:1). The version without the visited check gives **13**: node 2 is first reached via 0-2:5, then finalized via 1-2:2, and the stale (5, 2) entry still adds 5; node 3 is finalized via 2-3:1 and the stale (4, 3) entry still adds 4. 1 + 2 + 1 + 5 + 4 = 13. Both versions compile, both terminate, both return a number — and the wrong one is wrong by a margin no test with distinct, well-separated weights would necessarily catch if the test only checked that the answer was "reasonable". The test that catches it is an exact-weight assertion on a graph with a cheap path that supersedes an expensive first reach. Write that test.

The outer loop over `start` is the disconnected-graph handling: when the heap empties but nodes remain unvisited, Prim restarts from the next one, building a forest instead of a tree. Same `components` count as Kruskal's.

## Kruskal or Prim — which one to write

| | Kruskal | Prim |
| --- | --- | --- |
| Input shape | Edge list | Adjacency list |
| Core structure | Union-Find | Priority queue |
| Complexity | O(E log E) | O(E log V) with a binary heap |
| Natural on | Sparse graphs (sort is cheap) | Dense graphs (heap beats sorting E edges) |
| Disconnected input | Forest, naturally | Forest, with the restart loop |
| Classic bug | Forgetting path compression | Forgetting the visited check on pop |

In an interview, the input shape usually decides for you: an edge list points at Kruskal, an adjacency list at Prim. Know both, because the follow-up is often "now do it the other way", and because the second-best MST below is easiest to explain from Kruskal's tree.

Two more facts that come up as follow-ups. **Negative weights are fine** — MSTs don't care about them, unlike Dijkstra. A graph with a -1 edge just has a cheaper tree (verified: edges -1, 2, 3 give MST weight 1). **Equal weights mean the MST isn't unique.** A triangle with all edges weight 1 has three spanning trees, all weighing 2 — which is exactly why the second-best question needs a careful definition.

## The follow-up: the second-best MST

"Now find the cheapest spanning tree that isn't the MST." The definition matters before the algorithm does: second-best means strictly heavier than the MST. If equal-weight trees count, the question is trivial whenever any edge weight repeats.

The algorithm is a single edge swap. For every edge *not* in the MST, adding it to the tree creates exactly one cycle. Swapping out the heaviest edge on that cycle gives the cheapest tree that uses the new edge. Take the minimum over all non-tree edges, and that's the second-best MST.

```java
public static OptionalInt secondBest(int n, List<Edge> edges) {
    Result mst = kruskal(n, edges);
    if (mst.components() != 1) return OptionalInt.empty();
    List<List<Edge>> tree = adjacency(n, mst.edges());
    // Multiset of tree edges: a plain Set breaks on parallel edges
    // with the same endpoints and weight.
    Map<Edge, Integer> inTree = new HashMap<>();
    for (Edge e : mst.edges()) inTree.merge(e, 1, Integer::sum);
    int best = Integer.MAX_VALUE;
    for (Edge e : edges) {
        Integer count = inTree.get(e);
        if (count != null && count > 0) {
            inTree.put(e, count - 1); // consume one copy of this tree edge
            continue;
        }
        int[] maxOnPath = maxAndSecondMaxOnPath(tree, e.u(), e.v(), e.w());
        int swapOut = maxOnPath[0] == e.w() ? maxOnPath[1] : maxOnPath[0];
        if (swapOut == Integer.MIN_VALUE) continue; // no lighter edge to swap out
        best = Math.min(best, mst.weight() - swapOut + e.w());
    }
    return best == Integer.MAX_VALUE ? OptionalInt.empty() : OptionalInt.of(best);
}
```

The subtle line is `swapOut`: if the non-tree edge weighs the same as the heaviest edge on its cycle, swapping that edge out gives another tree of *equal* weight — another MST, not the second-best. So in that case, swap out the heaviest edge on the cycle that is strictly lighter than the new edge. That's what `maxAndSecondMaxOnPath` returns: the heaviest edge on the unique tree path between the endpoints, and the heaviest edge on that path strictly lighter than the new edge's weight (a BFS over the tree, since a tree has exactly one path between any two nodes).

Walk it on the running example. The MST (weight 4) uses 0-1:1, 1-2:2, 2-3:1. Non-tree edges:

- **0-2:4** — cycle 0-1-2, heaviest edge 1-2:2. Swap: 4 - 2 + 4 = 6.
- **1-3:3** — cycle 1-2-3, heaviest edge 1-2:2. Swap: 4 - 2 + 3 = **5**.
- **0-3:5** — cycle 0-1-2-3, heaviest edge 1-2:2. Swap: 4 - 2 + 5 = 7.

Minimum is 5, which is the second-best MST: edges 0-1:1, 2-3:1, 1-3:3. And the equal-weight case behaves as defined: the all-1 triangle has no strictly second-best tree, so the method returns empty — every spanning tree weighs 2.

Why trust a swap argument instead of a proof sketch? Because it was checked against brute force: enumerate every (n-1)-edge subset, keep the connected ones, and collect the distinct weights. On the 4-node graph the distinct spanning-tree weights are 4, 5, 6, 7, 8, 9, 10, 11, 12 — MST 4, second-best 5, exactly as the swap algorithm says. On a 6-node, 9-edge graph the weights run 9 through 19, and the swap algorithm's 10 matches the brute force's second value. The swap method is O(E·V) in the naive per-edge BFS form — fine for an interview, and the honest answer if asked to do better is that the max-edge-on-path queries can be answered with binary lifting in O(log V) each.

## Three problems to practice

**1. MST weight, edge list in.** The warm-up. Kruskal, as above; assert the weight is 4 on the running example and 9 on the 6-node graph (edges 0-1:4, 0-2:3, 1-2:1, 1-3:2, 2-3:4, 2-4:2, 3-4:5, 3-5:1, 4-5:3).

**2. The graph might be disconnected.** Return the forest: the number of components and the total weight. Edges 0-1:1 and 2-3:1 on 4 nodes give 2 components, weight 2. The trap is a Prim that assumes connectivity and silently returns a partial tree, or a Kruskal that returns `n - 1` edges without checking.

**3. Second-best MST.** The swap algorithm above. Test it on the running example (expect 5), on the 6-node graph (expect 10), and on the all-equal triangle (expect empty). If you only test the happy path, you'll never discover whether your equal-weight handling is right — and that's the case the interviewer will ask about next.

## The trap checklist

- **Prim without the visited check on pop** — stale heap entries inflate the total (13 instead of 4 on the diamond). The most common silent bug in this topic.
- **Disconnected input** — an MST doesn't exist; return the forest and say so, don't return a partial tree as if it were one.
- **Equal weights** — the MST isn't unique; define second-best as strictly heavier before you write code, and test the all-equal case.
- **Negative weights** — legal for MSTs, illegal for Dijkstra's assumptions. Don't let a Dijkstra reflex reject them.
- **0-indexed vs 1-indexed input** — the classic off-by-one. Size your Union-Find and adjacency list to the actual max node id, and convert once at the boundary.
- **Sorting a copy** — Kruskal sorts its input; sort a copy, or the caller's edge list comes back reordered.

## Sources

- Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, chapter 23 — the cut property, the cycle property, and the correctness arguments for both algorithms.
- All 11 tests run on JDK 25 (JUnit 6.0.3), each run twice, all passing: Kruskal and Prim on the running example (weight 4), negative weights (weight 1), Prim vs Kruskal agreement on the 6-node graph (weight 9), second-best 5 and 10 confirmed against brute-force enumeration of every spanning tree, no second-best on the all-equal triangle, forest handling on disconnected input, the Prim visited-set trap (buggy 13 vs correct 4), and input-order independence for Kruskal.
