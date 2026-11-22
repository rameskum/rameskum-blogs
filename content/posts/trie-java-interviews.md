---
title: 'Trie: The Tree Behind Autocomplete (and 3 Interview Questions That Use It)'
date: 2026-11-22T14:24:00-05:00
draft: false
ShowToc: true
description: >-
  Every keystroke in a search box is a prefix query, and the data structure
  that answers it in time proportional to the keystroke — not to the size of
  the dictionary — is the trie. This post builds it once in Java, then uses
  the same trie for three LeetCode questions: 208 (build it), 211 (wildcard
  search, the spell-checker pattern), and 212 (Word Search II, trie plus
  backtracking). Eleven tests, run twice, every number below measured.
tags:
  - java
  - algorithms
  - interview-prep
  - data-structures
categories: article
keywords:
  - trie java
  - prefix tree leetcode 208
  - trie autocomplete java
  - leetcode 211 word dictionary
  - leetcode 212 word search ii
  - trie vs hashmap
---

Type `app` into a search box and the suggestions appear before you finish the keystroke. The dictionary behind that box might hold a million words, but the suggestions do not take a million steps — they take four, one per letter you typed. That is the whole trie: a tree where the path from the root *is* the word, so every operation costs the length of its argument and nothing else. No operation in this post depends on how many words are stored.

The trie is the last classic data structure this series had not covered. Two pointers, sliding window, cyclic sort, minimum spanning trees, Floyd, Top-K heaps, and HashMap all have posts; the trie is the one that answers a question none of those can: *what starts with this prefix?* This post builds it once, then uses the same code for three LeetCode questions — 208 (build it), 211 (wildcard search), and 212 (Word Search II) — and measures the HashMap comparison instead of asserting it.

## The structure: one node per character, one flag

A trie node holds children and a boolean. That is the entire structure:

```java
static class Node {
    Node[] children = new Node[26];
    boolean isWord;
    String word; // full word, set only at a word's end node (used by 212)
}
```

The children are a 26-slot array because the problems guarantee lowercase English letters — state that assumption in the interview rather than silently baking it in. The `isWord` flag is the only thing separating a word from a prefix, and it is the first thing the interviewer checks: after inserting `apple`, is `app` a word? No — `app` is a path through the trie, not a word in it, until you insert it. The third field, `word`, stores the full word at its end node; only 212 needs it, and the reason will be obvious when the DFS reaches a word and has to report it without rebuilding the path.

Insert walks the characters, creating nodes as needed, and sets the flag at the end. Search and `startsWith` share one walk and differ by exactly one clause:

```java
public void insert(String word) {
    Node node = root;
    for (int i = 0; i < word.length(); i++) {
        int c = word.charAt(i) - 'a';
        if (node.children[c] == null) {
            node.children[c] = new Node();
        }
        node = node.children[c];
    }
    node.isWord = true;
    node.word = word;
}

/** Walks to the node for {@code prefix}; null if the prefix is absent. */
private Node walk(String prefix) {
    Node node = root;
    for (int i = 0; i < prefix.length(); i++) {
        node = node.children[prefix.charAt(i) - 'a'];
        if (node == null) {
            return null;
        }
    }
    return node;
}

public boolean search(String word) {
    Node node = walk(word);
    return node != null && node.isWord;
}

public boolean startsWith(String prefix) {
    return walk(prefix) != null;
}
```

That shared walk is the design made visible: `search` asks "did the walk end on a word?", `startsWith` asks "did the walk succeed at all?" If you write them as two separate traversals, the interviewer will ask why.

| Operation | Time | Depends on number of words? |
| --- | --- | --- |
| `insert` | O(L) | No |
| `search` | O(L) | No |
| `startsWith` | O(L) | No |

L is the length of the word or prefix. The space is the honest weakness, and it is measured below: a 26-slot array per node means one long word allocates 26 pointers per character, most of them null.

## LeetCode 208: build it

The statement: implement a `Trie` supporting `insert(word)`, `search(word)` (exact word only), and `startsWith(prefix)` (any inserted word with that prefix). Words are lowercase English letters, length 1 to 2000, at most 30,000 calls in total. The code above *is* the solution. The example from the statement is the test:

