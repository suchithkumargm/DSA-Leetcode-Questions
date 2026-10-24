# LC 572 — Subtree of Another Tree

**Difficulty:** Easy (rated) / genuinely medium
**Pattern:** A traversal that calls a *second* recursion at every node — and one of the few tree
problems that isn't O(n)

---

## 1. Problem

Given the roots of two binary trees `root` and `subRoot`, return `true` if there is a subtree of
`root` with the **same structure and values** as `subRoot`.

A subtree of a tree means **a node and all of its descendants** — not a partial or floating
fragment.

```
root = [3,4,5,1,2], subRoot = [4,1,2]   →  true
root = [3,4,5,1,2,null,null,null,null,0], subRoot = [4,1,2]  →  false
```

The second case is the one that catches people: the node `4` has children `1` and `2` as required,
but `1` has an extra child `0`. A subtree carries **everything** below it, so the match fails.

---

## 2. The two-part structure

Two different questions, and conflating them is the mistake:

| Question | Answered by |
|---|---|
| Does **this specific node** root an identical tree? | `isIdentical(a, b)` — walks both in lockstep |
| Does **any node anywhere** root an identical tree? | `isSubtree(root, subRoot)` — traverses and asks the above at each node |

> ### 🔑 The fact
> **"Is it a subtree" is "is it identical" asked at every node.** One recursion searches; a second,
> independent recursion verifies.

Note the two recursions descend differently. `isSubtree` moves **only through `root`**, keeping
`subRoot` fixed. `isIdentical` moves through **both trees together**. Mixing those up is the usual
source of confusion.

---

## 3. Solution

```java
class Solution {
    public boolean isSubtree(TreeNode root, TreeNode subRoot) {
        if (root == null) return false;          // ran out of tree, never matched

        if (isIdentical(root, subRoot)) return true;

        return isSubtree(root.left, subRoot) || isSubtree(root.right, subRoot);
    }

    private boolean isIdentical(TreeNode node, TreeNode subNode) {
        if (node == null && subNode == null) return true;                  // both ended together
        if (node == null || subNode == null || node.val != subNode.val) return false;

        return isIdentical(node.left,  subNode.left)
            && isIdentical(node.right, subNode.right);
    }
}
```

### Why `isSubtree` returns `false` on null

Running off the end of `root` means no node here rooted a match. **It is not symmetric with
`isIdentical`'s null case**, where two simultaneous nulls mean "both trees ended together" —
success.

(LeetCode guarantees `subRoot` is non-null. If it could be null, every tree trivially contains it
and the base case would need to say so.)

### `isIdentical`'s three-way null check, in order

```java
if (node == null && subNode == null) return true;     // both ended → match
if (node == null || subNode == null) return false;    // one ended → structure differs
if (node.val != subNode.val)         return false;    // values differ
```

**The order is load-bearing.** The both-null case must come first, or the second check catches it
and reports a false mismatch. And the value comparison must come after both null checks, or it
dereferences null.

---

## 4. Common pitfalls

### 🚩 A redundant value pre-check before calling `isIdentical`

```java
if (root.val == subRoot.val) {       // 🚩 does no work
    if (isIdentical(root, subRoot)) return true;
}
```

`isIdentical` compares the values as its first real test, so the wrapper adds nothing — and it
introduces a dependency on `subRoot` being non-null, since `subRoot.val` would throw before
`isIdentical` got a chance to handle it. Call `isIdentical` directly.

### 🚩 Discarding the result of `isIdentical`

```java
isIdentical(root, subRoot);                                    // 🚩 bare statement
return isSubtree(root.left, subRoot) || isSubtree(root.right, subRoot);
```

A match is found and thrown away; the function can never return true from the current node. The
call has to feed a `return`.

### 🚩 Treating a partial match as a match

`isIdentical` must require that **both** trees end at the same places. Returning `true` when
`subNode == null` but `node` still has children would match a *prefix*, which is not what "subtree"
means. That's the `[3,4,5,1,2,null,null,null,null,0]` test case.

### 🚩 Mixing up which tree each recursion descends

`isSubtree` moves through `root` only, holding `subRoot` fixed. `isIdentical` moves through both
together. Passing `subRoot.left` into `isSubtree` is the classic slip.

### 🚩 Assuming a value match means a structural match

Equal values at the root say nothing about what's below. The whole subtree must be verified.

---

## 5. Complexity

Let **n** = nodes in `root`, **m** = nodes in `subRoot`.

| | | why |
|---|---|---|
| **Time** | **O(n × m)** | `isSubtree` visits all `n` nodes; each call runs `isIdentical`, which costs up to O(m) |
| **Space** | **O(h_root + h_sub)** | the two recursions **nest** — an `isIdentical` stack sits on top of the `isSubtree` frames. Worst case **O(n + m)** on skewed trees |

> ⚠️ **This is one of the few tree problems that isn't O(n).** Flagging that unprompted is worth a
> point — an interviewer asking "can you do better?" is checking whether you noticed.

**The early return doesn't improve the bound.** It helps in practice when values differ quickly,
but the worst case is a tree of all-identical values — `[1,1,1,…,1]` — where every `isIdentical`
call walks the full `m` before failing.

---

## 6. Follow-up — O(n + m) by serialisation

### The idea

Flatten both trees to strings, then ask whether `subRoot`'s string is a **substring** of `root`'s.

### ⚠️ Plain preorder is not enough

A traversal sequence doesn't determine a tree. These two give the same preorder `1, 2`:

```
  1          1
 /            \
2              2
```

