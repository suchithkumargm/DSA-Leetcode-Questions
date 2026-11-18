# LC 100 — Same Tree

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** Lockstep recursion over two trees — a node's answer from its children's answers

---

## 1. Problem

Given the roots of two binary trees `p` and `q`, return `true` if they are **structurally
identical** and have the **same values** at corresponding positions.

```
p = [1,2,3], q = [1,2,3]   →  true
p = [1,2],   q = [1,null,2] →  false     (same values, different structure)
p = [1,2,1], q = [1,1,2]    →  false     (same structure, different values)
```

The second example is the one that matters: *"same values"* is not enough. Position is part of the
answer.

---

## 2. The recurrence

> ### 🔑 The fact
> **Two trees are the same iff their roots hold the same value AND their left subtrees are the same
> AND their right subtrees are the same.**

```
same(a, b) = (a.val == b.val) && same(a.left, b.left) && same(a.right, b.right)
```

That's the whole algorithm. The only real work is the base cases.

### The three null situations — and they don't agree

| situation | meaning | returns |
|---|---|---|
| **both null** | the two trees ended at the same place | **true** |
| **exactly one null** | one tree continues, the other doesn't → structure differs | **false** |
| **neither null** | compare values, then recurse | depends |

**The order is load-bearing:**

```java
if (p == null && q == null) return true;      // 1. both ended → match
if (p == null || q == null) return false;     // 2. one ended → mismatch
if (p.val != q.val)         return false;     // 3. values differ
```

- Both-null must come **first**, or check 2 catches it and reports a false mismatch.
- The value comparison must come **after** both null checks, or it dereferences null.

> This is exactly the sub-routine `isIdentical` in **LC 572** (Subtree of Another Tree) — that
> problem is this function called at every node of a larger tree. Get it right once and it's
> reusable.

---

## 3. Solution

```java
class Solution {
    public boolean isSameTree(TreeNode p, TreeNode q) {
        if (p == null && q == null) return true;
        if (p == null || q == null) return false;
        if (p.val != q.val)         return false;

        return isSameTree(p.left,  q.left)
            && isSameTree(p.right, q.right);
    }
}
```

**O(min(n, m)) time · O(min(h₁, h₂)) space.** See §6.

### Note the lockstep descent

Both pointers advance **together** — `p.left` pairs with `q.left`, `p.right` with `q.right`. Neither
tree is traversed independently; the recursion walks two trees as if they were one.

That's what distinguishes this from LC 572, where the *outer* recursion moves through one tree only
while holding the other fixed.

### `&&` short-circuits, and that's the early exit

If the left subtrees differ, the right ones are never visited. On trees that diverge near the root,
the function returns after a handful of comparisons regardless of how large the trees are — which is
where the `min(n, m)` bound comes from.

### The compact form

```java
public boolean isSameTree(TreeNode p, TreeNode q) {
    if (p == null || q == null) return p == q;      // both null → true; one null → false
    return p.val == q.val
        && isSameTree(p.left,  q.left)
        && isSameTree(p.right, q.right);
}
```

`p == q` is reference equality, which is `true` exactly when both are null — so one line collapses
the first two base cases. Clever and correct, but the three-case version states the reasoning; prefer
it unless brevity is being graded.

---

## 4. ⚠️ The typo that becomes a different algorithm

```java
return isSameTree(p.left, q.right) && isSameTree(p.right, q.left);   // 🚩 this is LC 101
```

Crossing the children compares each tree against the **mirror** of the other. That is *Symmetric
Tree* (LC 101), applied to two trees instead of one — a correct algorithm answering a different
question.

Trace two identical trees, root 1 with a left child 2 and no right child:

```
isSameTree(1, 1):   values match
  isSameTree(p.left = 2, q.right = null)   →  one null  →  false
```

**Returns false for two trees that are identical.**

A half-typo is worse. `isSameTree(p.left, q.right) && isSameTree(p.right, q.right)` compares
`q.right` twice and never looks at `q.left` at all — so a difference in `q`'s left subtree is
invisible.

> **Why this one deserves a second look rather than a shrug:** most typos produce obvious nonsense.
> This one produces a plausible, working, *wrong* function, and the failing case needs an asymmetric
> tree to surface. Scan the four `.left`/`.right` pairings before submitting.

---

## 5. Other pitfalls

**Comparing values before the null checks** → NPE on the first asymmetric pair.

**Ordering both-null after one-null** → identical trees report `false` the moment both recursions hit
null together, which is every leaf.

**`p.val == q.val` with boxed `Integer`s.** Not an issue here (`int` field, `==` compares
primitives), but if the value type were `Integer`, `==` would compare references and silently work
only for values in the cache range −128…127.

**Treating "same multiset of values" as the answer.** `[1,2]` vs `[1,null,2]` has identical values
and is still `false`.

**Returning `true` when one side runs out but all compared values matched so far.** That's a prefix
match, not identity — the same distinction LC 572 turns on.

---

## 6. Complexity

Let `n` and `m` be the node counts, `h₁` and `h₂` the heights.

| | | why |
|---|---|---|
| **Time** | **O(min(n, m))** | each recursive call does O(1) work and visits one node of each tree; `&&` stops at the first mismatch, and you can't visit more nodes than the smaller tree has |
| **Space** | **O(min(h₁, h₂))** | the recursion stack — one frame per level of the shared descent |

