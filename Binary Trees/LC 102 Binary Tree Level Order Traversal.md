# LC 102 — Binary Tree Level Order Traversal

**Difficulty:** Medium · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** BFS with level boundaries via queue size

---

## 1. Problem

Given the root of a binary tree, return the values of its nodes **level by level**, left to right,
as a list of lists.

```
    3
   / \
  9  20          →  [[3], [9,20], [15,7]]
     / \
    15  7
```

`root == null` → `[]`.

---

## 2. The idea

BFS gives you nodes in level order automatically — that part is easy. The actual problem is
**knowing where one level ends and the next begins**, because a plain BFS queue mixes levels
together as it goes.

> ### 🔑 The fact
> **At the top of each pass, the queue holds exactly one complete level — so `queue.size()` is that
> level's node count.**

Why: you only ever enqueue the children of nodes you dequeue during the current pass. So when a pass
finishes, everything left in the queue is precisely the next level — nothing more, nothing less.

Capture that size **before** the inner loop, consume exactly that many nodes, and the boundary is
free. No sentinels, no markers, no extra bookkeeping.

---

## 3. Solution — size-based BFS ⭐

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ans = new ArrayList<>();
        if (root == null) return ans;

        Deque<TreeNode> queue = new ArrayDeque<>();
        queue.offer(root);

        while (!queue.isEmpty()) {                     // one iteration = one LEVEL
            int size = queue.size();                   // capture BEFORE enqueuing children
            List<Integer> currLevel = new ArrayList<>();   // fresh list each level

            for (int i = 0; i < size; i++) {           // one iteration = one NODE
                TreeNode node = queue.poll();
                currLevel.add(node.val);

                if (node.left  != null) queue.offer(node.left);
                if (node.right != null) queue.offer(node.right);
            }

            ans.add(currLevel);                        // no copy needed — it's fresh
        }
        return ans;
    }
}
```

**O(n) time · O(w) space**, where `w` is the maximum width of the tree.

### Details that matter

**`int size = queue.size()` must be read before the inner loop.** Inside the loop you're adding
children, so the size changes as you go. Writing `for (int i = 0; i < queue.size(); i++)` chases a
moving target and never respects the level boundary.

**Declare `currLevel` inside the outer loop.** A fresh list each level means nothing else holds a
reference to it, so `ans.add(currLevel)` is safe directly — no `clear()`, no defensive copy, one
fewer O(w) pass per level.

**`ArrayDeque`, not `LinkedList`.** `LinkedList` allocates a wrapper node per enqueue and has poor
cache locality. `ArrayDeque` is array-backed and is what the JDK docs recommend for queue use. On a
BFS that touches every node, that's a measurable constant factor.

**`offer`/`poll` vs `add`/`remove`.** `offer` returns `false` and `poll` returns `null` on failure;
`add` throws `IllegalStateException` and `remove` throws `NoSuchElementException`. For an unbounded
queue guarded by `isEmpty()` neither can fail, so both pairs are correct — just **be consistent**,
don't mix `offer` with `remove`.

---

## 4. Alternative — the null-sentinel BFS

Push a `null` after each level as a delimiter.

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> ans = new ArrayList<>();
    if (root == null) return ans;

    Queue<TreeNode> queue = new LinkedList<>();   // must allow nulls
    queue.add(root);
    queue.add(null);

    List<Integer> currLevel = new ArrayList<>();

    while (!queue.isEmpty()) {
        TreeNode node = queue.remove();

        if (node == null) {
            ans.add(new ArrayList<>(currLevel));   // copy — currLevel is reused
            currLevel.clear();
            if (queue.isEmpty()) break;            // don't append an empty final level
            queue.add(null);                       // delimiter for the next level
        } else {
            currLevel.add(node.val);
            if (node.left  != null) queue.add(node.left);
            if (node.right != null) queue.add(node.right);
        }
    }
    return ans;
}
```

Correct, same complexity. But compare the moving parts:

| | sentinel | size-based |
|---|---|---|
| collection | **must** allow nulls → `LinkedList` (slower) | `ArrayDeque` fine |
| per level | re-add sentinel, `clear()`, defensive copy, `break` guard | fresh list, nothing else |
| branches | null-check on every poll | none |

> **The size-based version is the same algorithm with four moving parts removed**, and it's what an
> interviewer expects. Knowing both and being able to state the tradeoff is stronger than one
> working solution.

