# Binary Tree Traversals — LC 94, 144, 145

**Pattern:** One recursion, three orders. The position of the visit line relative to the two
recursive calls is the entire difference.

---

## 1. The structure you are walking

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
}
```

A binary tree is defined recursively: a node, plus a left subtree, plus a right subtree, either of
which may be `null`. That definition is why recursion fits so naturally — the code mirrors the data.

Two measurements you will need constantly:

| Symbol | Meaning | Worst case |
|---|---|---|
| `n` | number of nodes | — |
| `h` | height (longest root-to-leaf path) | `h = n` when the tree is a skewed chain |

A balanced tree has `h ≈ log n`. A skewed tree — every node has only one child — is a linked list
wearing a tree costume, and `h = n`. **Every space claim below turns on this distinction.**

> ⚠️ **Say "O(h)", then give both bounds.** Collapsing straight to O(log n) silently assumes a
> balanced tree, which no problem statement here guarantees. The honest worst case is O(n).

---

## 2. The three orders

Each traversal visits every node exactly once. The only question is *when* you record a node
relative to descending into its children.

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

Not three arbitrary conventions — each answers a different question.

- **Preorder** — parent before children. For when you must create or process a node before its
  descendants: serializing a tree, copying it, printing a directory listing top-down.
- **Inorder** — on a **BST**, inorder output is sorted ascending. This single fact powers validating
  a BST, finding the k-th smallest, and BST-to-sorted-list conversions. Remember it as one line:
  *inorder on a BST is sorted*.
- **Postorder** — children before parent. For when a node's answer depends on its subtrees'
  answers: computing height, diameter, deleting a tree, evaluating an expression tree.

**Recall trigger:** the prefix says where the node goes. **Pre**order = node first. **In**order =
node in the middle. **Post**order = node last.

---

## 3. Approach 1 — Recursive

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

### Complexity

- **Time: O(n).** Every node is visited exactly once and does O(1) work. Not O(n log n) — there's no
  splitting or sorting, just one pass over n items.
- **Space: O(h) auxiliary** — O(log n) balanced, O(n) skewed. This is the recursion stack: at any
  instant it holds exactly the chain of ancestors from the root down to the current node, and that
  chain is at most `h` long.

**Don't count the output list in auxiliary space.** It holds n elements, but it's the required
output, not workspace. If asked for total space, say O(n) output plus O(h) stack.

### Naming note

Name the helper's parameter `node`, not `root`. After the first call it isn't the root — it's an
arbitrary node partway down. This matters when tracing a deep recursion and every frame claims to be
the root.

---

## 4. Approach 2 — Iterative with an explicit stack

Recursion isn't free — it uses the *call* stack, and that stack is invisible. The iterative versions
make it explicit. This is what gets asked, because it proves you understand what recursion was doing
on your behalf.

> ### 🔑 What the recursion was actually storing
> When `traverse(node.left, ...)` is called, the JVM pushes a **stack frame** holding the current
> call's locals — including `node`. When the left call returns, that frame comes back into scope and
> `node` is still there, ready to be recorded.
>
> **The call stack is remembering the parent for you while you go left.** Going iterative means
> taking over that job by hand.

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

**The one insight:** push right before left. A stack is LIFO, so whatever goes on last comes off
first — and preorder needs left processed first.

**Why preorder is the simplest:** you record a node *before* descending, so once it's popped you're
**done** with it. Nothing ever needs revisiting.

Use `ArrayDeque`, not `java.util.Stack`. `Stack` extends `Vector` and synchronises every method;
it's legacy and a reviewer will notice.

> **A valid alternative:** you can also adapt the inorder shape below by moving `result.add(...)`
> into the descent loop. It works — but then the stack's only remaining job is to hand back
> `node.right` later, which is more machinery than the four-line version needs.

### 4b. Inorder — descend left as far as possible

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
        curr = stack.pop();           // leftmost unrecorded node
        result.add(curr.val);
        curr = curr.right;            // left-walk restarts from here
    }
    return result;
}
```

**Invariant — write this before the loop:** *the stack holds exactly the ancestors of `curr` whose
values have not yet been recorded.* Once that sentence is on the page, the placement of every line
is forced rather than guessed.

**The loop condition needs both halves.** `curr != null` covers "still descending";
`!stack.isEmpty()` covers "finished a subtree, still have ancestors owed". Drop either and it
terminates early:

- `curr != null` alone fails the moment you pop a node with no right child, while ancestors remain.
- `!stack.isEmpty()` alone fails at the very start, when the stack is empty and `curr` is the root.

**The non-obvious step** is `curr = curr.right` at the end. After recording a node you don't resume
the left-walk from where you were — you jump to the node's right child and the descent begins again
from there.

### 4c. Postorder via reversal — the practical version

> **The structural fact:** postorder is `Left → Right → Node`. **Reverse it** and you get
> `Node → Right → Left` — which is preorder with the children swapped.

Confirm it on the example tree before trusting it:

```
postorder:           4  5  2  3  1
postorder reversed:  1  3  2  5  4     ← root first, right subtree before left
```

So: run preorder with the pushes swapped, then reverse. Best done with `addFirst`, which does the
reversal as you go rather than in a second pass:

