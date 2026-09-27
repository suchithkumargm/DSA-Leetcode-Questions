# Binary Tree Traversals — LC 94, 144, 145

**Solved:** 27 Sep 2026 (recursive only) · **Iterative versions still owed**
**Pattern:** One recursion, three orders. The position of the visit line relative to the two recursive calls is the entire difference.

---

## 1. The structure you are walking

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
}
```

A binary tree is defined recursively: a node, plus a left subtree, plus a right subtree, either of which may be `null`. That definition is why recursion fits so naturally — the code mirrors the data.

Two measurements you will need constantly:

| Symbol | Meaning | Worst case |
|---|---|---|
| `n` | number of nodes | — |
| `h` | height (longest root-to-leaf path) | `h = n` when the tree is a skewed chain |

A balanced tree has `h ≈ log n`. A skewed tree — every node has only a right child — is a linked list wearing a tree costume, and `h = n`. **Every space claim below turns on this distinction.**

---

## 2. The three orders

Each traversal visits every node exactly once. The only question is *when* you record a node relative to descending into its children.

| Order | Sequence | Record the node… |
|---|---|---|
| Preorder | Node → Left → Right | before going anywhere |
| Inorder | Left → Node → Right | between the two subtrees |
| Postorder | Left → Right → Node | after both subtrees are done |

Worked example:

```
        1
       / \
      2   3
     / \
    4   5
```

- **Preorder:** `1 2 4 5 3`
- **Inorder:** `4 2 5 1 3`
- **Postorder:** `4 5 2 3 1`

### Why each order exists

These are not three arbitrary conventions. Each answers a different question.

- **Preorder** — parent before children. Used when you must create or process a node before its descendants: serializing a tree, copying it, printing a directory listing top-down.
- **Inorder** — on a **BST**, inorder output is sorted ascending. This single fact powers validating a BST, finding the k-th smallest, and the BST-to-sorted-list conversions. Remember it as one line: *inorder on a BST is sorted*.
- **Postorder** — children before parent. Used when a node's answer depends on its subtrees' answers: computing height, diameter, deleting a tree, evaluating an expression tree.

**Recall trigger:** the prefix tells you where the node goes. **Pre**order = node first. **In**order = node in the middle. **Post**order = node last.

---

## 3. Approach 1 — Recursive (what you wrote)

```java
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    traverse(root, result);
    return result;
}