---

## 5. Alternative — DFS carrying the depth

Level order doesn't actually require a queue. Recurse with a depth parameter and index into `ans`.

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> ans = new ArrayList<>();
    dfs(root, 0, ans);
    return ans;
}

private void dfs(TreeNode node, int depth, List<List<Integer>> ans) {
    if (node == null) return;
    if (depth == ans.size()) ans.add(new ArrayList<>());   // first node at this depth

    ans.get(depth).add(node.val);
    dfs(node.left,  depth + 1, ans);
    dfs(node.right, depth + 1, ans);
}
```

**The trick:** `depth == ans.size()` is true exactly when you reach a depth for the first time,
because you always visit left before right and depths are reached in increasing order.

**O(n) time · O(h) stack space**, h = height. Better space than BFS on a wide tree, worse on a
skewed one (O(n) stack vs O(1) queue).

> **Worth knowing**, because the interviewer may ask "can you do it without a queue?" — and because
> it's the natural fit for LC 107 (bottom-up) where you can just reverse at the end.

---

## 6. Common pitfalls

### 🚩 `ArrayDeque` rejects null elements

```java
Deque<TreeNode> queue = new ArrayDeque<>();
queue.add(null);     // 🚩 throws NullPointerException immediately
```

The entire null-sentinel scheme depends on storing nulls in a collection that forbids them. Either
switch to `LinkedList` or — better — drop the sentinel. (Same `ArrayDeque` restriction noted for LC
232, where it's harmless because you're storing ints.)

### 🚩 Using `queue.isEmpty()` as the level boundary

```java
while (!queue.isEmpty()) {
    TreeNode node = queue.remove();
    if (queue.isEmpty()) {            // 🚩 not a level boundary
        ans.add(new ArrayList<>(currLevel));
        currLevel.clear();
        break;
    } else { ... }
}
```

`isEmpty()` answers "has the queue run dry", which has nothing to do with level ends — at a level
boundary the queue is usually **full of the next level**. The two only coincide on the final level.

It also **drops nodes**: when the branch fires, `node.val` is never added and its children are never
enqueued.

Trace `[1,2,3]`:
```
queue=[1]. remove 1 → queue now empty → take the if-branch
         → add an EMPTY currLevel to ans, break
```
Returns `[[]]`. Node 1 lost; 2 and 3 never reached.

### 🚩 Reading `queue.size()` inside the loop condition

```java
for (int i = 0; i < queue.size(); i++)   // 🚩 size grows as children are added
```

Capture it into a local first.

### 🚩 Adding the same list reference every level

If `currLevel` is declared **outside** the loop and reused, `ans.add(currLevel)` stores the same
mutable object every time — and `clear()` then wipes every entry in `ans`. With a reused list you
**must** copy: `ans.add(new ArrayList<>(currLevel))`.

The inverse is also worth seeing: with a **fresh** list per level the copy is redundant work. Match
the copy to the lifetime — copy a reused buffer, add a fresh list directly.

### 🚩 Forgetting `root == null`

Enqueuing null and then dereferencing `node.val` → NPE. Return an empty list up front.

### 🚩 Enqueuing null children

`if (node.left != null)` before offering. Otherwise `ArrayDeque` throws, or (with `LinkedList`) your
level counts include phantom nodes.

---

## 7. Dry run — `[3,9,20,null,null,15,7]`

```
    3
   / \
  9  20
     / \
    15  7
