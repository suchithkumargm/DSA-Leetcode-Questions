# LC 543 — Diameter of Binary Tree

**Difficulty:** Easy (rated) / Medium (in practice) · **Sheets:** NeetCode 150 | Striver SDE
**Pattern:** Return one value upward, record another sideways — one traversal, two quantities

---

## 1. Problem

Given the root of a binary tree, return the length of its **diameter**: the longest path between
any two nodes. The path may or may not pass through the root.

```
    1
   / \
  2   3        →  3     (path 4 → 2 → 1 → 3, or 5 → 2 → 1 → 3)
 / \
4   5
```

### ⚠️ Diameter is measured in EDGES, not nodes

This is the single most common mistake on the problem, and it's the **opposite convention from
LC 104**, where depth counts nodes.

| | counted in | single node | root with two children |
|---|---|---|---|
| LC 104 depth | **nodes** | 1 | 2 |
| LC 543 diameter | **edges** | 0 | **2** |

Same problem family, different units. **Check which the problem means every time.**

---

## 2. The idea

Any path in a tree has a single **highest** node — the point where it turns around. For a path
turning at node `X`, its length is:

```
(deepest descent into X's left subtree) + (deepest descent into X's right subtree)
```

in edges. So:

> ### 🔑 The fact
> **At every node, the diameter candidate is `leftHeight + rightHeight`.** The answer is the maximum
> of that candidate over all nodes.

No `+ 1`: `leftHeight` already counts the edge from `X` down into the left subtree, and
`rightHeight` the edge down into the right. Adding one more would count `X` itself as an edge.

Sanity check on root-with-two-children: `leftHeight = 1`, `rightHeight = 1`, candidate = **2** ✅

**Every node must be considered as a turning point** — the longest path need not pass through the
root. Classic counterexample:

```
        1
       /
      2
     / \
    3   4
   /     \
  5       6
```

The diameter is `5 → 3 → 2 → 4 → 6` (4 edges), turning at node **2**, not at the root. Any solution
that only measures from the root returns 3.

---

## 3. Approach 1 — brute force (height recomputed per node)

The natural first version: at each node, compute both subtree heights with a helper, take the
candidate, and recurse to compare against the children's answers.

```java
class Solution {
    public int diameterOfBinaryTree(TreeNode root) {
        if (root == null) return 0;

        int leftDiameter  = diameterOfBinaryTree(root.left);
        int rightDiameter = diameterOfBinaryTree(root.right);

        int leftHeight  = findHeight(root.left);
        int rightHeight = findHeight(root.right);
        int selfDiameter = leftHeight + rightHeight;      // NO +1 — edges, not nodes

        return Math.max(selfDiameter, Math.max(leftDiameter, rightDiameter));
    }

    private int findHeight(TreeNode node) {
        if (node == null) return 0;
        return Math.max(findHeight(node.left), findHeight(node.right)) + 1;
    }
}
```

Correct, and worth stating in an interview as the starting point — then improving.

### Complexity

| | | why |
|---|---|---|
| **Time (skewed)** | **O(n²)** | `findHeight` is O(subtree size), called once per node |
| **Time (balanced)** | O(n log n) | subtree sizes halve, so the total is n per level × log n levels |
| **Space** | O(h) | recursion stack; the two recursions **nest** but `findHeight` completes before the outer call returns, so it's O(h), not O(h²) |

> With `n` up to 10⁴ in the constraints, the skewed case is on the edge of TLE. Quote the **worst**
> case as O(n²), but know both.

### Why it's wasteful

Every node's height is recomputed **once for each of its ancestors**. The work is not just
duplicated — it's duplicated proportionally to depth.

> **The optimisation trigger:** *"I'm recursing to compute X, and separately recursing to compute Y
> over the same nodes."* Those are one traversal.

---

## 4. Approach 2 — one pass ⭐ the intended solution

### The insight

Look at `findHeight`. At every node it **already holds** `leftHeight` and `rightHeight` — which is
exactly what the diameter candidate needs. The information is being computed and then thrown away.

The obstacle is that a Java method returns one value, and the caller needs the **height**. So:

> ### 🔑 The technique
> **Return upward what the parent needs (the height); record sideways what the answer needs (the
> diameter), in a field.**

One traversal produces two different quantities.

```java
class Solution {
    private int maxDiameter = 0;

    public int diameterOfBinaryTree(TreeNode root) {
        height(root);
        return maxDiameter;
    }

    private int height(TreeNode node) {
        if (node == null) return 0;

        int leftHeight  = height(node.left);
        int rightHeight = height(node.right);

        maxDiameter = Math.max(maxDiameter, leftHeight + rightHeight);   // sideways

        return Math.max(leftHeight, rightHeight) + 1;                    // upward
    }
}
```

