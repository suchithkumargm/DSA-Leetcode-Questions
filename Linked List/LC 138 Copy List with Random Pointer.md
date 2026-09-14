# LC 138 — Copy List with Random Pointer

**Difficulty:** Medium
**Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** Hashing on node *identity* / Linked list deep copy
**Date solved:** Sep 2026

---

## 1. Problem

You're given a linked list where each node has:
- `val` — an int
- `next` — the usual forward pointer
- `random` — points to **any** node in the list, or `null`

Return a **deep copy**. Every node in the returned list must be a brand-new node. No pointer in the new list may point at a node from the old list, and no pointer in the old list may be modified.

```
Input:  [[7,null],[13,0],[11,4],[10,2],[1,0]]
Output: [[7,null],[13,0],[11,4],[10,2],[1,0]]
```

The `[val, random_index]` notation is just how LeetCode serialises it. Internally `random` is a real reference, not an index.

---

## 2. Why this problem is actually hard

Copying the `next` spine is trivial — walk the list, `new Node(val)`, chain them. You've done that a hundred times.

The whole difficulty is this one question:

> I'm standing at old node `X`. I know `X.random` points to old node `Y`. I need to set `X'.random = Y'`. **How do I find `Y'` given `Y`?**

That's a **lookup by object identity**. And that's the entire problem. Every approach to LC 138 is just a different answer to "how do I map an old node to its copy?"

| Approach | How it answers "given `Y`, find `Y'`?" |
|---|---|
| Brute force | Linear scan to find `Y`'s index, then walk the copy that many steps |
| HashMap | `map.get(Y)` — O(1) |
| Interleaving | `Y.next` **is** `Y'` — O(1), no extra memory |

Recognising it as a **lookup problem** is the insight. Once you name it that, the HashMap is obvious and the O(1) trick is "can I store the mapping inside the list itself?"

---

## 3. My first (broken) attempt — and what was wrong

```java
temp = head;
Node newTemp = null;
while (temp != null) {
    Node oldRandom = temp.random;
    newTemp = map.get(oldRandom);   // BUG
    temp = temp.next;
    newTemp = newTemp.next;
}
return head;                         // BUG
```

**The plan was right.** Build all copies in pass 1, wire them up in pass 2. That's the correct algorithm. The bugs were all in execution:

### Bug 1 — assigning to a local variable instead of a field

```java
newTemp = map.get(oldRandom);
```

This *reads* a copy node out of the map and drops it into a local. Nothing in the copied list changed. A local assignment writes to a stack slot, not to the heap object.

To mutate the copy, the left side must be a **field of a node**:

```java
newTemp.random = map.get(temp.random);   // writes into the heap object
```

> **Lesson to internalise:** whenever you're "building" or "linking" a structure, the left-hand side of your assignment must be `something.field`. If the LHS is a bare local, you are reading, not building.

### Bug 2 — `map.get(temp)` instead of `map.get(temp.next)`

```java
newTemp.next = map.get(temp);   // BUG: copy of current node
```

`map.get(temp)` **is** `newTemp`. So this says `newTemp.next = newTemp` — a node pointing at itself. Then `newTemp = newTemp.next` puts you right back where you were → infinite loop (or TLE / memory blowup).

Correct: look up the copy of the node the original *points at*.

```java
newTemp.next = map.get(temp.next);
```

Notice this is the exact same shape as the `random` line — `map.get(temp.X)`, never `map.get(temp)`. The symmetry is the tell.

### Bug 3 — returning `head`

Returning the original list. `newHead` was declared but never assigned. Once the map is fully built, the new head is one lookup away: `map.get(head)`.

### Bug 4 — duplicating the loop body before the loop

```java
newTemp.next = map.get(temp.next);      // before the loop
newTemp.random = map.get(temp.random);
while (temp != null) { ... }             // loop overwrites it all anyway
```