```java
Trie trie = new Trie();
trie.insert("apple");
assertTrue(trie.search("apple"));   // [null, null, true, ...]
assertFalse(trie.search("app"));    // ... false ...
assertTrue(trie.startsWith("app")); // ... true ...
trie.insert("app");
assertTrue(trie.search("app"));     // ... null, true]
```

The output sequence `[null, null, true, false, true, null, true]` is the whole interview in miniature: the two `search("app")` calls returning different answers is what the `isWord` flag exists for. The test suite also checks the implementation against a `HashSet` on 500 seeded random words and 500 random queries — `search` must agree with `set.contains`, `startsWith` must agree with "any word has this prefix" — so a wrong trie cannot hide behind the one example.

### How much does sharing save? Count the nodes

The prefix-sharing claim is usually drawn, not measured. Count nodes instead. Insert `apple` (root + five nodes), then `app`, then `apply`, then `banana`:

```java
trie.insert("apple");
assertEquals(6, trie.nodeCount()); // root + a,p,p,l,e
trie.insert("app"); // a prefix of apple: no new nodes, only the flag
assertEquals(6, trie.nodeCount());
trie.insert("apply"); // shares a,p,p,l; adds only y
assertEquals(7, trie.nodeCount());
trie.insert("banana"); // shares nothing with the a-branch
assertEquals(13, trie.nodeCount()); // root shared; b,a,n,a,n,a = 6 new
```

Three words sharing a prefix cost seven nodes; a fourth word on a different branch costs six more. That is the memory story in exact numbers: sharing is free where prefixes overlap and full price where they do not.

## Trie vs HashMap: the question the interviewer actually asks

"Why not a `HashSet`?" is the first follow-up, and the honest answer has two halves. For exact lookup the HashSet wins: `search` is O(L) either way (hashing a word of length L costs L), but the set has the better constant factor and far less code. For prefix queries the HashSet cannot compete: it has no prefix index, so `startsWith` scans every stored word — O(N·L) for N words. The trie answers in O(L) because the structure *is* the prefix relation.

Measured, not asserted. Ten thousand words (plus one planted `prelude` so the prefix exists), one prefix query:

```
prefix "pre": trie steps = 3, HashSet words scanned before first hit = 6659 (of 10001 stored)
```

The trie touched 3 nodes — one per letter of `pre`. The HashSet scan stopped at the first word starting with `pre`, and it still examined 6,659 of the 10,001 words to get there. And that is the *generous* case for the set: the scan stops at the first hit, while a real prefix query (autocomplete, "give me all of them") cannot stop at all. The reverse is equally true and worth saying out loud: for exact lookup, use the set. The trie earns its space only when prefixes are the query.

The space side of the trade: the 26-slot array is fast and wasteful. A `HashMap<Character, Node>` per node, or a sorted array with binary search, trades time for memory. Say so in the interview; the interviewer is checking that you know the array is a choice, not a law. For the problems in this post, with a 26-letter alphabet and short words, the array is the right call — and the node counts above show exactly what it costs.

## Autocomplete: the walk plus a collect

Autocomplete is `startsWith` with the results attached: walk to the prefix node, then collect every word below it. The collect is a depth-first walk over the children in array order, which is why the results come out sorted:

```java
/** Autocomplete: every stored word below the prefix node, in order. */
public List<String> wordsWithPrefix(String prefix) {
    List<String> out = new ArrayList<>();
    Node node = walk(prefix);
    if (node != null) {
        collect(node, out);
    }
    return out;
}

private void collect(Node node, List<String> out) {
    if (node.isWord) {
        out.add(node.word);
    }
    for (Node child : node.children) {
        if (child != null) {
            collect(child, out);
        }
    }
}
```

The trace, executed, not hand-written:

```
wordsWithPrefix("app") = [app, apple, application, apply]
```

For the dictionary `[apple, app, application, apply, apt, banana, band]`, prefix `app` returns the four words in alphabetical order — the array-order DFS gives you sorting for free — and prefix `ban` returns `[banana, band]`. The test suite checks 200 random prefixes against a brute-force filter over the same word list, so the ordering claim is verified, not just illustrated. The cost is O(L + number of matches): the walk plus one visit per node below the prefix. Note what that means for a search box: a one-letter prefix can return the whole dictionary, and production autocomplete caps the results (usually 5–10) for exactly that reason — the trie answers fast, but the answer can still be enormous.

## LeetCode 211: wildcard search is spell-check