```java
public List<Integer> postorderTraversal(TreeNode root) {
    LinkedList<Integer> result = new LinkedList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();
    if (root != null) stack.push(root);

    while (!stack.isEmpty()) {
        TreeNode node = stack.pop();
        result.addFirst(node.val);        // prepend == reversing as we build

        // LEFT first, then right — the reverse of preorder's push order
        if (node.left  != null) stack.push(node.left);
        if (node.right != null) stack.push(node.right);
    }
    return result;
}
```

Preorder's four lines, with two changes: the pushes swap, and `add` becomes `addFirst`.

> ⚠️ **The push order is the only thing making this postorder rather than preorder.** If you comment
> that line, make sure the comment says *left first* — a stale "right first" comment copied from
> preorder is actively misleading.

#### Three traps in this version

**`addFirst` on a `LinkedList` is O(1). `add(0, x)` on an `ArrayList` is O(n).** Using the latter
makes the whole traversal **O(n²)**. If you prefer `ArrayList`, append normally and call
`Collections.reverse(result)` once at the end — one extra O(n) pass, so still O(n) overall.
`O(n) + O(n) = O(n)`; **never write `O(2n)`.**

**`List` declares `addFirst` only from Java 21**, where `List` extends the new
`SequencedCollection`. On Java 21+ (which LeetCode runs) `List<Integer> result = new LinkedList<>()`
plus `result.addFirst(...)` compiles. On anything older, declare the variable as `LinkedList<Integer>`
or `Deque<Integer>`.

**This is not a true postorder traversal.** It *produces the postorder sequence*, but it never
actually visits a node after its children — it visits root-first and fixes the order afterwards.
Fine when you want the sequence. **Not fine** when the requirement is "process each node after its
children": freeing memory, computing subtree aggregates, deleting a tree, evaluating an expression
tree. For those you need 4d.

### 4d. Postorder with one stack — the genuine version

The real test. **The problem:** when you pop a node you can't tell whether you're arriving *from
above* (children still owed) or *returning from the right subtree* (safe to record). The stack alone
doesn't distinguish the two encounters — so you need a memory of where you came from.

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

`lastVisited` is the whole idea: if the node you just finished **is** this node's right child, the
right subtree is complete and the parent can finally be recorded. **`peek` rather than `pop`,**
because a node must stay on the stack across two separate decisions.

#### The alternative: push each node twice

Same disambiguation, different mechanism. Push every node with a flag (or push it twice). The first
pop means "expand" — push it back as visited, then push right and left. The second pop means "emit".
The flag is what tells the two encounters apart.

Both are ~15 lines and need an explicit way to distinguish arrivals. That's why §4c is the usual
answer when you only need the sequence.

---

## 5. Approach 3 — Morris traversal, O(1) space

The true space optimum. If you may modify the tree temporarily, you can traverse with **no stack and
no recursion**.

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

**The idea:** a stack exists *only* to remember how to get back up. In inorder, the node you return
to after finishing a left subtree is the **inorder predecessor** — the rightmost node of that
subtree — and its `right` pointer is `null`, therefore free storage. Point it at the current node,
use it to climb back, then undo it.

> This is the same move as **LC 73**, where the first row and column were already destined to be
> zeroed and so were free marker storage. And the same family as **LC 138**, where interleaving made
> the list its own map. Three problems, one idea: **find storage the structure is already giving you
> for free.**

### Complexity

- **Time: O(n).** It *looks* worse because of the inner `while` hunting for the predecessor. But
  each edge is walked at most twice across the entire run — once to build a thread, once to remove
  it. Total edge work is O(n), so the traversal stays linear. This is an **amortised** argument: the
  per-step cost varies, the total doesn't.
- **Space: O(1) auxiliary.** No stack, no recursion.

**The cost:** it mutates the tree mid-flight. Not thread-safe, and unacceptable if another reader
might see the tree in a threaded state. Say that trade-off out loud — knowing when *not* to use
Morris is worth as much as knowing it.

---

## 6. Complexity summary

| Approach | Time | Auxiliary space | Notes |
|---|---|---|---|
| Recursive | O(n) | O(h) → O(log n) balanced, O(n) skewed | **Call stack**; simplest to write |
| Iterative, one stack | O(n) | O(h) | Same bound, but on the **heap** |
| Postorder via reversal | O(n) | O(h) | Preorder plus `addFirst`; produces the sequence, doesn't truly post-visit |
| Postorder, `lastVisited` | O(n) | O(h) | The genuine traversal |
| Morris | O(n) | **O(1)** | Amortised; mutates the tree temporarily |

**Space intuition, one sentence:** the stack holds the ancestor chain of the current node, so it's
bounded by the height — never by the node count, unless the tree *is* a chain.

> ### 🔑 Why iterative matters when both are O(h)
> Recursive uses the **call stack**; iterative uses the **heap**. The JVM's thread stack is a fixed,
> small allocation (default ~512 KB–1 MB, roughly 10⁴–10⁵ frames) while the heap is gigabytes. Same
> asymptotic bound, wildly different ceiling — which is why the iterative version survives a deep
> tree that overflows the recursive one.
>
> That's the answer to "both are O(h), so why bother?"

