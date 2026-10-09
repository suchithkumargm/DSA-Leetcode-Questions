# LC 111 — Minimum Depth of Binary Tree

**Difficulty:** Easy (rated) / trap-heavy in practice
**Pattern:** Tree recursion with an asymmetric base case — and the problem where BFS genuinely beats DFS

---

## 1. Problem

Return the **minimum depth** of a binary tree: the number of nodes along the shortest path from the
root down to the **nearest leaf**.

A **leaf** is a node with **no children**.

```
    3
   / \
  9  20          →  2      (root → 9)
     / \
    15  7
```

`root == null` → `0`. Depth counts **nodes**, so a single node has depth 1.

---

## 2. ⚠️ Why this is not LC 104 with `min` instead of `max`

The obvious adaptation is wrong:

```java
return Math.min(minDepth(root.left), minDepth(root.right)) + 1;   // 🚩
```

**Counterexample:** `[2, null, 3, null, 4, null, 5, null, 6]` — a right-leaning chain.

```
2
 \
  3
   \
    4
     \
      5
       \
        6
```

At the root, `root.left` is null so `minDepth(root.left)` returns **0**, and `min(0, 4) + 1 = 1`.
The answer is **5**.

> ### 🔑 The fact
> **A null child returns 0, but a null child is not a leaf.** `min` happily picks that 0 and reports
> a path that doesn't end anywhere.
>
> The shortest path must terminate at a **leaf** — a node with *no* children. A node with one child
> is not a leaf, so the missing side must be ignored rather than minimised over.

This asymmetry doesn't exist for LC 104: `max` never picks the 0, so maximum depth really is the
symmetric version.

---

## 3. Solution — recursion

```java
class Solution {
    public int minDepth(TreeNode root) {
        if (root == null) return 0;

        int leftDepth  = minDepth(root.left);
        int rightDepth = minDepth(root.right);

        // a null child isn't a leaf, so it can't contribute a depth —
        // when one side is missing, take the other
        if (root.left == null || root.right == null) {
            return 1 + Math.max(leftDepth, rightDepth);
        }
        return 1 + Math.min(leftDepth, rightDepth);
    }
}
```

**O(n) time · O(h) space** — O(log n) balanced, O(n) skewed.

### Reading the `max` branch correctly

`Math.max` inside a *minimum*-depth function looks like a bug until you see why. It isn't "take the
bigger depth" — it's **"take the non-null side"**, and `max` expresses that because the null side is
0.

That's exactly why the comment earns its place. Without it, a reader re-derives the logic every
time.

### The leaf case is redundant

A three-branch version is also correct:

```java
if (root.left == null && root.right == null) return 1;                     // redundant
else if (root.left == null || root.right == null) return 1 + Math.max(...);
else return 1 + Math.min(...);
```

When both children are null, both depths are 0, so the `max` branch already gives
`max(0,0) + 1 = 1`. Keeping the explicit leaf case is defensible as documentation; just know it
isn't load-bearing.

---

## 4. ⭐ BFS — the version this problem is actually asking for

**The recursive solution visits every node.** Minimum depth doesn't require that: once you find a
leaf at depth 2, nothing deeper can beat it. DFS has no way to stop early, so it walks the entire
tree regardless.

BFS explores level by level, so:

> **The first leaf it dequeues is the shallowest leaf.** Return immediately.

```java
class Solution {
    public int minDepth(TreeNode root) {
        if (root == null) return 0;

        Deque<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);
        int depth = 1;

        while (!queue.isEmpty()) {
            int size = queue.size();            // this level's node count

            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();

                if (node.left == null && node.right == null) return depth;   // first leaf wins

                if (node.left  != null) queue.offer(node.left);
                if (node.right != null) queue.offer(node.right);
            }
            depth++;
        }
        return depth;   // unreachable — a non-empty tree always has a leaf
    }
}
```

**O(n) worst case · O(w) space.** But on a tree with a shallow leaf near the root and a long chain
elsewhere, BFS touches a handful of nodes where recursion touches all of them.

Note `depth` starts at **1**, not 0 — the root is already the current level when the loop begins.

### Why the asymmetry exists

| | LC 104 (max depth) | LC 111 (min depth) |
|---|---|---|
| can you stop early? | **no** — the deepest leaf could be anywhere | **yes** — the first leaf found is the answer |
| best approach | recursion (simpler, faster) | **BFS** (early exit) |
| null-child handling | symmetric, `max` is correct | asymmetric, needs the one-child case |

> **Maximum depth must see everything either way. Minimum depth is asking for the first thing you
> find — and BFS finds things in the right order.** That's the whole reason this problem is paired
> with 104.

This is the LC 102 level-order loop with a depth counter and one extra check — the same chassis
again.

---