**O(n) time · O(h) space.**

Compared with `findHeight` from §3, the recursive method is **unchanged except for one added line**.
That's the whole optimisation.

### Why `maxDiameter` must be a field (or equivalent)

The recursion can't return both values. Three ways to carry the second one:

1. **Instance field** (above) — simplest, and what interviewers expect.
2. **`int[1]` holder** passed down — avoids mutable instance state, useful if the method must be
   static or thread-safe.
3. **Return a small object/record** `(height, diameter)` — see below.

### ⚠️ Never make the field `static`

A `static` field belongs to the **class**, not the instance, so it is **never reset**. LeetCode
constructs a fresh `Solution` per test case — which resets an *instance* field but does nothing for
a static one. Test 2 starts with test 1's value still in it, and since `Math.max` can only hold or
raise, you get the maximum diameter across every test so far and never anything smaller.

**And it isn't a LeetCode quirk.** Any mutable state outliving one invocation is a bug waiting for
the second call — reuse a single `Solution` instance across two trees and an *instance* field fails
identically.

**Reset at the entry point** and both problems disappear:

```java
public int diameterOfBinaryTree(TreeNode root) {
    maxDiameter = 0;          // safe to call more than once
    height(root);
    return maxDiameter;
}
```

Saying *"I'd reset this at the entry point so the method is re-entrant"* is worth a point in an
interview.

### Option 3 in full — returning a small class

```java
class Solution {
    public int diameterOfBinaryTree(TreeNode root) {
        return findNodeInfo(root).diameter;
    }

    private NodeInfo findNodeInfo(TreeNode node) {
        if (node == null) return new NodeInfo(0, 0);

        NodeInfo left  = findNodeInfo(node.left);
        NodeInfo right = findNodeInfo(node.right);

        int diameter = Math.max(Math.max(left.diameter, right.diameter),
                                left.height + right.height);
        int height = Math.max(left.height, right.height) + 1;

        return new NodeInfo(diameter, height);
    }
}

class NodeInfo {
    int diameter, height;
    NodeInfo(int diameter, int height) { this.diameter = diameter; this.height = height; }
}
```

Correct, no mutable state, same O(n) time. **Two costs:**

- **It allocates one object per node** — n allocations and the GC churn that follows, where the
  field version allocates nothing.
- **It carries the diameter upward, and nobody upward reads it as a diameter.** Each parent only
  takes a max over it — which is exactly what a single field does for free. **A value in the return
  type that no caller consumes is the sign it doesn't belong there.**

**Where the class genuinely earns its place:** when the parent actually *acts* on more than one
child value. LC 110 (balanced) needs the height plus a signal the parent branches on; LC 124 (max
path sum) needs the best downward sum, with the global tracked separately. Know both forms and pick
by whether the parent consumes the second value.

### The ordering detail

`maxDiameter` is updated **before** the return, using the two child heights. It cannot be done after
— the method has already exited. And it must happen at **every** node, not just the root, because
the turning point can be anywhere (§2).

---

## 5. Common pitfalls

### 🚩 `leftHeight + rightHeight + 1`

The off-by-one. Diameter is edges; the two heights already include the edges down from the current
node. Test on root-with-two-children: should be 2, the `+ 1` version gives 3.

> The `+ 1` *is* correct in `height()` itself — there you're counting nodes on a downward path. Two
> different quantities in the same method; only one takes the `+ 1`.

### 🚩 Only measuring through the root

`leftHeight(root) + rightHeight(root)` alone fails whenever the longest path turns lower down. See
the counterexample in §2.

### 🚩 Returning the diameter from the recursion instead of the height

Tempting, but then the parent has nothing to build its own candidate from. The parent needs the
**height**; the diameter is a side effect.

### 🚩 Recomputing heights (§3)

Correct but O(n²) on a skewed tree.

### 🚩 Driving the recursion from the children instead of the root

```java
public int diameterOfBinaryTree(TreeNode root) {
    int leftHeight  = height(root.left);    // 🚩
    int rightHeight = height(root.right);   // 🚩
    return maxDiameter;
}
```

The root is never evaluated as a turning point. Trace `[1,2,3]`: both calls hit leaves, each with
candidate 0, so `maxDiameter` stays 0 — and the root's own candidate of `1 + 1 = 2` is never
computed.

**One call, on the root.** The recursion reaches every node including the root, which is the whole
point of accumulating in a field. The driver is two lines:

```java
height(root);
return maxDiameter;
```

A `root == null` guard in the driver is also dead code — `height(null)` returns 0 and `maxDiameter`
stays 0.

### 🚩 A `static` accumulator, or no reset

See §4. `static` is never reset between test cases; an instance field is safe on LeetCode but not
across two calls on the same object. Reset at the entry point.