**Quote it as O(n) if the trees are the same size**, which is the interesting case — but knowing
it's bounded by the *smaller* tree shows you noticed the short-circuit.

**Space bounds:** O(log n) when balanced, **O(n) when skewed** — the usual pair. Give both.

> As always, don't claim O(1) space for a recursive tree solution. The call stack is the cost.

---

## 7. Iterative alternative

Replace the call stack with an explicit one holding **pairs** of nodes.

```java
public boolean isSameTree(TreeNode p, TreeNode q) {
    Deque<TreeNode[]> stack = new ArrayDeque<>();
    stack.push(new TreeNode[]{p, q});

    while (!stack.isEmpty()) {
        TreeNode[] pair = stack.pop();
        TreeNode a = pair[0], b = pair[1];

        if (a == null && b == null) continue;          // this branch matches, move on
        if (a == null || b == null) return false;
        if (a.val != b.val)         return false;

        stack.push(new TreeNode[]{a.left,  b.left});
        stack.push(new TreeNode[]{a.right, b.right});
    }
    return true;
}
```

**The structural difference from the recursion:** both-null becomes `continue`, not `return true`.
A matching leaf pair means *this* branch is fine, not that the whole comparison succeeded — so you
keep draining the stack and only return `true` when it empties.

Same O(min(n,m)) time; the stack moves from the thread stack to the heap, which is the argument for
it on very deep trees. A queue works identically — order doesn't matter, since every pair must be
checked unless one fails.

⚠️ `ArrayDeque` rejects nulls, but `new TreeNode[]{null, null}` is a non-null *array* containing
nulls, so this is fine. Pushing the nodes individually would throw.

---

## 8. Dry run — `p = [1,2,1]`, `q = [1,1,2]`

Same structure, swapped children.

```
    p: 1          q: 1
      / \            / \
     2   1          1   2
```

| call | p.val | q.val | action |
|---|---|---|---|
| `(1, 1)` | 1 | 1 | match → recurse left |
| ↳ `(2, 1)` | **2** | **1** | **differ → return false** |

`&&` short-circuits, so the right subtrees are never compared. **Two nodes visited out of six.**

### The structural case — `p = [1,2]`, `q = [1,null,2]`

```
    p: 1          q: 1
      /                \
     2                  2
```

| call | action |
|---|---|
| `(1, 1)` | values match → recurse left |
| ↳ `(p.left = 2, q.left = null)` | one null → **false** |

Identical values, different shape. The null check is what catches it — nothing about the values ever
disagrees.

---

## 9. Edge cases

| case | expected | ✓ |
|---|---|---|
| both trees null | **true** | ✅ first base case |
| one null, one a single node | false | ✅ second base case |
| both single nodes, same value | true | ✅ |
| both single nodes, different values | false | ✅ |
| identical skewed chains | true | ✅ — the **deepest** recursion, O(n) stack |
| same values, mirrored structure | false | ✅ (and the §4 typo makes it `true`) |
| same shape, one value differing at a leaf | false | ✅ full descent before the mismatch |
| negative values / duplicates | fine | ✅ nothing assumes distinctness or sign |

---

## 10. Interview Q&A

**Q: Complexity?**
O(min(n, m)) time — bounded by the smaller tree, because `&&` stops at the first mismatch.
O(min(h₁, h₂)) space for the recursion stack: O(log n) balanced, O(n) skewed.

**Q: Why must the both-null check come first?**
Otherwise the one-null check catches it and reports a mismatch for two trees that legitimately ended
together — which happens at every leaf.

**Q: Why isn't comparing the values enough?**
`[1,2]` and `[1,null,2]` hold the same values in different positions. Structure is part of the
equality.

**Q: Iteratively?**
A stack or queue of node *pairs*, with both-null becoming `continue` rather than an early `true`.

**Q: Could you compare serialisations instead?**
Yes — preorder with explicit null markers makes the serialisation a bijection, so two trees are the
same iff their strings match. O(n) time but O(n) extra space for the strings, so it's worse here.
It's the right tool when you need to compare *many* trees, or check subtree containment (LC 572).

**Q: What if you only cared about structure, ignoring values?**
Drop the `p.val != q.val` check. Everything else is unchanged.

**Q: What if the trees were N-ary?**
Compare the child-list lengths, then recurse pairwise over the children. The two null checks and the
value check are identical; only the fixed left/right pair becomes a loop.

---

## 11. Pattern

> **Walk both trees in lockstep — a node's answer is its own value check combined with its children's
> answers, and the three null cases are the base.**

The transferable half: **when comparing two recursive structures, recurse over both simultaneously
rather than traversing one and searching the other.** The null cases encode "did these two end at the
same place", which is where structural equality actually lives — the value comparison is the easy
part.

---

## 12. Takeaways

1. **Three null cases, in order:** both-null → true, one-null → false, then compare values.
2. **Both-null first**, or every leaf reports a mismatch.
3. **Values after the null checks**, or you dereference null.
4. **Check all four `.left`/`.right` pairings before submitting.** Crossing them is LC 101 — a
   working algorithm answering a different question.
5. **Same values ≠ same tree.** Position is part of the answer.
6. **`&&` short-circuits**, which is both the early exit and the reason the bound is `min(n, m)`.
7. **Space is O(h)** — O(log n) balanced, O(n) skewed. Never O(1).
8. **This function is LC 572's `isIdentical`.** Write it once, properly.