---

## 7. Dry run — iterative inorder

```
        1
       / \
      2   3
     / \
    4   5
```

| Step | Action | Stack (top→bottom) | Output |
|---|---|---|---|
| 1 | descend 1 → 2 → 4 | 4, 2, 1 | — |
| 2 | pop 4, record | 2, 1 | 4 |
| 3 | `curr = 4.right = null` | 2, 1 | 4 |
| 4 | pop 2, record | 1 | 4 2 |
| 5 | `curr = 5`, descend & push | 5, 1 | 4 2 |
| 6 | pop 5, record | 1 | 4 2 5 |
| 7 | pop 1, record | — | 4 2 5 1 |
| 8 | `curr = 3`, push, pop, record | — | 4 2 5 1 3 |

Result `4 2 5 1 3` ✅

---

## 8. Edge cases

- **Empty tree** (`root == null`) → return an empty list, never `null`. The recursive version
  handles this via its base case; the iterative ones need a guard or a loop condition that tolerates
  it.
- **Single node** → a one-element list in all three orders.
- **Skewed tree** → where O(h) becomes O(n). Name it aloud; it's the answer to *"when does your
  space bound get bad?"*
- **Duplicate values** → irrelevant. Traversal is structural; it never compares values.
- **Very deep tree** → recursion overflows the JVM stack at roughly 10⁴–10⁵ frames. The real
  argument for iterative versions in production code.

---

## 9. Interview Q&A

**Why is the recursive space O(h) and not O(n)?**
The stack holds only the current root-to-node path, not every node visited. Frames pop as subtrees
finish.

**Both recursive and iterative are O(h) — why write the iterative one?**
Call stack vs heap. The JVM thread stack is a fixed, small allocation; the heap isn't. §6.

**Can you reconstruct a tree from one traversal?**
No — one order alone is ambiguous. You need two: preorder + inorder, or postorder + inorder.
Preorder + postorder is *not* sufficient for a general binary tree. Inorder is the one that splits
left from right; the other supplies the root. (LC 105, 106.)

**Which traversal for a BST, and why?**
Inorder — it emits sorted ascending, because everything left of a node is smaller and everything
right is larger, and inorder takes them in exactly that sequence.

**Postorder iteratively with one stack — how do you know a node is ready?**
Track `lastVisited`. If it equals the node's right child, both subtrees are finished.

**Why is iterative postorder harder than the other two?**
In preorder and inorder, popping a node means you're done with it. In postorder you encounter each
node **twice** — once descending, once after its right subtree — and the stack alone can't tell the
two apart.

**Your reversal trick gives the right sequence — is it a real postorder traversal?**
No. It visits root-first and reverses. Fine for producing the sequence; not fine when the
requirement is to *act* on a node after its children.

**How do you traverse with O(1) space?**
Morris. Thread the inorder predecessor's null right pointer back to the current node, then undo it.

**Preorder vs BFS — both go top-down, what's the difference?**
Preorder is depth-first and dives to a leaf before backtracking; BFS finishes each level. Preorder
uses a stack (O(h)), BFS a queue (O(w), where w is the maximum width — up to n/2 at the bottom level
of a full tree).

---

## 10. Pattern

**Primary:**
> **One recursion, three orders — the position of the visit line relative to the two recursive calls
> is the entire difference.**

**Supporting:**
- A stack exists only to remember the way back up; if the structure offers a free pointer, you don't
  need one (Morris).
- **Reversing a traversal turns postorder into mirrored preorder** — find the structural
  relationship instead of memorising a second algorithm.
- Space is bounded by height, not by node count.
- Preorder and inorder see each node once; **postorder sees each node twice**, and that's the whole
  difficulty.

---

## 11. Takeaways

1. **Say "O(h)", then give both bounds** — O(log n) balanced, O(n) skewed. Don't collapse to the
   convenient one.
2. **Write the stack invariant before the loop:** *what does the stack contain right now?* That one
   sentence forces every line's placement instead of leaving it to guesswork.
3. **Preorder: push right before left.** Stack is LIFO.
4. **Inorder's loop condition needs both halves**, and the non-obvious step is `curr = curr.right`
   after recording.
5. **Postorder reversed is mirrored preorder.** Use `addFirst`, not `add(0, x)` on an `ArrayList` —
   the latter is O(n) per insert and makes the traversal O(n²).
6. **The reversal trick produces the sequence but doesn't truly post-visit.** Know when that
   distinction matters.
7. **Postorder's real difficulty is the double encounter** — hence `lastVisited` or a two-push flag.
8. **`ArrayDeque` over `java.util.Stack`.**
9. **`List.addFirst` exists from Java 21** (`SequencedCollection`); declare `LinkedList` or `Deque`
   on older versions.
10. **Keep comments honest** — a stale "push right first" on postorder's swapped pushes is worse
    than no comment.
11. **Iterative beats recursive on deep trees** because the heap is not the thread stack, even
    though both are O(h).
12. **Morris is the O(1) answer**, and it costs temporary mutation — state the trade-off unprompted.