### 🚩 Claiming O(1) space

It's O(h) — the call stack. O(log n) balanced, O(n) skewed. Same point as LC 104.

---

## 6. Dry run — one-pass version

```
    1
   / \
  2   3
 / \
4   5
```

Calls resolve bottom-up. `maxDiameter` starts at 0.

| call | leftHeight | rightHeight | candidate | `maxDiameter` after | returns |
|---|---|---|---|---|---|
| `height(4)` | 0 | 0 | 0 | 0 | 1 |
| `height(5)` | 0 | 0 | 0 | 0 | 1 |
| `height(2)` | 1 | 1 | **2** | **2** | 2 |
| `height(3)` | 0 | 0 | 0 | 2 | 1 |
| `height(1)` | 2 | 1 | **3** | **3** | 3 |

Return **3** ✅ — the path `4 → 2 → 1 → 3`, which is 3 edges.

### Dry run on the off-centre case

```
        1
       /
      2
     / \
    3   4
   /     \
  5       6
```

| call | L | R | candidate | max |
|---|---|---|---|---|
| `height(5)` | 0 | 0 | 0 | 0 |
| `height(3)` | 1 | 0 | 1 | 1 |
| `height(6)` | 0 | 0 | 0 | 1 |
| `height(4)` | 0 | 1 | 1 | 1 |
| `height(2)` | 2 | 2 | **4** | **4** |
| `height(1)` | 3 | 0 | 3 | 4 |

Return **4** ✅ — and note the maximum was set at node **2**, then *not* beaten at the root. That's
why every node must be evaluated.

---

## 7. Edge cases

| case | expected | ✓ |
|---|---|---|
| `root == null` | 0 | ✅ `maxDiameter` never updated |
| single node | **0** | ✅ no edges |
| root with one child | 1 | ✅ `1 + 0` |
| root with two children | **2** | ✅ the case that exposes the `+ 1` bug |
| left-skewed chain of n | n − 1 | ✅ |
| diameter not through the root | correct | ✅ every node is a candidate turning point |
| perfectly balanced | 2 × height | ✅ |

---

## 8. Interview Q&A

**Q: Is the diameter in nodes or edges?**
Edges, on LeetCode. Ask if it's ambiguous — the two differ by one and the problem statement is the
only authority.

**Q: Does the path have to pass through the root?**
No. That's the whole difficulty — every node must be considered as the turning point.

**Q: Why does the recursion return height rather than diameter?**
The parent needs the height to compute its own candidate. The diameter is accumulated on the side.

**Q: Why is the brute force O(n²)?**
Each node's height is recomputed once per ancestor. On a chain that's 1 + 2 + … + n.

**Q: Can you avoid the instance field?**
Yes — pass an `int[1]` holder, or return a small `(height, diameter)` record. Same algorithm.

**Q: What if nodes had weights and you wanted the heaviest path?**
LC 124 (Maximum Path Sum). Same structure, with two changes: the "height" becomes the best downward
*sum*, and you clamp negative contributions to 0 since a negative branch is better skipped.

**Q: How does this relate to checking if a tree is balanced?**
LC 110 — same "one traversal computing height plus something else" shape. There the extra signal is
a −1 sentinel meaning "already unbalanced", propagated upward.

---

## 9. Pattern

> **Return upward what the parent needs; record sideways what the answer needs.**

The problem-specific half: *at every node, the diameter candidate is `leftHeight + rightHeight`, and
the height computation already has both in hand.*

The transferable half is the first line. Whenever a tree problem needs a **global** answer that's
computed from **local** information already available during a traversal, don't traverse twice —
return the value the recursion needs and accumulate the answer in a field.

---

## 10. Takeaways

1. **Diameter is edges; depth (LC 104) is nodes.** Opposite conventions in the same problem family.
2. **`leftHeight + rightHeight`, no `+ 1`** — the heights already include the edges down from the
   current node.
3. **Every node is a candidate turning point.** The longest path need not touch the root.
4. **"Recursing for X, separately recursing for Y over the same nodes" = one traversal.** That's the
   optimisation trigger.
5. **Return the height, record the diameter.** A method returns one value; the field carries the
   other.
6. **Update the field before returning**, using the child heights.
7. **Brute force is O(n²) skewed, O(n log n) balanced.** Quote the worst case.
8. **Space is O(h)**, never O(1).
9. **Drive the recursion from the root**, one call — not from the children, or the root is never
   a turning point.
10. **Never `static`** for the accumulator; reset an instance field at the entry point so the
    method is re-entrant.
11. **If no caller consumes the second return value, it belongs in a field, not the return type.**
    A returned class costs one allocation per node and earns its place only when the parent
    actually acts on that value (LC 110, 124).