```

| pass | queue at top | size | nodes consumed | children enqueued | `ans` after |
|---|---|---|---|---|---|
| 1 | `[3]` | 1 | 3 | 9, 20 | `[[3]]` |
| 2 | `[9,20]` | 2 | 9, 20 | 15, 7 (from 20) | `[[3],[9,20]]` |
| 3 | `[15,7]` | 2 | 15, 7 | none | `[[3],[9,20],[15,7]]` |
| 4 | `[]` | — | loop exits | | |

Note pass 2: `size` is captured as **2** before the loop, so even though the queue grows to 4
mid-pass, exactly two nodes are consumed. That's the whole mechanism.

---

## 8. Edge cases

| case | expected | ✓ |
|---|---|---|
| `root == null` | `[]` | ✅ early return |
| single node | `[[x]]` | ✅ one pass, size 1 |
| left-skewed chain | one list per level, each of size 1 | ✅ |
| complete tree | levels of 1, 2, 4, 8, … | ✅ |
| duplicate values | all appear | ✅ nothing deduplicates |
| negative values | fine | ✅ |

---

## 9. Complexity

| | | why |
|---|---|---|
| **Time** | **O(n)** | every node enqueued once and dequeued once |
| **Space** | **O(w)** | the queue holds at most one level; `w` = max width |

> ⚠️ **O(w) is the tight bound; O(n) is just the upper bound.** For a complete binary tree the
> bottom level is ~n/2 nodes, so it degenerates to O(n) — but for a skewed tree it's O(1). Saying
> *"O(w), worst case O(n) for a balanced tree"* shows you know what's actually in the queue.

The output list is O(n) but that's the deliverable, not auxiliary space.

**On runtime percentiles:** both the sentinel and size-based versions are optimal O(n). LeetCode
percentiles on BFS problems are dominated by JIT warmup and submission timing — identical code
resubmitted lands anywhere from 20% to 95%. Fix the redundant copy and the `LinkedList` on
principle, not to chase the number.

---

## 10. Interview Q&A

**Q: How do you know where a level ends?**
At the top of each pass the queue contains exactly the current level, because only that level's
children have been enqueued. `queue.size()` is the count.

**Q: Why capture the size instead of checking it in the loop condition?**
Because the inner loop enqueues children, so the live size grows mid-pass.

**Q: Can you do it without a queue?**
Yes — DFS carrying a depth, indexing into `ans` by depth (§5). O(h) stack instead of O(w) queue.

**Q: Space complexity?**
O(w) — max width. Worst case O(n) on a balanced tree's bottom level; O(1) on a skewed tree.

**Q: Why `ArrayDeque` over `LinkedList`?**
Array-backed, no per-element node allocation, better locality. The only reason to pick `LinkedList`
is if you need to store nulls — which the size-based approach doesn't.

### The standard follow-ups

| # | Problem | Change from this solution |
|---|---|---|
| **107** | Level Order **Bottom-Up** | `Collections.reverse(ans)` at the end, or `ans.add(0, currLevel)` (the latter is O(n²) on an ArrayList — prefer reversing once) |
| **103** | **Zigzag** Level Order | Track a boolean that flips each pass; on reversed levels either `Collections.reverse(currLevel)` before adding, or build `currLevel` as a `LinkedList` and `addFirst`. **Only the size-based version takes this cleanly** — with sentinels you don't have a clean per-level block to reverse. |
| **199** | Right Side View | Take the **last** node of each level: `if (i == size - 1) ans.add(node.val)` |
| **637** | Average of Levels | Sum inside the inner loop, divide by `size` |
| **515** | Largest Value in Each Row | Track a max inside the inner loop |
| **111** | Minimum Depth | Return the pass count the first time you dequeue a leaf — BFS finds it faster than DFS |
| **116/117** | Populating Next Right Pointers | Link `node.next` to the next node in the same pass; `i == size - 1` gets null |

> Notice how many of these are *the same loop with one line changed inside it*. That's why the
> size-based structure is worth having automatic — it's the chassis for an entire family.

---

## 11. Pattern

> **BFS — and the queue holds exactly one level at the top of each pass, so its size is that level's
> node count.**

The first half says which algorithm; the second half is the part that's easy to lose and hard to
rederive under pressure. Without it you end up reaching for null sentinels or, worse,
`queue.isEmpty()`.

---

## 12. Takeaways

1. **Capture `queue.size()` before the inner loop.** That one local is the entire level-boundary
   mechanism.
2. **`queue.isEmpty()` is not a level boundary.** At a boundary the queue is usually full of the
   next level.
3. **`ArrayDeque` rejects nulls.** It rules out sentinel-based designs — and prompts the better one.
4. **Declare the per-level list inside the loop.** Fresh list ⇒ no `clear()`, no copy.
5. **Copy a reused buffer; add a fresh list directly.** Match the copy to the lifetime.
6. **Space is O(w), not O(n).** Know what's actually in the queue.
7. **`ArrayDeque` over `LinkedList`** for queues, unless you need nulls.
8. **Be consistent with `offer`/`poll` or `add`/`remove`** — don't mix the throwing and
   non-throwing pairs.
9. **This loop is the chassis for 103, 107, 111, 199, 515, 637, 116/117** — usually one line
   changed inside it.

---