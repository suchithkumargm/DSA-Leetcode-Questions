# LC 104 — Maximum Depth of Binary Tree

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** Recursive tree aggregation — a node's answer from its children's answers

---

## 1. Problem

Given the root of a binary tree, return its **maximum depth** — the number of nodes along the
longest path from the root down to the furthest leaf.

```
    3
   / \
  9  20          →  3
     / \
    15  7
```

`root == null` → `0`.

> ⚠️ **Depth is counted in nodes, not edges.** A single-node tree has depth **1**, not 0. Some
> textbooks define height in edges; LeetCode doesn't. The `+ 1` in the recursion is counting the
> current node.

---

## 2. The idea

> ### 🔑 The fact
> **A node's depth is `max(left subtree depth, right subtree depth) + 1`.**

That's the whole problem. The answer at any node is defined entirely in terms of the answers at its
children, which is exactly the shape recursion exists for: solve the children, combine, done. The
base case is `null → 0`, which makes a leaf resolve to `max(0, 0) + 1 = 1`.

This "a node's answer from its children's answers" shape is the chassis for a large family of tree
problems — see §9.

---

## 3. Solution — recursion ⭐ ship this

```java
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;

        int leftDepth  = maxDepth(root.left);
        int rightDepth = maxDepth(root.right);

        return Math.max(leftDepth, rightDepth) + 1;
    }
}
```

**O(n) time · O(h) space**, h = height of the tree.

### Why this is the default

No allocation, no data structure, five lines. It reads exactly like the recurrence in §2, which
means a reader verifies it by reading it rather than by tracing it.

The one-liner `return root == null ? 0 : Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;`
is equivalent. The named-variable form is easier to debug and no slower.

---

## 4. Complexity

| | | why |
|---|---|---|
| **Time** | **O(n)** | every node is visited exactly once |
| **Space** | **O(h)** | the **recursion call stack**, which holds one frame per level on the current path |

There is no data structure in the method — the space is entirely the call stack, and that's the
thing people forget to count.

| tree shape | h | stack space |
|---|---|---|
| balanced | ≈ log n | **O(log n)** |
| skewed (a chain) | n | **O(n)** ⚠️ |

> ⚠️ **The skewed case can genuinely stack-overflow.** A million-node chain means a million frames.
> That's not a theoretical worry — it's the standard follow-up on this problem.

> **"Recursive looks like O(1) space, but the call stack isn't free."** The same observation drives
> LC 25's recursion follow-up (O(n/k) frames, not O(1)) and LC 138's recursive variant. Whenever
> you offer a recursive solution, state its stack cost before being asked.

**Say "O(h), the height of the tree"** — `h` is the conventional symbol and it's immediately
recognised. Then give the two bounds; "O(h)" alone doesn't convey the range.

---

## 5. Iterative alternative — BFS counting levels

The answer to *"what if the tree is a million nodes in a single chain?"*

```java
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;

        Deque<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);
        int depth = 0;

        while (!queue.isEmpty()) {          // one iteration = one LEVEL
            int size = queue.size();        // capture BEFORE enqueuing children

            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();
                if (node.left  != null) queue.offer(node.left);
                if (node.right != null) queue.offer(node.right);
            }
            depth++;                        // one more level survived
        }
        return depth;
    }
}
```

**O(n) time · O(w) space**, w = maximum width.

### This is the LC 102 loop with one thing changed

Identical structure to level-order traversal — same `size` capture, same inner loop — except you
count passes instead of collecting values. **That loop is a chassis**: LC 102, 103, 107, 111, 199,
515, 637 and this are all the same while/for skeleton with a different line inside.

> **Why `size` must be captured before the inner loop:** children are enqueued during the loop, so
> the live `queue.size()` grows mid-pass. See the LC 102 notes.

### The tradeoff, stated properly

| | recursive | iterative BFS |
|---|---|---|
| space | O(h) — call stack | O(w) — queue |
| skewed tree (chain) | **O(n)** — can overflow | **O(1)** — one node per level |
| balanced tree | O(log n) | **O(n)** — bottom level is ~n/2 |
| speed | faster — no allocation | slower — object allocation, queue overhead |
| readability | mirrors the recurrence | more moving parts |

> **It isn't "iterative is safer" — it's a straight trade of h for w.** The two are worst-case
> opposite: a chain has h = n and w = 1; a complete tree has h = log n and w ≈ n/2.

### Which to prefer

**Recursive by default** — simpler, faster, and it reads like the definition.

**Iterative when depth could be large** — a deep or skewed tree, or a constrained stack. That's the
only real reason to switch, and "it's cleaner" is not it.

> There's also iterative **DFS** with an explicit stack of `(node, depth)` pairs. It keeps the O(h)
> profile while moving the frames onto the heap, so it survives depths that would overflow the call
> stack. Worth mentioning; rarely worth writing.

---

## 6. Common pitfalls

**Returning 0 for a single node.** Depth counts nodes. `max(0,0) + 1 = 1` — the base case is `null`,
not "leaf".

**Writing a leaf base case.** `if (root.left == null && root.right == null) return 1;` is redundant
— the `null → 0` case already produces it — and it forces a separate `root == null` guard anyway.
One base case is enough.

**Forgetting the `+ 1`.** Then you're returning the deeper subtree's depth and the root never counts.

**Claiming O(1) space for the recursive version.** It's O(h).

