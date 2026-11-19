# LC 662 — Maximum Width of Binary Tree

**Difficulty:** Medium · **Sheets:** Striver SDE | NeetCode (Trees)
**Pattern:** Level-order traversal carrying a positional index — width from the index span, not the node count

---

## 1. Problem

Return the **maximum width** of a binary tree, where the width of a level is the number of nodes
between its leftmost and rightmost non-null nodes — **counting the null slots between them that a
complete binary tree would have at that depth.**

```
root = [1,3,2,5,null,null,9,6,null,7]   →  7
```

Constraints: `1 ≤ n ≤ 3000`, and **the answer is guaranteed to fit in a 32-bit signed integer**.

> That last guarantee is not decoration — it's what makes §4 work.

---

## 2. ⚠️ Reading the definition correctly

**Width is not `queue.size()`.** A quick read suggests "how many nodes are on this level", and that
is a clean, correct solution to a different question. Most small test cases won't distinguish them.

The statement measures each level against the **slots a perfect tree would have at that depth.**

### The example, drawn

`[1,3,2,5,null,null,9,6,null,7]` decodes to:

```
        1
      /   \
     3     2
    /       \
   5         9
  /         /
 6         7
```

Level 3 holds only `6` and `7`. A perfect tree would have **8** slots there. Which two are occupied?
Read the root-to-node path as binary, left = 0, right = 1:

- **6**: `1 → 3(L) → 5(L) → 6(L)` = `LLL` = slot **0**
- **7**: `1 → 2(R) → 9(R) → 7(L)` = `RRL` = slot **6**

```
level 3 slots:    0     1     2     3     4     5     6     7
                  6     ·     ·     ·     ·     ·     7     ·
                  └───────────── 7 wide ─────────────┘
```

Slots 0…6 inclusive = **7**. Slot 7 is past the rightmost node, so it isn't counted — only gaps
*between* the two end-nodes count.

| level | nodes | slots occupied | width |
|---|---|---|---|
| 0 | 1 | 0 | 1 |
| 1 | 3, 2 | 0, 1 | 2 |
| 2 | 5, 9 | 0, 3 | **4** |
| 3 | 6, 7 | 0, 6 | **7** |

**Level 2 is the one that proves the definition is doing work:** two nodes, width 4, because `5` sits
at `LL` and `9` at `RR` with two phantom slots between them. Any `queue.size()` solution reports 2.

---

## 3. The index trick

> ### 🔑 The fact
> **Give each node the index of the slot it would occupy in a perfect tree. Then the width of a level
> is `lastIndex − firstIndex + 1`.**

And the index propagates with one rule:

> **A node at slot `i` has children at slots `2i` and `2i + 1`.**

```
level 2 slots:     0        1        2        3
                   |\       |\       |\       |\
level 3 slots:     0  1     2  3     4  5     6  7
```