## 5. Common pitfalls

- **`min(left, right) + 1` unconditionally.** The headline bug. §2.
- **Treating a one-child node as a leaf.** A leaf has *no* children; one child means the path must
  continue.
- **Returning 0 for a single node.** Depth counts nodes, so it's 1.
- **Starting the BFS depth counter at 0.** The root is the first level, so it starts at 1.
- **Checking for a leaf when enqueuing rather than when dequeuing.** Dequeue order is what
  guarantees shallowest-first; a check at enqueue time can report a leaf from a deeper level before
  a shallower one has been examined.
- **Claiming O(1) space for the recursion.** It's O(h).
- **Assuming recursion is fine because it was fine for LC 104.** Here it forfeits the early exit.

---

## 6. Dry run — the chain `[2,null,3,null,4,null,5,null,6]`

The case that kills the naive `min`.

| call | `root.left` | `leftDepth` | `rightDepth` | branch | returns |
|---|---|---|---|---|---|
| `minDepth(6)` | null | 0 | 0 | one-null → `max` | `1 + 0 = 1` |
| `minDepth(5)` | null | 0 | 1 | one-null → `max` | `1 + 1 = 2` |
| `minDepth(4)` | null | 0 | 2 | one-null → `max` | `1 + 2 = 3` |
| `minDepth(3)` | null | 0 | 3 | one-null → `max` | `1 + 3 = 4` |
| `minDepth(2)` | null | 0 | 4 | one-null → `max` | `1 + 4 = **5**` |

✅ **5**. The naive version returns 1 at the very first call.

### BFS on `[3,9,20,null,null,15,7]`

| pass | depth | nodes dequeued | leaf found? |
|---|---|---|---|
| 1 | 1 | 3 → has both children, enqueue 9, 20 | no |
| 2 | 2 | **9** → no children | ✅ **return 2** |

Node 20's subtree (15 and 7) is never visited. The recursive version would have walked all of it.

---

## 7. Edge cases

| case | expected | ✓ |
|---|---|---|
| `root == null` | 0 | ✅ base case |
| single node | **1** | ✅ not 0 |
| left-skewed chain of n | n | ✅ the one-child branch carries it |
| right-skewed chain of n | n | ✅ same |
| perfectly balanced | ⌈log₂(n+1)⌉ | ✅ |
| shallow leaf, deep chain elsewhere | the shallow depth | ✅ — and BFS stops early here |
| root with exactly one child | 1 + that subtree's min depth | ✅ the case the naive `min` breaks on |

---

## 8. Interview Q&A

**Q: Why can't you just swap `max` for `min` from LC 104?**
A null child returns 0, and `min` picks it — but a null child isn't a leaf, so that 0 represents a
path that doesn't terminate anywhere. `max` never has this problem because it never picks the 0.

**Q: What's a leaf?**
A node with **no** children. One child means the path must continue — this is the definition the
whole problem hinges on.

**Q: Which is better, DFS or BFS, and why?**
BFS. The first leaf dequeued is the shallowest, so you return immediately instead of exploring the
whole tree. DFS has no early exit.

**Q: Is BFS asymptotically better?**
No — both are O(n) worst case (a perfect tree has no shallow leaf to find early). The win is
practical, on trees where a shallow leaf exists. The space profile also differs: O(w) vs O(h).

**Q: When is DFS the better choice here?**
On a deep, narrow tree, where O(h) stack beats O(w) queue — but that's also where the stack might
overflow. Mostly BFS is the right call for this problem.

**Q: Why does `depth` start at 1 in the BFS version?**
The root occupies the first level before the loop body runs.

---

## 9. Pattern

> **A null child isn't a leaf — so the shortest path can't be minimised over a missing side.**

And the structural half:

> **When the answer is the first thing you'd find, search in the order that finds it first.** BFS
> reaches nodes in increasing depth, so the first leaf is the answer; DFS has no such ordering and
> must exhaust the tree.

---

## 10. Takeaways

1. **`min(left, right) + 1` is wrong** whenever exactly one child is null.
2. **A leaf has no children.** One child means the path continues.
3. **`max` in the one-child branch means "take the non-null side"** — comment it, because it reads
   like a bug.
4. **The explicit leaf case is redundant** — `max(0,0) + 1` already produces 1.
5. **BFS wins here because minimum admits an early exit.** Maximum doesn't, which is why LC 104
   prefers recursion.
6. **Check for the leaf when dequeuing**, not when enqueuing — dequeue order is what guarantees
   shallowest-first.
7. **BFS depth starts at 1.**
8. **Depth counts nodes**, so a single node is 1.
9. **Don't carry an approach across from a similar-looking problem** without checking whether the
   base cases still hold. 104 and 111 differ in exactly one place, and it's the base case.