Pointless (the loop's first iteration does the same work) *and* it NPEs on an empty list, because `temp.next` runs before anything checks `temp != null`.

> **Lesson:** if you find yourself copy-pasting the loop body above the loop, you're patching a symptom. Fix the loop.

### Bug 5 — `map.put(null, null)`

Not a bug, just noise. `HashMap` permits a `null` key, and `get()` on a missing key already returns `null`. So `map.get(null)` returns `null` whether or not you inserted it. It costs nothing but an interviewer will ask why it's there.

**Where it *does* matter:** the one-pass `computeIfAbsent` version (§6). There, `null` keys genuinely blow up. See that section.

---

## 4. Approach 1 — Brute force (no extra data structure)

### Idea

1. **Pass A:** build the copy list with `next` pointers only. Ignore `random` entirely.
2. **Pass B:** for each old node `X`, walk the *original* list from the head, counting steps, until you hit `X.random`. That gives you an index `j`. Then walk the *copy* list `j` steps from `newHead` to land on the right copy. Assign.

### Code

```java
public Node copyRandomListBrute(Node head) {
    if (head == null) return null;

    // Pass A: copy the spine
    Node newHead = new Node(head.val);
    Node oldTemp = head, newTemp = newHead;
    while (oldTemp.next != null) {
        newTemp.next = new Node(oldTemp.next.val);
        oldTemp = oldTemp.next;
        newTemp = newTemp.next;
    }

    // Pass B: resolve randoms by index
    oldTemp = head;
    newTemp = newHead;
    while (oldTemp != null) {
        if (oldTemp.random != null) {
            // find index of oldTemp.random in the original list
            int idx = 0;
            Node scan = head;
            while (scan != oldTemp.random) {
                scan = scan.next;
                idx++;
            }
            // walk idx steps in the copy
            Node target = newHead;
            for (int i = 0; i < idx; i++) target = target.next;
            newTemp.random = target;
        }
        oldTemp = oldTemp.next;
        newTemp = newTemp.next;
    }
    return newHead;
}
```

### Complexity

- **Time: O(n²).** For each of `n` nodes we do up to two O(n) walks.
- **Space: O(n)** for the output (unavoidable), **O(1)** auxiliary.

### Why it's bad

Every single `random` resolution is a linear search. We're doing `n` linear searches. Classic O(n²) smell.

> **The optimisation trigger:** *"I am repeatedly searching a collection for a specific element."* That sentence should immediately make you reach for a hash table. This is the same reflex as Two Sum going from O(n²) → O(n).

---

## 5. Approach 2 — Two-pass HashMap ✅ (my accepted solution)

### Idea

Replace the linear search with an O(1) lookup. Build `old node → new node` in one pass, then use it to wire everything in a second pass.

The reason it needs two passes: when you're at node `X` in the wiring pass, `X.random` might point **forward** to a node you haven't reached yet. You can't link to a copy that doesn't exist. So pass 1 guarantees *every* copy exists before any linking begins.

### Code

```java
class Solution {
    public Node copyRandomList(Node head) {

        HashMap<Node, Node> map = new HashMap<>();

        // Pass 1: create every copy, record old -> new
        Node temp = head;
        while (temp != null) {
            Node newNode = new Node(temp.val);
            map.put(temp, newNode);
            temp = temp.next;
        }

        // Pass 2: wire up next and random on the copies
        temp = head;
        Node newHead = map.get(temp);
        Node newTemp = newHead;

        while (temp != null) {
            newTemp.next   = map.get(temp.next);
            newTemp.random = map.get(temp.random);
            temp    = temp.next;
            newTemp = newTemp.next;
        }

        return newHead;
    }
}
```

### The invariant that makes it work

> **At the top of every iteration, `newTemp == map.get(temp)`.**

- Base: `temp = head`, `newTemp = map.get(head)`. ✓
- Step: we set `newTemp.next = map.get(temp.next)`, then advance *both*. So the new `newTemp` (`= old newTemp.next`) equals `map.get(old temp.next)` = `map.get(new temp)`. ✓

Because of this invariant, `newTemp` is redundant — you could write `map.get(temp).next = ...` everywhere and drop it. Keeping `newTemp` avoids a hash lookup per iteration, so it's slightly faster and arguably clearer.

### Why `map.get(null)` returning `null` is load-bearing

Both `temp.next` and `temp.random` can be `null`. `map.get(null)` returns `null` (key absent), which is *exactly* the value we want to assign. So the null cases handle themselves — no `if` needed. That's why the loop body is two clean lines.

This is a small but genuinely elegant property of the solution. Point it out in an interview.

### Complexity

- **Time: O(n).** Two passes, O(1) hash ops each.
- **Space: O(n)** for the map + O(n) for the output. Auxiliary space is O(n) — the map is what the follow-up asks you to eliminate.

---

## 6. Approach 2b — One-pass HashMap (variant, worth knowing)

You can fuse the passes if, whenever you need a copy, you create it on demand instead of requiring it to already exist.

```java
public Node copyRandomList(Node head) {
    Map<Node, Node> map = new HashMap<>();
    Node temp = head;
    while (temp != null) {
        Node copy = getOrCreate(map, temp);
        copy.next   = getOrCreate(map, temp.next);
        copy.random = getOrCreate(map, temp.random);
        temp = temp.next;
    }
    return map.get(head);
}

private Node getOrCreate(Map<Node, Node> map, Node node) {
    if (node == null) return null;
    return map.computeIfAbsent(node, n -> new Node(n.val));
}
```

### ⚠️ `computeIfAbsent` + null key

`HashMap` allows a `null` key, so `map.computeIfAbsent(null, n -> new Node(n.val))` will happily call the lambda with `n = null` and **NPE on `n.val`**. That's why the `null` guard is inside `getOrCreate` and not optional.

This is a nice concrete case of the `computeIfAbsent` gotcha — the lambda runs with whatever key you passed, including `null`.

### Verdict

Same O(n)/O(n). Fewer passes but more hash lookups per node (3 instead of ~2) and a helper method. **The two-pass version is easier to explain under pressure.** Know this one exists; ship the two-pass.

---

## 7. Approach 3 — O(1) space interleaving (the follow-up)

*Deferred — sketched here so future-me can pick it up.*

### Idea

The map exists only to answer "given old node `Y`, where is `Y'`?" What if the answer were stored **in the list itself**?

Weave each copy in directly after its original:

```
A → A' → B → B' → C → C' → null
```

Now `Y'` is literally `Y.next`. The map is gone.

### Three passes

1. **Interleave.** For each old node `X`: `X' = new Node(X.val); X'.next = X.next; X.next = X';`
2. **Set randoms.** For each old node `X` (step by 2): if `X.random != null`, then `X.next.random = X.random.next`.
   Read it as: *copy-of-X dot random = copy-of (X's random)*. Both `.next`s are "jump to the copy".
3. **Unzip.** Separate the two lists, **restoring the original's `next` pointers**. You must restore them — the problem says don't modify the input.

### Complexity

- **Time: O(n)** — three linear passes.
- **Space: O(1)** auxiliary. ✅

### Why it's harder to write

The unzip pass is where people bleed. You're advancing two pointers through one interleaved list and re-terminating both with `null`. Off-by-one on the last node is the standard bug.

---

## 8. Dry run (Approach 2)

**Input:** `[[7,null],[13,0],[11,4],[10,2],[1,0]]`

Name the original nodes `A..E`:

| Node | val | next | random |
|---|---|---|---|
| A | 7 | B | null |
| B | 13 | C | A |
| C | 11 | D | E |
| D | 10 | E | C |
| E | 1 | null | A |

### Pass 1 — build the map

| temp | action | map after |
|---|---|---|
| A | `new Node(7)` → A' | `{A→A'}` |
| B | `new Node(13)` → B' | `{A→A', B→B'}` |
| C | `new Node(11)` → C' | `{... C→C'}` |
| D | `new Node(10)` → D' | `{... D→D'}` |
| E | `new Node(1)` → E' | `{A→A', B→B', C→C', D→D', E→E'}` |
| null | loop exits | |

At this point A'..E' all exist but are **five disconnected islands** — every `next` and `random` is still `null`. This is the state my broken code returned from, and why nothing worked.

### Pass 2 — wire them

`newHead = map.get(A) = A'`, `newTemp = A'`

| # | temp | newTemp | `newTemp.next = map.get(temp.next)` | `newTemp.random = map.get(temp.random)` | advance to |
|---|---|---|---|---|---|
| 1 | A | A' | `map.get(B)` = **B'** | `map.get(null)` = **null** | temp=B, newTemp=B' |
| 2 | B | B' | `map.get(C)` = **C'** | `map.get(A)` = **A'** | temp=C, newTemp=C' |
| 3 | C | C' | `map.get(D)` = **D'** | `map.get(E)` = **E'** | temp=D, newTemp=D' |
| 4 | D | D' | `map.get(E)` = **E'** | `map.get(C)` = **C'** | temp=E, newTemp=E' |
| 5 | E | E' | `map.get(null)` = **null** | `map.get(A)` = **A'** | temp=null, newTemp=null |

Loop exits (`temp == null`).

### Result

```
A'(7) → B'(13) → C'(11) → D'(10) → E'(1) → null
 │        │         │         │        │
rand:   null       A'        E'       C'       A'
```

Serialised: `[[7,null],[13,0],[11,4],[10,2],[1,0]]` ✅ — matches input.

**Check no cross-contamination:** every value written was `map.get(...)`, and the map's *values* are all new nodes. So no new node can ever point into the old list. And we never wrote to `temp.anything`, so the original is untouched. Both requirements satisfied structurally, not by luck.

### Edge case: `head == null`

- Pass 1: loop doesn't execute, map stays empty.
- `newHead = map.get(null)` → `null`.
- Pass 2: `temp == null`, loop doesn't execute.
- Return `null`. ✅ No explicit guard needed.

### Edge case: single node, self-random `A.random = A`

- Pass 1: `{A→A'}`
- Pass 2: `A'.next = map.get(null) = null`; `A'.random = map.get(A) = A'` — the copy points at itself, correctly mirroring the original. ✅

---

## 9. The critical Java detail: identity vs equals

`HashMap` uses `hashCode()` and `equals()` to find keys. `Node` overrides **neither**, so it inherits `Object`'s versions — which are **reference identity**.

That's exactly what we need. Two different nodes that both hold `val = 7` are *different keys*, because they're different objects.

**If `Node` had overridden `equals`/`hashCode` by `val`**, this entire solution would break: two nodes with the same value would collide into one map entry, and duplicate-valued lists would produce a mangled copy.

> **Interview answer:** "It works because `Node` doesn't override `equals`, so `HashMap` falls back to reference equality — which is the semantics I want. If I needed to guarantee that regardless of the class, I'd use `IdentityHashMap`, which uses `==` and `System.identityHashCode()` by contract."

Mentioning `IdentityHashMap` unprompted is a strong signal.

---

## 10. Complexity summary

| Approach | Time | Aux Space | Notes |
|---|---|---|---|
| Brute force (index walk) | O(n²) | O(1) | Repeated linear search |
| **Two-pass HashMap** ✅ | **O(n)** | **O(n)** | Clearest to explain. What I shipped. |
| One-pass HashMap | O(n) | O(n) | `computeIfAbsent`; watch null keys |
| Interleaving | O(n) | **O(1)** | The follow-up. Unzip pass is fiddly. |

Output space is O(n) in all cases — that's the deliverable, not overhead. "Auxiliary space" is what's being compared.

---

## 11. Common bugs checklist

- [ ] Assigning `map.get(...)` to a **local** instead of to `node.field` → nothing gets built
- [ ] `map.get(temp)` instead of `map.get(temp.next)` for the next pointer → self-loop → **infinite loop**
- [ ] Forgetting to return `map.get(head)`; returning `head` instead
- [ ] Trying to wire in a single pass without on-demand creation → NPE on forward `random` pointers
- [ ] `computeIfAbsent` with a `null` key → NPE inside the lambda
- [ ] Interleaving version: forgetting to restore the original list's `next` in the unzip pass
- [ ] Interleaving version: not null-guarding `X.random` before `X.random.next`
- [ ] Redundant `map.put(null, null)` — harmless but invites the question

---

## 12. Interview Q&A

**Q: Why two passes? Why can't you do it in one?**
Because `random` can point forward. At node `X`, `X.random` may be a node you haven't visited, so its copy doesn't exist yet. Pass 1 guarantees every copy exists. *(You can do one pass if you create copies lazily on demand — see §6 — but the two-pass version is cleaner.)*

**Q: Why does `HashMap<Node, Node>` work when `Node` has no `equals`/`hashCode`?**
It inherits identity semantics from `Object`, which is precisely the lookup I want. See §9.

**Q: Can you do it in O(1) extra space?**
Yes — interleave copies into the original list so `X.next` is `X'`, set randoms via `X.next.random = X.random.next`, then unzip and restore. §7.

**Q: What if the list has a cycle in the `next` pointers?**
The HashMap version doesn't terminate as written — but neither does the problem's definition of a list. If cycles were allowed, the fix is to guard pass 1 with a visited check (`if (map.containsKey(temp)) break;`). Pass 2 would also need the same bound. The interleaving approach breaks badly under cycles.

**Q: Recursive solution?**
Yes — a memoised DFS. `copy(node)` returns `map.get(node)` if present, else creates the copy, memoises *before* recursing (critical — otherwise `random` cycles infinitely), then recurses into `next` and `random`. O(n) time, O(n) space plus O(n) stack. The memo-before-recurse ordering is the whole trick.

**Q: Does this generalise?**
Directly. This is **graph cloning**. LC 133 (Clone Graph) is the same algorithm with an adjacency list instead of two fixed pointers — same `old → new` map, same "create all nodes first, then wire edges" structure.

---

## 13. Pattern extraction

> **When you need to translate references from one object graph into references in a parallel graph, build an `original → copy` map first, then wire edges using it.**

The map is the translation dictionary. Every deep-copy-with-arbitrary-references problem is this.

And the meta-lesson from the O(1) follow-up:

> **When you need a side map keyed by nodes, ask whether the mapping can be embedded into the structure itself.** Interleaving here; parent pointers, in-place marking, and sign-flipping elsewhere.

---

## 14. Related problems

| # | Problem | Connection |
|---|---|---|
| 133 | Clone Graph | Same map-then-wire pattern, general graph |
| 1485 | Clone Binary Tree With Random Pointer | Direct tree analogue |
| 430 | Flatten Multilevel Doubly Linked List | Extra pointer per node, in-place rewiring |
| 146 | LRU Cache | HashMap + hand-rolled DLL; map-to-node lookup |
| 141/142 | Linked List Cycle I/II | Pointer manipulation discipline |
| 61 | Rotate List | Multi-pointer traversal with restoration |

**Next up:** LC 133 (Clone Graph) while this pattern is warm — it's the same solution and it'll cement the idea. Then come back for the O(1) interleaving follow-up here.

---

## 15. Final accepted code

```java
/*
// Definition for a Node.
class Node {
    int val;
    Node next;
    Node random;

    public Node(int val) {
        this.val = val;
        this.next = null;
        this.random = null;
    }
}
*/

class Solution {
    public Node copyRandomList(Node head) {

        HashMap<Node, Node> map = new HashMap<>();

        Node temp = head;
        while (temp != null) {
            Node newNode = new Node(temp.val);
            map.put(temp, newNode);
            temp = temp.next;
        }

        temp = head;
        Node newHead = map.get(temp);
        Node newTemp = newHead;

        while (temp != null) {
            newTemp.next   = map.get(temp.next);
            newTemp.random = map.get(temp.random);
            temp    = temp.next;
            newTemp = newTemp.next;
        }

        return newHead;
    }
}
```

**Time:** O(n) · **Space:** O(n) · Beats ~100% runtime *(noisy — the complexity is what matters)*