A node at slot 1 has children at **2 and 3** — not 1 and 2. (`i` and `i+1` happens to be right only
for `i = 0`, which is why a root-only trace won't catch the error.)

**The formula is level-independent**, and that's the elegant part: each level has exactly twice as
many slots as the one above, and `2i` scales by exactly that factor. You never need to know what
depth you're at.

*(Rooting at index 1 instead of 0 gives `2i` / `2i+1` with the root at 1 — same thing shifted. Pick
one and stay with it.)*

---

## 4. ⚠️ The index grows exponentially — and what that does and doesn't break

Apply `2i + 1` down a right-only chain:

```
depth 0:  0
depth 1:  1
depth 2:  3
depth 3:  7
depth d:  2^d − 1
```

It **doubles** every level. With `n ≤ 3000`, a skewed tree reaches **depth 3000**, so the index
reaches `2³⁰⁰⁰` — a number with about 900 digits.

| | holds up to | buys you |
|---|---|---|
| `int` | ~2³¹ | depth ~31 |
| `long` | ~2⁶³ | depth ~63 |
| needed | 2³⁰⁰⁰ | — |

**No fixed-width type is big enough.** The fix is not a wider type.

### ⚠️ But in Java, overflow does NOT break the answer

This is counter-intuitive and worth being precise about.

Java specifies `int` arithmetic as **two's-complement wraparound** (JLS 15.18.2), so `2i` and `2i+1`
are *exact modulo 2³²*. By induction from the root (`0 ≡ 0`), every stored index is congruent to its
true index mod 2³². Therefore:

```
(last − first) computed in wrapped int arithmetic  =  (true difference) mod 2³²
```

and since the answer is **guaranteed to fit in a 32-bit signed int**, the true difference satisfies
`0 ≤ d < 2³¹` — so the wrapped result *is* `d`, exactly.

Verify in miniature with 4-bit signed ints (range −8…7, mod 16). True indices at depth 3 are 7…14,
which store as 7 and −2:

```
last − first   =  −2 − 7  =  −9   →  wraps to  7
true width − 1 =  14 − 7  =        7            ✓
```

**And a pure chain can't break it either** — one node per level means `first == last`, so the width
is 1 whatever the index wrapped to.

### So why normalise at all?

Three reasons, none of them about Java correctness:

- **C++**: signed overflow is **undefined behaviour**, not wraparound. The identical code is UB
  there, which is why editorials normalise or use `unsigned`.
- **Python**: no overflow, but the indices genuinely become 900-digit integers. Arbitrary-precision
  arithmetic turns an O(n) algorithm slow.
- **Readability**: correctness-by-modular-arithmetic requires the proof above. Correctness-by-small-
  numbers requires nothing.

> **The interview line:** *"It's actually fine in Java, because two's-complement subtraction preserves
> the difference and the answer fits in an int — but I'd normalise per level so correctness doesn't
> depend on that argument."*

### How to normalise

At the **start** of each level, read the leftmost node's index and subtract it from every index in
that level. The width is unchanged (both endpoints shift by the same constant), and the indices
restart from 0 at every level instead of compounding from the root.

**And notice what falls out:** after subtracting the base, the first index is **always 0** — so
`width = lastIndex + 1` and the `firstIndex` variable is dead.

---

## 5. Solution

```java
class Solution {
    public int widthOfBinaryTree(TreeNode root) {
        int maxWidth = 0;
        Deque<Pair> queue = new ArrayDeque<>();
        queue.offer(new Pair(root, 0));

        while (!queue.isEmpty()) {                  // one iteration = one LEVEL
            int size = queue.size();                // capture BEFORE enqueuing children
            int base = queue.peek().index();        // this level's leftmost index
            int lastSlot = 0;

            for (int i = 0; i < size; i++) {        // one iteration = one NODE
                Pair p = queue.poll();
                TreeNode curr = p.node();
                lastSlot = p.index() - base;        // normalised: leftmost becomes 0

                if (curr.left  != null) queue.offer(new Pair(curr.left,  2 * lastSlot));
                if (curr.right != null) queue.offer(new Pair(curr.right, 2 * lastSlot + 1));
            }

            maxWidth = Math.max(maxWidth, lastSlot + 1);   // first is 0 by construction
        }
        return maxWidth;
    }
}

record Pair(TreeNode node, int index) {}
```

**O(n) time · O(w) space**, `w` = maximum width.

### This is the LC 102 chassis with two additions

Same `size = queue.size()` capture, same inner loop, same per-node child enqueue. What's new:

1. each queue entry carries an **index** alongside the node, and
2. `peek()` before the inner loop reads the level's base.

**`peek()`, not `poll()`** — the base must be read while the first entry is still in the queue, since
the inner loop needs to process it like any other node.

### `record` over a hand-written Pair class

```java
record Pair(TreeNode node, int index) {}     // one line
```

replaces ~15 lines of constructor and getters, and generates `equals`/`hashCode`/`toString` too.
Accessors are `p.node()` and `p.index()`. LeetCode runs Java 21.

---

## 6. Common pitfalls

### 🚩 Assigning `maxWidth` instead of maxing it

```java
maxWidth = lastSlot - firstSlot + 1;      // 🚩 overwrites every level
```

After the loop this holds the **last** level's width, not the largest. It passes the sample only
because there the widest level happens to also be the last.

Counterexample:

```
        1
      /   \
     2     3
    /
   4
```

Level 1 has width 2; level 2 has only node 4 at slot 0, width 1. **Returns 1; answer is 2.**

> When the aggregate is a maximum over levels, the update must be a `Math.max`, not an assignment.
> Any tree whose widest level isn't the bottom one exposes it.

### 🚩 Reading the base but never subtracting it

```java
int base = queue.peek().index();                            // captured ✓
...
queue.offer(new Pair(curr.left, 2 * p.index()));            // 🚩 raw index, not normalised
```

The width arithmetic is right — `last − first` cancels the base — but the children are enqueued from
the **raw** index, so the growth still compounds from the root exactly as before. Half the fix is no
fix. **Use the normalised value for both the width and the children.**

### 🚩 `i` and `i+1` for the children

Correct only for slot 0, so a root-only trace won't catch it. Each level is twice as wide; it's `2i`
and `2i+1`.

### 🚩 `queue.size()` as the width

The §2 misreading. Level 2 of the example has 2 nodes and width 4.

### 🚩 `poll()` instead of `peek()` for the base

Consumes the leftmost node before the inner loop, so it's never processed and its children never
enqueued.

### 🚩 A leaf guard on the root

```java
if (root.left == null && root.right == null) return 1;      // 🚩
```

NPEs on a null root, and is dead anyway — a single node flows through the main loop correctly
(`size = 1`, `base = 0`, `lastSlot = 0`, width 1).

### 🚩 Per-level variables declared outside the loop

`firstSlot` / `lastSlot` are facts about one level. Declaring them in method scope works but invites
the reader to check whether stale values leak across iterations.

### 🚩 One variable doing two jobs

Using `lastSlot` as both *"this node's index"* (for the children) and *"the last index in the level"*
(for the width) is correct — the final assignment wins — but a reader has to verify that. A separate
`idx` local for the current node costs nothing.

---

## 7. DFS alternative

```java
class Solution {
    private int maxWidth = 0;
    private final List<Integer> firstAtDepth = new ArrayList<>();

    public int widthOfBinaryTree(TreeNode root) {
        dfs(root, 0, 0);
        return maxWidth;
    }

    private void dfs(TreeNode node, int depth, int index) {
        if (node == null) return;

        if (depth == firstAtDepth.size()) {
            firstAtDepth.add(index);               // first node reached at this depth
        }
        maxWidth = Math.max(maxWidth, index - firstAtDepth.get(depth) + 1);

        dfs(node.left,  depth + 1, 2 * index);     // LEFT first — mandatory
        dfs(node.right, depth + 1, 2 * index + 1);
    }
}
```

**`depth == firstAtDepth.size()`** is true exactly the first time you reach a depth — the same trick
as the DFS variant of LC 102.

⚠️ **Descending left before right is load-bearing**, not stylistic. It's what guarantees the first
node reached at a depth is the *leftmost* one. Swap the two calls and every baseline is wrong.

**O(n) time · O(h) space** (stack plus the depth list).

> ⚠️ **DFS cannot normalise.** A level isn't visited contiguously, so there's no moment at which you
> hold all of that level's indices. The raw indices therefore do grow to 2³⁰⁰⁰ — harmless in Java by
> §4's argument, genuinely slow in Python. That's a real reason to prefer BFS here.

### BFS vs DFS

| | BFS | DFS |
|---|---|---|
| space | **O(w)** — O(1) on a chain, ~n/2 on a complete tree | **O(h)** — O(log n) balanced, O(n) on a chain |
| leftmost-of-level | free: the first node dequeued | an invariant you must maintain (left-first) |
| normalisation | natural, at the top of each level | not possible |
| allocation | one wrapper per node | none per node |

Worst-case opposite, so neither dominates. **Prefer BFS** — fewer invariants, and normalisation has
an obvious home.

---

## 8. Dry run — the example tree

`root = [1,3,2,5,null,null,9,6,null,7]`, indices normalised per level.

| level | queue at top (node, idx) | base | normalised indices | children enqueued | width |
|---|---|---|---|---|---|
| 0 | (1, 0) | 0 | 1→0 | (3, 0), (2, 1) | `0 + 1` = **1** |
| 1 | (3, 0), (2, 1) | 0 | 3→0, 2→1 | (5, 0), (9, 3) | `1 + 1` = **2** |
| 2 | (5, 0), (9, 3) | 0 | 5→0, 9→3 | (6, 0), (7, 6) | `3 + 1` = **4** |
| 3 | (6, 0), (7, 6) | 0 | 6→0, 7→6 | none | `6 + 1` = **7** |

`maxWidth = 7` ✅ — and the per-level widths match the slot analysis in §2 exactly.

Level 1 is where the doubling is visible: node `2` sits at normalised index 1, so its **right** child
gets `2·1 + 1 = 3`, not 2. That gap of two empty slots is what makes level 2's width 4.

### Why the base is 0 at every level here

The leftmost node of each level in this tree happens to descend all-left from the root, so its raw
index is already 0. On a tree where the leftmost branch dies off, the base is non-zero and the
subtraction does visible work.

---

## 9. Edge cases

| case | expected | ✓ |
|---|---|---|
| single node | 1 | ✅ `size = 1`, `lastSlot = 0`, width 1 — no guard needed |
| perfect tree of depth d | 2^d | ✅ bottom level fully occupied |
| left-skewed chain | 1 | ✅ one node per level, `first == last` |
| right-skewed chain of 3000 | 1 | ✅ same — and the overflow is harmless (§4) |
| two nodes at opposite extremes of a deep level | large | ✅ the case the whole index trick exists for |
| leftmost branch truncated | correct | ✅ the base is non-zero, subtraction does real work |
| widest level is **not** the last | correct | ✅ only with `Math.max` (§6) |

---

## 10. Interview Q&A

**Q: Why can't you just use the number of nodes on a level?**
Because null slots *between* the end-nodes count. Level 2 of the example has 2 nodes and width 4.

**Q: How do you assign indices?**
Slot `i` → children at `2i` and `2i + 1`. Level-independent, because each level has twice the slots.

**Q: What breaks on a deep tree?**
The index doubles per level, so a 3000-node chain reaches 2³⁰⁰⁰. No fixed-width integer holds it.

**Q: Doesn't that make the answer wrong?**
In Java, no — `int` overflow is two's-complement wraparound, so every index is correct mod 2³², and
the *difference* of two such values is exact whenever the true difference fits in an int, which the
problem guarantees. In C++ it's undefined behaviour, and in Python it's a performance problem.

**Q: So why normalise?**
So correctness doesn't rest on that argument, and so the solution ports. Also it deletes a variable:
after subtracting the base, the first index is 0 and `width = last + 1`.

**Q: Complexity?**
O(n) time — every node enqueued and dequeued once. O(w) space for the queue: O(1) on a chain, ~n/2
on a complete tree.

**Q: DFS or BFS?**
Either, O(h) vs O(w) and worst-case opposite. BFS gets "leftmost of the level" for free and can
normalise; DFS needs left-first descent as an invariant and can't normalise.

**Q: Why `peek()` rather than `poll()` for the base?**
The leftmost node still has to be processed by the inner loop like any other.

---

## 11. Pattern

> **Index every node by the slot it would occupy in a perfect tree — `2i` and `2i+1` for the children
> — and a level's width is the span between its first and last index, not its node count.**

The transferable half: **when a structure's gaps matter as much as its contents, stop counting what's
there and start measuring positions.** The index is a coordinate in the *hypothetical complete*
structure, which is what makes absent nodes measurable at all.

And the secondary one, which generalises past trees:

> **When only a difference is needed, you can re-base the values at any time.** Shifting both
> endpoints by the same constant leaves the answer untouched and keeps the magnitudes small.

---

## 12. Takeaways

1. **Read the definition before the approach.** "Width" here counts phantom slots; a `queue.size()`
   solution is clean, correct, and answers a different question.
2. **Slot `i` → children `2i` / `2i+1`**, level-independent. `i` / `i+1` is right only at slot 0.
3. **Width = `last − first + 1`**, which is why only the *difference* ever matters.
4. **The index is exponential in depth**, not polynomial — `2^d`, not `d²`. A 3000-node chain reaches
   depth 3000.
5. **No integer type is big enough**, so the fix is re-basing, not a wider type.
6. **Java's `int` overflow is harmless here** — two's-complement subtraction preserves the difference
   and the answer fits in an int. Know the argument; don't rely on it.
7. **Normalise at the start of each level**, and use the normalised value for *both* the width and the
   children. Capturing the base without subtracting it is no fix at all.
8. **After normalising, `first` is always 0** — delete the variable, `width = last + 1`.
9. **A max over levels needs `Math.max`**, not an assignment. Test a tree whose widest level isn't the
   bottom one.
10. **`peek()`, not `poll()`,** to read a level's base.
11. **`record` for a two-field carrier.** One line instead of fifteen.
12. **In DFS, descending left before right is an invariant**, not a style choice.