private void traverse(TreeNode node, List<Integer> result) {
    if (node == null) return;

    traverse(node.left, result);
    traverse(node.right, result);
    result.add(node.val);          // move this line to change the order
}
```

Inorder: put `result.add(...)` between the two calls.
Preorder: put it before both.

**Complexity**

- **Time: O(n).** Every node is visited exactly once and does O(1) work. Not O(n log n) — there is no splitting or sorting here, just one pass over n items.
- **Space: O(h) auxiliary**, which is **O(log n)** on a balanced tree and **O(n)** on a skewed one. This is the recursion stack: at any instant the stack holds exactly the chain of ancestors from the root down to the current node, and that chain is at most `h` long.

**Do not count the output list in auxiliary space.** It holds n elements, but it is the required output, not workspace. The convention is to report auxiliary space separately. If an interviewer asks for total space, say `O(n)` output plus `O(h)` stack.

### Naming note

Your helper took a parameter called `root`. After the first call it is not the root — it is an arbitrary node partway down. Call it `node`. This matters when you are tracing a deep recursion and every frame claims to be the root.

---

## 4. Approach 2 — Iterative with an explicit stack

Recursion is not free — it uses the *call* stack, and it is invisible. The iterative versions make that stack explicit. This is what actually gets asked in interviews, because it proves you understand what recursion was doing on your behalf.

### 4a. Preorder — the easy one

```java
public List<Integer> preorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.add(node.val);

        if (node.right != null) stack.push(node.right);   // right FIRST
        if (node.left  != null) stack.push(node.left);    // so left pops first
    }
    return result;
}
```

**The one insight:** push right before left. A stack reverses order, so pushing right first makes left come out first — which is what preorder demands.

Use `ArrayDeque`, not `java.util.Stack`. `Stack` extends `Vector` and every method is synchronised; it is legacy and a reviewer will notice.

### 4b. Inorder — go left as far as possible

```java
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root;

    while (curr != null || !stack.isEmpty()) {
        while (curr != null) {        // descend, remembering the path
            stack.push(curr);
            curr = curr.left;
        }
        curr = stack.pop();           // leftmost unvisited node
        result.add(curr.val);
        curr = curr.right;            // now handle its right subtree
    }
    return result;
}
```

**Invariant — write this before the loop:** *the stack holds exactly the ancestors of `curr` whose values have not yet been recorded.* Once that sentence is on the page, the placement of every line is forced rather than guessed.

The loop condition needs both halves. `curr != null` covers "still descending"; `!stack.isEmpty()` covers "finished a subtree, still have ancestors owed". Dropping either one terminates early.

### 4c. Postorder, two stacks — the trick version

```java
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    if (root == null) return result;

    Deque<TreeNode> stack = new ArrayDeque<>();
    stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.add(node.val);                             // building Node-Right-Left

        if (node.left  != null) stack.push(node.left);
        if (node.right != null) stack.push(node.right);
    }
    Collections.reverse(result);                          // reverse into Left-Right-Node
    return result;
}
```

**Why this works:** postorder is `Left → Right → Node`. Reverse that and you get `Node → Right → Left` — which is just preorder with the children swapped. So run modified preorder, then reverse.

That is worth saying out loud in an interview. It shows you found a structural relationship rather than memorising a second algorithm.

Still `O(n)` time — the reversal is one extra `O(n)` pass, and `O(n) + O(n) = O(n)`. **Constants are dropped.** Writing `O(2n)` is wrong, and it cost you on LC 232.

### 4d. Postorder, one stack — the hard version

The genuine test. The problem: when you pop a node, you cannot tell whether you are arriving *from above* (children still owed) or *returning from the right subtree* (safe to record). You need a memory of where you came from.

```java
public List<Integer> postorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    TreeNode curr = root, lastVisited = null;

    while (curr != null || !stack.isEmpty()) {
        while (curr != null) {
            stack.push(curr);
            curr = curr.left;
        }
        TreeNode peeked = stack.peek();

        if (peeked.right != null && peeked.right != lastVisited) {
            curr = peeked.right;              // right subtree still owed
        } else {
            result.add(peeked.val);           // both children done
            lastVisited = stack.pop();
        }
    }
    return result;
}
```

`lastVisited` is the whole idea: if the node you just finished *is* this node's right child, the right subtree is complete and the parent can finally be recorded. `peek` rather than `pop`, because a node may need to stay on the stack across two decisions.

---

## 5. Approach 3 — Morris traversal, O(1) space

The true optimum for space. If you may modify the tree temporarily, you can traverse with **no stack and no recursion**.

```java
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    TreeNode curr = root;

    while (curr != null) {
        if (curr.left == null) {
            result.add(curr.val);
            curr = curr.right;
        } else {
            TreeNode pred = curr.left;                       // rightmost node of left subtree
            while (pred.right != null && pred.right != curr) pred = pred.right;

            if (pred.right == null) {
                pred.right = curr;                           // thread: a way back up
                curr = curr.left;
            } else {
                pred.right = null;                           // unthread, restore the tree
                result.add(curr.val);
                curr = curr.right;
            }
        }
    }
    return result;
}
```

**The idea:** a stack exists only to remember how to get back up. In inorder, the node you return to after finishing a left subtree is the *inorder predecessor* — the rightmost node of that subtree — and its `right` pointer is `null` and therefore free storage. Point it at the current node, use it to climb back, then undo it.

This is the same move as **LC 73 Set Matrix Zeroes**, where the first row and column were already destined to be zeroed and so were free marker storage. And the same family as **LC 138**, where you interleaved clones so the list became its own map. Three problems, one idea: *find storage the structure is already giving you for free.*

**Complexity**

- **Time: O(n).** It *looks* worse because of the inner `while` hunting for the predecessor. But each edge is walked at most twice across the entire run — once to build a thread, once to remove it. Total edge work is `O(n)`, so the whole traversal stays linear. This amortised argument is exactly the kind you missed on LC 232.
- **Space: O(1) auxiliary.** No stack, no recursion.

**Cost:** it mutates the tree mid-flight. Not thread-safe, and unacceptable if another reader might see the tree in a threaded state. Say this trade-off out loud — knowing when *not* to use Morris is worth as much as knowing it.

---

## 6. Complexity summary

| Approach | Time | Auxiliary space | Notes |
|---|---|---|---|
| Recursive | O(n) | O(h) → O(log n) balanced, O(n) skewed | Call stack; simplest to write |
| Iterative, one stack | O(n) | O(h) | Same cost, stack made explicit |
| Postorder, two stacks | O(n) | O(n) | Second structure holds the whole output |
| Morris | O(n) | **O(1)** | Amortised; mutates the tree temporarily |

**Space intuition, in one sentence:** the stack holds the ancestor chain of the current node, so it is bounded by the height — never by the node count, unless the tree *is* a chain.

---

## 7. Dry run — iterative inorder

Tree:

```
        1
       / \
      2   3
     / \
    4   5