The statement: design a `WordDictionary` with `addWord(word)` and `search(word)`, where the search pattern may contain `.`, matching any single letter. Add `bad`, `dad`, `mad`; then `search("pad")` is false, `search("bad")` is true, `search(".ad")` is true, and `search("b..")` is true. Word length 1 to 25, at most 2 dots per search query, at most 10,000 calls.

This is the spell-check pattern: a dictionary plus a query with unknown letters. `addWord` is the plain trie insert; all the work is in `search`, which becomes a DFS because a dot fans out to every child at that level:

```java
public boolean search(String pattern) {
    return search(trie.root(), pattern, 0);
}

private boolean search(Trie.Node node, String pattern, int i) {
    if (node == null) {
        return false;
    }
    if (i == pattern.length()) {
        return node.isWord;
    }
    char c = pattern.charAt(i);
    if (c == '.') {
        for (Trie.Node child : node.children) {
            if (child != null && search(child, pattern, i + 1)) {
                return true;
            }
        }
        return false;
    }
    return search(node.children[c - 'a'], pattern, i + 1);
}
```

The state is "which node am I at, and which character of the pattern am I matching?" A letter follows one child; a dot tries every child and succeeds if any branch succeeds. Trace `search(".ad")` after adding `bad`, `dad`, `mad`: the root fans out to `b`, `d`, `m`; the `b` branch walks `a`, then `d`, reaches a word end, and returns true — `d` and `m` are never explored, because the recursion short-circuits on the first success.

The complexity is where the constraint matters. A dot branches up to 26 ways, so the worst case is O(26^m · L) for m dots — exponential in the dots, which is why the problem caps them at 2. With at most 2 dots the worst case is 676·L: small, and the only reason the DFS is acceptable. Say that in the interview: the algorithm is exponential in the number of wildcards, and the constraint is what saves it. The brute-force alternative — compare the pattern against every stored word of the same length — is O(N·L) per search, and the test suite checks 300 seeded random patterns against exactly that brute force.

## LeetCode 212: Word Search II — one DFS for all words

The statement: given an `m × n` board of letters and a list of words, return every word that can be formed by a path of adjacent cells (up, down, left, right), using each cell at most once per word. Board up to 12×12, up to 30,000 words, each word at most 10 letters, all unique. The example board returns `["oath", "eat"]` — `pea` and `rain` cannot be traced.

The brute force is one Word Search per word: W full grid searches, each up to O(R·C·4^L). The trie turns it into one search: put every word into the trie, then run a single backtracking DFS from each cell, carrying a pointer into the trie. Step to a neighbour only if its letter is a child of the current trie node — the moment a path spells a prefix no word has, the DFS stops. Shared prefixes (`oa…`, `oat…`) are explored once, for every word that uses them:

```java
public List<String> findWords(char[][] board, String[] words) {
    Trie trie = new Trie();
    for (String w : words) {
        trie.insert(w);
    }
    List<String> found = new ArrayList<>();
    for (int r = 0; r < board.length; r++) {
        for (int c = 0; c < board[r].length; c++) {
            dfs(board, r, c, trie.root(), found);
        }
    }
    return found;
}

private void dfs(char[][] board, int r, int c, Trie.Node node, List<String> found) {
    if (r < 0 || r >= board.length || c < 0 || c >= board[r].length) {
        return;
    }
    char ch = board[r][c];
    if (ch == '#' || node.children[ch - 'a'] == null) {
        return; // used cell, or no word has this prefix: prune
    }
    Trie.Node next = node.children[ch - 'a'];
    if (next.word != null) {
        found.add(next.word);
        next.word = null; // report once, even if another path spells it
    }
    board[r][c] = '#';
    for (int[] d : DIRS) {
        dfs(board, r + d[0], c + d[1], next, found);
    }
    board[r][c] = ch;
}
```

Three details do the work, and all three are worth naming in the interview:

1. **The prune is the first check.** `node.children[ch - 'a'] == null` ends the branch before recursing. The DFS never explores a path the trie does not contain — that is the entire advantage over the brute force.
2. **The cell is marked `'#'` and restored.** A cell is used at most once per word because the path marks it on entry and restores the letter on exit. Forgetting the restore is the classic bug: the second DFS from a neighbouring cell then sees a hole.
3. **A found word's marker is cleared.** `next.word = null` after reporting means the same word is never reported twice, even if two different paths spell it. This is also why the `word` field exists on the node: the DFS reports the word it is standing on without rebuilding the path from the root.