A substring match would report a false positive.

### The fix: a marker for every null

```
  1          →  "1,2,#,#,#"
 /
2

  1          →  "1,#,2,#,#"
   \
    2
```

Different strings. **With null markers, preorder becomes a bijection** — one string, one tree. That
is also why reconstruction problems (LC 105/106) need *two* traversals without markers, but
serialisation (LC 297) needs only one *with* them.

### ⚠️ Delimit the values

Two separate collisions:

- Without separators, `12` and `1,2` serialise identically.
- Even with commas, `root` containing `…,12,…` and `subRoot` starting `2,…` can match on the
  digits. Prefixing each value — `"^12"` or a leading comma — prevents a match starting mid-number.

### The honest complexity claim

Serialising is O(n + m). **The substring search is the catch:** Java's `String.contains()` is a
naive scan, O(n × m) worst case — so you've gained nothing asymptotically.

**KMP gets the search to O(n + m)**, and only then is the overall solution O(n + m). Say that
explicitly; claiming O(n + m) while using `contains()` is wrong.

### The other O(n + m) route — Merkle hashing

Hash each subtree from its own value and its children's hashes, bottom-up. Two subtrees are
identical iff their hashes match. One pass over each tree, then compare.

O(n + m), with the usual caveat that hash collisions give false positives unless you verify a match
structurally. Worth naming; rarely worth writing.

Neither follow-up is expected unless asked. **Knowing the naive bound is O(n·m), and that O(n+m)
exists, is the signal.**

---

## 7. Dry run

```
root:        3            subRoot:   4
            / \                     / \
           4   5                   1   2
          / \
         1   2
```

| call | action | result |
|---|---|---|
| `isSubtree(3, 4)` | `isIdentical(3, 4)` → values differ | false, recurse |
| `isSubtree(4, 4)` | `isIdentical(4, 4)` → values match, check children | |
| ↳ `isIdentical(1, 1)` | both leaves, all four child calls are null/null | true |
| ↳ `isIdentical(2, 2)` | same | true |
| | `isIdentical(4, 4)` returns **true** | ✅ `isSubtree` returns true |

### The near-miss case

Add a child `0` under the `1`:

| call | action | result |
|---|---|---|
| `isIdentical(4, 4)` | values match | descend |
| ↳ `isIdentical(1, 1)` | values match | descend |
| ↳↳ `isIdentical(0, null)` | one null, one not | **false** |

The whole chain collapses to false. The extra node below makes it a different subtree — which is
exactly the distinction the problem is testing.

---

## 8. Edge cases

| case | expected | note |
|---|---|---|
| `subRoot` is a single leaf | true iff some leaf of `root` has that value | the common case where the naive bound is cheap |
| `root` and `subRoot` identical | true | matches at the root immediately |
| `subRoot` larger than `root` | false | `isIdentical` fails on a null mismatch |
| `root == null` | false | the base case |
| all values identical, e.g. `[1,1,1,…]` | depends on structure | ⚠️ the **worst case** — no early exit ever fires |
| duplicate values scattered | correct | structure is checked, not just values |
| `subRoot` matches a *prefix* of a subtree | **false** | a subtree includes all descendants |

---

## 9. Interview Q&A

**Q: What's the complexity?**
O(n × m). `isSubtree` visits n nodes, each running an O(m) comparison. Not the usual O(n) for a
tree problem.

**Q: Does the early return help the bound?**
No. On all-identical values every comparison walks the full `m`.

**Q: Can you do better?**
Yes — serialise both with null markers and run a substring search. O(n + m) with KMP; `contains()`
leaves you back at O(n × m). Or Merkle-hash each subtree.

**Q: Why do you need null markers in the serialisation?**
A traversal sequence alone doesn't determine a tree — `1,2` is both a left child and a right child.
Markers make preorder a bijection.

**Q: Why is `isSubtree(null, …)` false but `isIdentical(null, null)` true?**
Different questions. Running off `root` means no match was found; two simultaneous nulls mean the
two trees ended together, which is a match.

**Q: Is a single node a subtree of itself?**
Yes — a node plus all its descendants, and the root qualifies.

**Q: What if you had to match a *subtree shape* ignoring values?**
Drop the value comparison from `isIdentical`. Everything else is unchanged.

---

## 10. Pattern

> **"Is it a subtree" is "is it identical" asked at every node — one recursion searches, a second
> verifies.**

The transferable half: **when a problem is "does any position satisfy P", and P is itself
recursive, you get two nested recursions and a product complexity.** Recognising the product — n
positions × the cost of checking one — is what tells you an O(n + m) approach must exist and what
it has to avoid: re-walking `subRoot` from scratch at every position.

---

## 11. Takeaways

1. **Two recursions, descending differently.** `isSubtree` moves through `root` alone; `isIdentical`
   moves through both in lockstep.
2. **The null cases aren't symmetric.** `isSubtree(null, …)` is false; `isIdentical(null, null)` is
   true.
3. **Order the three-way null check:** both-null first, then one-null, then values.
4. **A subtree carries all its descendants** — matching a prefix is not a match.
5. **A value pre-check before `isIdentical` does no work** and adds a null dependency.
6. **O(n × m), not O(n).** One of the few tree problems where that's true; say so unprompted.
7. **The early return doesn't change the worst case** — all-identical values defeat it.
8. **Nested recursions nest their stacks:** O(h_root + h_sub).
9. **Serialisation needs null markers and value delimiters**, and KMP rather than `contains()`, or
   the O(n + m) claim is false.