```

| Step | curr | Stack (top→bottom) | Output |
|---|---|---|---|
| 1 | 1 → 2 → 4 | 4, 2, 1 | — |
| 2 | pop 4 | 2, 1 | 4 |
| 3 | 4.right = null | 2, 1 | 4 |
| 4 | pop 2 | 1 | 4 2 |
| 5 | curr = 5, push | 5, 1 | 4 2 |
| 6 | pop 5 | 1 | 4 2 5 |
| 7 | pop 1 | — | 4 2 5 1 |
| 8 | curr = 3, push, pop | — | 4 2 5 1 3 |

Result `4 2 5 1 3` — correct inorder.

---

## 8. Edge cases

- **Empty tree** (`root == null`) → return an empty list, never `null`. The recursive version handles this through its base case; the iterative ones need a guard or a loop condition that tolerates it.
- **Single node** → a one-element list in all three orders.
- **Skewed tree** (every node right-only, or left-only) → the case where `O(h)` becomes `O(n)`. Worth naming aloud; it is the answer to "when does your space bound get bad?"
- **Duplicate values** → irrelevant. Traversal is structural; it never compares values.
- **Very deep tree** → recursion can overflow the JVM stack at roughly 10⁴–10⁵ frames. The real argument for iterative versions in production code.

---

## 9. Interview questions you should be able to answer

**Why is the recursive space O(h) and not O(n)?**
The stack holds only the current root-to-node path, not every node visited. Frames pop as subtrees finish.

**Can you reconstruct a tree from one traversal?**
No — one order alone is ambiguous. You need two: preorder + inorder, or postorder + inorder. Preorder + postorder is *not* sufficient for a general binary tree. Inorder is the one that splits left from right; the other supplies the root. (LC 105, 106.)

**Which traversal for a BST, and why?**
Inorder — it emits sorted ascending, because everything left of a node is smaller and everything right is larger, and inorder takes them in exactly that sequence.

**Postorder iteratively with one stack — how do you know a node is ready?**
Track `lastVisited`. If it equals the node's right child, both subtrees are finished.

**How do you traverse with O(1) space?**
Morris. Thread the inorder predecessor's null right pointer back to the current node, then undo it.

**Preorder vs BFS — both go top-down, what is the difference?**
Preorder is depth-first and dives to a leaf before backtracking; BFS is breadth-first and finishes each level. Preorder uses a stack (`O(h)`), BFS a queue (`O(w)` where w is the maximum width — up to `n/2` at the bottom level of a full tree).

---

## 10. Pattern lines

**Primary:** One recursion, three orders — the position of the visit line relative to the two recursive calls is the entire difference.

Supporting:

- A stack exists only to remember the way back up; if the structure offers a free pointer, you do not need one (Morris).
- Reversing a traversal turns postorder into mirrored preorder — find the structural relationship instead of memorising a second algorithm.
- Space is bounded by height, not by node count.

---

## 11. Related problems

**Direct follow-ups**

- LC 102 Level Order — the queue pattern almost every other tree problem reuses. Do this next.
- LC 103 Zigzag Level Order, LC 199 Right Side View — level order plus one tweak.

**Built on postorder (children before parent)**

- LC 104 Maximum Depth, LC 543 Diameter, LC 110 Balanced Binary Tree, LC 124 Max Path Sum.
  All share one trick: a helper that returns the subtree's answer while updating a running global.

**Built on inorder (sorted on a BST)**

- LC 98 Validate BST, LC 230 K-th Smallest in a BST.

**Built on preorder (parent before children)**

- LC 297 Serialize and Deserialize, LC 105 / 106 Construct from two traversals.

**Same idea as Morris — storage the structure gives you free**

- LC 73 Set Matrix Zeroes, LC 138 Copy List with Random Pointer.

---

## 12. What is still owed on this topic

You wrote the recursive versions. The iterative ones are the part that gets asked:

1. Iterative preorder — push right before left
2. Iterative inorder — descend left, invariant written first
3. Postorder two-stack — modified preorder, then reverse
4. Postorder one-stack — `lastVisited`
5. Morris inorder — optional, but a strong answer to the O(1) space question

Before each, write the invariant on a blank line: *what does the stack contain right now?* That is the step that was missing on LC 283 and LC 26, where four corrections went into guessing where one line belonged.