The example, executed: `findWords(example) = [oath, eat]`. And the cross-check that makes the post honest: 40 seeded random 4×4 boards with random word lists, where the trie result is compared against the brute force — a separate Word Search I run for every word. They agree on all 40 boards. The worst-case complexity is still O(M·N·4^L) — the trie prunes, it does not change the bound — but with 30,000 words the brute force multiplies that by 30,000 and the trie does not.

## Delete: the follow-up, and the bug it caught

`delete(word)` is the follow-up that exposes whether the structure is understood: remove the word-end flag, then prune nodes that become childless and are not themselves word ends, on the way back up. Deleting `app` must leave `apple` untouched — the shared nodes stay — while deleting `apple` must free the `e` node it alone needed:

```java
public boolean delete(String word) {
    if (!search(word)) {
        return false; // not stored: nothing to delete
    }
    delete(root, word, 0);
    return true;
}

/** Returns true if this node can be pruned (not a word end, no children). */
private boolean delete(Node node, String word, int i) {
    if (i == word.length()) {
        node.isWord = false;
        node.word = null;
        return hasNoChildren(node);
    }
    Node child = node.children[word.charAt(i) - 'a'];
    if (child == null) {
        return false;
    }
    boolean pruneChild = delete(child, word, i + 1);
    if (pruneChild) {
        node.children[word.charAt(i) - 'a'] = null;
    }
    return !node.isWord && hasNoChildren(node);
}
```

The trap here is one this post caught by running the test instead of reading the code. The first version had the public method return the recursive helper's result directly — and the helper returns *whether this node can be pruned*, not *whether the word was deleted*. Deleting `apple` from a trie that still holds `apply` returns false from the helper, because the `l` node still has the `y` child — even though the word was deleted. The test failed, the code was wrong, and the fix is the two-line guard at the top: check presence first, then prune. Two walks, still O(L), and the return value finally means what its name says. That is the kind of bug that passes a code review and fails a test — which is why the test exists.

## What to say in the interview

The pattern to name is "prefix" — any problem about words sharing prefixes, completing a prefix, or searching many words at once is a trie problem until proven otherwise. The structure is one node per character plus the `isWord` flag; the operations are one shared walk; the complexity is O(L) per operation, independent of the dictionary size. The HashMap comparison is the answer to the first follow-up: exact lookup belongs to the set, prefix queries belong to the trie, and the measured gap for one prefix over ten thousand words was 3 steps against 6,659 scanned. The wildcard search is a DFS whose cost is exponential in the dots and saved by the 2-dot cap. Word Search II is one DFS guided by the trie instead of thirty thousand unguided ones. And if the interviewer asks for delete, prune on the way back up — and make the return value mean "deleted", not "prunable".

The trie connects to the rest of the series in one line: it is the data structure the [HashMap post](/posts/hashmap-internals-java-interview/) cannot replace, because a hash function destroys exactly the information the trie preserves — the order of the characters. Two pointers and sliding window ([playbook](/posts/two-pointer-playbook-java/)) answer questions about positions in a sequence; the trie answers questions about prefixes in a dictionary. Different question, different structure.

## Verification note

Problems 208, 211, and 212 verified against their LeetCode statements (operations, examples, and constraints, including the 2-dot cap on 211 and the 12×12 / 30,000-word limits on 212) before drafting. Eleven JUnit 6 tests, run twice on Temurin JDK 25.0.5.1, 11/11 both times: the 208 and 211 example sequences exactly as stated, the 212 example board returning `[oath, eat]`, and seeded property checks against brute-force references — 500 search/startsWith queries against a `HashSet`, 200 autocomplete prefixes against a filter, 300 wildcard patterns against a matcher, and 40 random boards against a per-word DFS. The measured numbers in the post are the executed traces: `wordsWithPrefix("app") = [app, apple, application, apply]`, node counts 6 / 6 / 7 / 13, and the prefix-cost comparison (3 trie steps vs 6,659 HashSet words scanned before the first hit, of 10,001 stored). The delete trap — the helper returning "prunable" instead of "deleted" — was caught by the test suite on the first run, not by review. No live LeetCode submission was made; the tests are the verification.