**Nested loops as an "iterative" approach.** A tree isn't indexable — there is no "left part" and
"right part" you can walk with counters. You can't loop over a structure you have to discover as you
go; you need a queue or an explicit stack.

**Mixing `offer`/`poll` with `add`/`remove`.** Both pairs work on an unbounded queue; mixing them is
just inconsistent. (`offer` returns false and `poll` returns null on failure; `add` and `remove`
throw.)

**Capturing `queue.size()` inside the loop condition** in the BFS version.

---

## 7. Dry run — recursion on `[3,9,20,null,null,15,7]`

```
    3
   / \
  9  20
     / \
    15  7
```

Calls resolve bottom-up:

| call | left | right | returns |
|---|---|---|---|
| `maxDepth(9)` | `maxDepth(null)` = 0 | `maxDepth(null)` = 0 | `max(0,0)+1 = 1` |
| `maxDepth(15)` | 0 | 0 | `1` |
| `maxDepth(7)` | 0 | 0 | `1` |
| `maxDepth(20)` | `maxDepth(15)` = 1 | `maxDepth(7)` = 1 | `max(1,1)+1 = 2` |
| `maxDepth(3)` | `maxDepth(9)` = 1 | `maxDepth(20)` = 2 | `max(1,2)+1 = **3**` |

Maximum stack depth during this: 3 frames (`3 → 20 → 15`), matching h = 3.

### BFS on the same tree

| pass | queue at top | size | enqueued | depth after |
|---|---|---|---|---|
| 1 | `[3]` | 1 | 9, 20 | 1 |
| 2 | `[9,20]` | 2 | 15, 7 | 2 |
| 3 | `[15,7]` | 2 | none | 3 |
| 4 | `[]` | — | exit | **3** ✅ |

---

## 8. Edge cases

| case | expected | ✓ |
|---|---|---|
| `root == null` | 0 | ✅ base case |
| single node | **1** | ✅ not 0 — depth counts nodes |
| left-skewed chain of n | n | ✅ recursive uses n stack frames ⚠️ |
| right-skewed chain | n | ✅ same |
| complete tree | ⌈log₂(n+1)⌉ | ✅ |
| one child only at each level | that chain's length | ✅ `max` with a 0 branch |

---

## 9. Interview Q&A

**Q: Time and space?**
O(n) time — every node visited once. O(h) space for the call stack: O(log n) balanced, O(n) skewed.

**Q: What if the tree has a million nodes in a single chain?**
The recursive version overflows the stack. Switch to BFS counting levels (O(w), which is O(1) on a
chain) or DFS with an explicit stack.

**Q: Is iterative strictly better?**
No — it trades O(h) for O(w), and those are worst-case opposite. Recursive is also faster, since
there's no allocation.

**Q: How would you find the *minimum* depth?**
LC 111, and it's **not** symmetric: `min(left, right) + 1` is wrong when one child is null, because
a null child isn't a leaf. You must take the non-null branch when only one exists. BFS is also
better here — the first leaf you dequeue gives the answer and you can stop early.

**Q: Diameter of the tree?**
LC 543. Same recursion, but at each node also compute `left + right` as a candidate answer and keep
a running maximum in a field. You return the depth upward while recording the diameter sideways.

**Q: Is the tree balanced?**
LC 110. Same recursion with a sentinel: return −1 to signal "already unbalanced" and propagate it
up, which avoids recomputing depths at every node (O(n) instead of O(n²)).

---

## 10. Pattern

> **A node's depth is `max(left, right) + 1` — the answer at a node comes from the answers at its
> children.**

The transferable half is the second clause. **Whenever a tree problem's answer at a node is a
function of the answers at its children, the recursion writes itself**: handle `null`, recurse both
sides, combine. Depth, diameter, balance, sum, node count, "same tree", "symmetric tree" are all
that same shape with a different combine step.

---

## 11. Takeaways

1. **Depth counts nodes, not edges.** Single node = 1.
2. **One base case (`null → 0`) is enough** — a leaf base case is redundant.
3. **Recursive space is O(h), never O(1).** State the stack cost before being asked.
4. **O(h) means O(log n) balanced, O(n) skewed** — give both bounds, not just the symbol.
5. **Prefer recursive; switch to iterative only when depth could overflow the stack.**
6. **Iterative BFS trades h for w**, and the two are worst-case opposite.
7. **Nested loops don't traverse a tree.** You need a queue or an explicit stack.
8. **The LC 102 size-based loop is a chassis** — this problem is that loop counting passes.
9. **"A node's answer from its children's answers"** is the shape to recognise; it generalises far
   beyond depth.

---

## 12. Related problems

| # | Problem | Connection |
|---|---|---|
| 111 | Minimum Depth | ⚠️ **not** symmetric — a null child isn't a leaf; BFS can exit early |
| 110 | Balanced Binary Tree | same recursion with a −1 sentinel to avoid O(n²) |
| 543 | Diameter of Binary Tree | same recursion; record `left + right` sideways |
| 124 | Binary Tree Maximum Path Sum | same shape, harder combine step |
| 102 | Level Order Traversal | the BFS chassis this problem's iterative version reuses |
| 100 | Same Tree | children's answers combined with `&&` |
| 101 | Symmetric Tree | mirrored version of the same recursion |
| 226 | Invert Binary Tree | same traversal, mutating instead of aggregating |