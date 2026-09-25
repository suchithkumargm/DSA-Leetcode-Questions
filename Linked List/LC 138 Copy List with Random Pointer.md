# LC 138 — Copy List with Random Pointer

**Difficulty:** Medium · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** Hashing on node *identity* / linked list deep copy

---

## 1. Problem

Each node has `val`, `next`, and `random` — where `random` points to **any** node in the list, or
`null`.

Return a **deep copy**. Every node in the returned list must be brand new. No pointer in the new
list may point at a node from the old list, and **no pointer in the old list may be modified**.

```
Input:  [[7,null],[13,0],[11,4],[10,2],[1,0]]
Output: [[7,null],[13,0],[11,4],[10,2],[1,0]]
```

The `[val, random_index]` notation is only how LeetCode serialises it — internally `random` is a
real reference, not an index.

---

## 2. Why this is actually hard

Copying the `next` spine is trivial. The entire difficulty is one question:

> I'm standing at old node `X`. I know `X.random` points to old node `Y`. I need to set
> `X'.random = Y'`. **How do I find `Y'` given `Y`?**

That's a **lookup by object identity**. Every approach is a different answer to it:

| Approach | How it answers "given `Y`, find `Y'`?" |
|---|---|
| Brute force | Linear scan for `Y`'s index, then walk the copy that many steps |
| HashMap | `map.get(Y)` — O(1) |
| Interleaving | `Y.next` **is** `Y'` — O(1), no extra memory |

Recognising it as a **lookup problem** is the insight. Once named, the HashMap is obvious and the
O(1) version is "can I store the mapping inside the list itself?"

---

## 3. Approach 1 — Brute force

1. **Pass A:** build the copy list with `next` pointers only; ignore `random`.
2. **Pass B:** for each old node `X`, walk the *original* from the head counting steps until you hit
   `X.random`, giving an index `j`. Then walk the *copy* `j` steps from `newHead`.

```java
public Node copyRandomListBrute(Node head) {
    if (head == null) return null;

    Node newHead = new Node(head.val);
    Node oldTemp = head, newTemp = newHead;
    while (oldTemp.next != null) {
        newTemp.next = new Node(oldTemp.next.val);
        oldTemp = oldTemp.next;
        newTemp = newTemp.next;
    }

    oldTemp = head;
    newTemp = newHead;
    while (oldTemp != null) {
        if (oldTemp.random != null) {
            int idx = 0;
            Node scan = head;
            while (scan != oldTemp.random) { scan = scan.next; idx++; }

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

**O(n²) time · O(1) auxiliary.**

> **The optimisation trigger:** *"I am repeatedly searching a collection for a specific element."*
> That sentence should immediately make you reach for a hash table — the same reflex that takes Two
> Sum from O(n²) to O(n).

---

## 4. Approach 2 — Two-pass HashMap ⭐ ship this by default

### Why two passes

When you're at node `X` in the wiring pass, `X.random` might point **forward** to a node you haven't
reached yet, whose copy doesn't exist. Pass 1 guarantees every copy exists before any linking.

> ### 🔑 How to phrase this in an interview
> Not *"the random can be added later in the list"* — the node isn't *added* later, it already
> exists.
>
> Say: **"a pointer can reference a node that appears later in traversal order, so its clone hasn't
> been created yet at the moment I need it."**
>
> And note it's true of **`next` too**, not just `random` — it's forward references in general.

### Code

```java
class Solution {
    public Node copyRandomList(Node head) {
        HashMap<Node, Node> map = new HashMap<>();

        // Pass 1: create every copy, record old -> new
        Node cur = head;
        while (cur != null) {
            map.put(cur, new Node(cur.val));
            cur = cur.next;
        }

        // Pass 2: wire next and random on the copies
        cur = head;
        while (cur != null) {
            Node clone   = map.get(cur);
            clone.next   = map.get(cur.next);
            clone.random = map.get(cur.random);
            cur = cur.next;
        }

        return map.get(head);
    }
}
```

**The invariant:** at the top of every iteration, `clone == map.get(cur)`. If you keep a separate
`newTemp` pointer advancing in lockstep (a valid variant), that invariant justifies it and saves one
hash lookup per iteration.

### Why `map.get(null)` returning `null` is load-bearing

Both `cur.next` and `cur.random` can be null. `map.get(null)` returns null for an absent key — which
is *exactly* the value to assign. The null cases handle themselves, so the loop body is two clean
lines with no `if`.

> ⚠️ **Know that you're relying on it.** A `Map` that rejects null keys on lookup — `TreeMap`,
> `ConcurrentHashMap` — throws here. Saying so shows you know *why* it works.

### Complexity

- **Time O(n).** Two passes, **amortised** O(1) hash ops. (Say "amortised" — Java 8+ treeifies long
  buckets, so the true worst case per op is O(log n), not O(n).)
- **Space O(n)** auxiliary for the map. The output's O(n) is the deliverable, not overhead.

---

## 5. Approach 2b — One-pass HashMap

Fuse the passes by creating copies **on demand**. The forward-reference problem evaporates: whoever
points at a node first creates its clone.

```java
public Node copyRandomList(Node head) {
    Map<Node, Node> map = new HashMap<>();
    Node cur = head;
    while (cur != null) {
        Node copy = getOrCreate(map, cur);
        copy.next   = getOrCreate(map, cur.next);
        copy.random = getOrCreate(map, cur.random);
        cur = cur.next;
    }
    return map.get(head);
}

private Node getOrCreate(Map<Node, Node> map, Node node) {
    if (node == null) return null;
    return map.computeIfAbsent(node, n -> new Node(n.val));
}
```

**Three things that make this correct, all easy to get wrong:**

1. A clone created on demand is **val-only** — do *not* wire its `next`/`random` at creation. Those
   get filled when the main traversal reaches its original. Otherwise you'd recurse down the whole
   list from one node.
2. When the traversal later reaches a node whose clone already exists, you must **reuse** it, never
   create a second one. That "create only if absent" check is the entire reason one pass works.
3. ⚠️ **`computeIfAbsent` + null key:** `HashMap` allows a null key, so the lambda runs with
   `n = null` and **NPEs on `n.val`**. The guard in `getOrCreate` is not optional.

**Verdict:** same O(n)/O(n), more hash lookups per node. Know it exists; ship the two-pass.

---

## 6. Approach 3 — O(1) space interleaving (the follow-up)

### The idea

The map exists only to answer *"given old node `Y`, where is `Y'`?"* So store the answer **in the
list itself**:

```
A → A' → B → B' → C → C' → null
```

> ### 🔑 The whole technique in one sentence
> **Placing each clone directly after its original turns the original's `next` field into the map —
> the clone of any node `X` is reachable as `X.next`, in one hop, with no extra memory.**

That sentence reconstructs the code. "Create a node next to the original, point the randoms, then
separate" does not — that's the steps restated.

### Three passes — and they cannot be merged

1. **Interleave.** For each old `X`: `X' = new Node(X.val); X'.next = X.next; X.next = X';`
   Advance with `cur = cur.next.next` — **not** `cur = cur.next`, which is now the clone.
2. **Set randoms.** For each old `X`: `if (X.random != null) X.next.random = X.random.next;`
   Read it as *clone-of-X dot random = clone-of (X's random)*. Both `.next`s mean "jump to the clone".
3. **Unzip**, restoring the originals' `next` pointers. The problem forbids modifying the input, and
   **LeetCode checks it** — leaving them mutated is a wrong answer, not just an interview nicety.

> ### 🔴 THE INVARIANT — a bug a dry run cannot catch
> **Pass 2 depends on every original still pointing at its clone.** So nothing may be restored until
> pass 2 has finished the *whole* list.
>
> Merge passes 2 and 3 and any `random` pointing **backward** to an already-restored node makes
> `cur.random.next` return that node's real successor — an *original* — instead of a clone.
>
> **A two-node trace can never catch this**, because a two-node list has no backward random. This is
> why tracing is necessary but not sufficient: *before writing a loop, state what the data must look
> like for that loop to be correct.*

### Code

```java
class Solution {
    public Node copyRandomList(Node head) {

        // PASS 1 — interleave: A → A' → B → B' → ...
        Node cur = head;
        while (cur != null) {
            Node clone = new Node(cur.val);
            clone.next = cur.next;
            cur.next   = clone;
            cur = clone.next;          // hop over the clone
        }

        // PASS 2 — randoms. Clone of X is X.next, so clone of X.random is X.random.next.
        cur = head;
        while (cur != null) {
            Node clone = cur.next;
            if (cur.random != null) clone.random = cur.random.next;
            cur = clone.next;
        }

        // PASS 3 — unzip, restoring the originals. Dummy = no head case, no tail guard.
        Node dummy = new Node(-1);
        Node newCur = dummy;

        cur = head;
        while (cur != null) {
            Node clone = cur.next;
            cur.next    = clone.next;   // restore the original
            newCur.next = clone;        // append the clone to the new list
            cur    = clone.next;
            newCur = newCur.next;
        }

        return dummy.next;
    }
}
```

**O(n) time · O(1) auxiliary.**

### 🔑 Why the dummy node in pass 3 matters

Building the clone list **in place** instead needs two patches the dummy version doesn't:

| | in place | with dummy |
|---|---|---|
| last clone's `next` | needs `if (clone.next == null) break;` | falls out — nothing is appended after it |
| `head == null` | needs `if (head == null) return null;` at the top | all 3 loops skip, `dummy.next` is null ✅ |
| first clone | `newHead = head.next` captured before restoring | `dummy` is already the anchor |

**Appending onto a dummy means there is no first case and no last case.**

---

## 7. Common pitfalls

### HashMap version

**Assigning to a local instead of a field.**

```java
newTemp = map.get(oldRandom);   // 🚩 reads into a stack slot; nothing is built
```

To mutate the copy the left side must be `something.field`:

```java
newTemp.random = map.get(cur.random);   // ✅ writes into the heap object
```

> **If the LHS of your assignment is a bare local, you are reading, not building.**

**`map.get(cur)` instead of `map.get(cur.next)`.** `map.get(cur)` *is* the current clone, so
`clone.next = map.get(cur)` makes a node point at itself → infinite loop. The correct shape is
always `map.get(cur.X)`, never `map.get(cur)`. The symmetry with the `random` line is the tell.

**Returning `head`.** The new head is one lookup away: `map.get(head)`.

**Duplicating the loop body above the loop.** Pointless (the first iteration does the same work) and
it NPEs on an empty list, because `cur.next` runs before anything checks `cur != null`. If you're
copy-pasting the body above the loop, you're patching a symptom.

**Hand-rolled `newHead`/`prev` bookkeeping in pass 2.** Unnecessary — the map already knows the clone
of `cur.next`, so both pointers come from lookups and the return value is `map.get(head)`. Roughly
14 lines collapse to 4, and the null-input case disappears.

**`map.put(null, null)`.** Harmless — `get()` on an absent key already returns null — but an
interviewer will ask why it's there.

### Interleaving version

- **`cur = cur.next` in pass 1** instead of `cur.next.next` → infinite loop, then OOM.
- **Setting `clone.random` in pass 1** → points the clone at an **original**, and it's a dead write
  anyway since pass 2 overwrites it. Set `random` in exactly one place.
- **Merging the restore into the random pass** → the invariant bug above.
- **Not null-guarding `X.random`** before `X.random.next` → NPE. (The HashMap version got this for
  free; this one doesn't.)
- **`clone.next.next` on the final clone** → NPE, `clone.next` is already null.
- **Advance overshoot after restoring.** Once `cur.next = clone.next` runs, `cur.next` *is* the next
  original, so `cur = cur.next.next` jumps one too far onto a clone.
- **Starting pass 3 at `head.next`** → the first original is never restored → *"next pointer of node
  N was modified."*

---

## 8. Dry runs

### HashMap version — `[[7,null],[13,0],[11,4],[10,2],[1,0]]`

| Node | val | next | random |
|---|---|---|---|
| A | 7 | B | null |
| B | 13 | C | A |
| C | 11 | D | E |
| D | 10 | E | C |
| E | 1 | null | A |

**Pass 1** produces `{A→A', B→B', C→C', D→D', E→E'}` — five **disconnected islands**, every `next`
and `random` still null.

**Pass 2:**

| # | cur | `clone.next = map.get(cur.next)` | `clone.random = map.get(cur.random)` |
|---|---|---|---|
| 1 | A | `map.get(B)` = **B'** | `map.get(null)` = **null** |
| 2 | B | `map.get(C)` = **C'** | `map.get(A)` = **A'** |
| 3 | C | `map.get(D)` = **D'** | `map.get(E)` = **E'** |
| 4 | D | `map.get(E)` = **E'** | `map.get(C)` = **C'** |
| 5 | E | `map.get(null)` = **null** | `map.get(A)` = **A'** |

Serialised: `[[7,null],[13,0],[11,4],[10,2],[1,0]]` ✅

**No cross-contamination, structurally:** every value written is `map.get(...)`, and the map's
*values* are all new nodes — so no clone can point into the old list. And nothing writes to
`cur.anything`, so the original is untouched.

### Interleaving version — `A → B`

After pass 1: `A → A' → B → B' → null`. Pass 3 with `dummy`, `newCur = dummy`, `cur = A`:

| iter | clone | `cur.next = clone.next` | `newCur.next = clone` | `cur = clone.next` | newCur |
|---|---|---|---|---|---|
| 1 | A' | `A.next = B` ✅ restored | `dummy.next = A'` | `cur = B` | A' |
| 2 | B' | `B.next = null` ✅ restored | `A'.next = B'` | `cur = null` | B' |

`B'.next` is never assigned → stays null ✅ (this is the guard the dummy version doesn't need).

Return `dummy.next` = `A' → B' → null`. Originals: `A → B → null`. ✅

> ⚠️ **Always extend to three nodes mentally.** A two-node trace uses the trailing pointer only once
> and can hide bugs — and it has no backward random, so it cannot catch the §6 invariant bug at all.

---

## 9. Edge cases

| case | HashMap | Interleaving (dummy) |
|---|---|---|
| `head == null` | ✅ loops skip, `map.get(null)` = null | ✅ all 3 loops skip, `dummy.next` = null |
| single node | ✅ | ✅ |
| self-random `A.random = A` | ✅ `map.get(A)` = A' | ✅ `A.random.next` = A' |
| all randoms null | ✅ `map.get(null)` = null | ✅ the guard skips |
| randoms pointing backward | ✅ | ✅ **only because passes 2 and 3 stay separate** |

---

## 10. The critical Java detail: identity vs equals

`HashMap` uses `hashCode()` and `equals()` to find keys. `Node` overrides **neither**, so it inherits
`Object`'s — which are **reference identity**. Exactly what's needed: two different nodes both
holding `val = 7` are *different keys*.

**If `Node` had overridden `equals`/`hashCode` by `val`**, this breaks entirely — duplicate-valued
nodes would collide into one entry and produce a mangled copy.

> **Interview answer:** *"It works because `Node` doesn't override `equals`, so `HashMap` falls back
> to reference equality — which is the semantics I want. If I needed to guarantee that regardless of
> the class, I'd use `IdentityHashMap`, which uses `==` and `System.identityHashCode()` by
> contract."*

Mentioning `IdentityHashMap` unprompted is a strong signal.

---

## 11. Complexity summary

| Approach | Time | Aux Space | Verdict |
|---|---|---|---|
| Brute force (index walk) | O(n²) | O(1) | mention only |
| **Two-pass HashMap** | **O(n)** | O(n) | ✅ default — clearest to explain |
| One-pass HashMap (`computeIfAbsent`) | O(n) | O(n) | know it, don't lead with it |
| **Interleaving** | **O(n)** | **O(1)** | ✅ the follow-up answer |

Output space is O(n) in all cases — that's the deliverable, not overhead. "Auxiliary space" is what's
being compared, and stating that distinction unprompted is worth a point.

---

## 12. Interview Q&A

**Q: Why two passes? Why not one?**
A pointer can reference a node appearing later in traversal order, so its clone doesn't exist when
you need it. True of `next` as well as `random`. Pass 1 guarantees every clone exists. (One pass
works with lazy creation — §5 — but two is cleaner.)

**Q: Why does `HashMap<Node, Node>` work when `Node` has no `equals`/`hashCode`?**
It inherits identity semantics from `Object`, which is precisely the lookup wanted. §10.

**Q: Can you do it in O(1) extra space?**
Yes — interleave clones so `X.next` is `X'`, set randoms via `X.next.random = X.random.next`, then
unzip and restore. §6.

**Q: Why can't the unzip and the random pass be combined?**
The random pass reads `X.random.next` to find a clone, which only works while every original still
points at its clone. Restoring as you go breaks that for any backward-pointing random. **This is the
follow-up that separates people who memorised the code from people who understand it.**

**Q: What if the `next` pointers contain a cycle?**
The HashMap version doesn't terminate as written — but neither does the problem's definition of a
list. If cycles were allowed, guard pass 1 with `if (map.containsKey(cur)) break;`. The interleaving
approach breaks badly under cycles.

**Q: Recursive solution?**
A memoised DFS. `copy(node)` returns the memo if present, else creates the clone, **memoises before
recursing** (critical — otherwise `random` cycles infinitely), then recurses into `next` and
`random`. O(n) time, O(n) space plus O(n) stack. The memo-before-recurse ordering is the whole trick.

**Q: Does this generalise?**
Directly. This is **graph cloning** — LC 133 (Clone Graph) is the same algorithm with an adjacency
list instead of two fixed pointers.

---

## 13. Pattern

**HashMap version:**
> **When translating references from one object graph into a parallel graph, build an
> `original → copy` map first, then wire edges using it.**

**Interleaving version:**
> **Place each clone directly after its original so the list becomes its own map — the clone of `X`
> is `X.next`, one hop, no extra memory.**

**The transferable meta-lesson:**
> **When you need a side map keyed by nodes, ask whether the mapping can be embedded into the
> structure itself.** Interleaving here; parent pointers, in-place marking, index-as-hash and
> sign-flipping elsewhere.

---

## 14. Related problems

| # | Problem | Connection |
|---|---|---|
| 133 | Clone Graph | Same map-then-wire pattern, general graph |
| 1485 | Clone Binary Tree With Random Pointer | Direct tree analogue |
| 430 | Flatten Multilevel Doubly Linked List | Extra pointer per node, in-place rewiring |
| 146 | LRU Cache | HashMap + hand-rolled DLL; map-to-node lookup |
| 287 | Find the Duplicate Number | "embed the mapping in the structure" — index-as-pointer |
| 41 | First Missing Positive | same meta-lesson: in-place marking instead of a side set |
| 25 | Reverse Nodes in k-Group | in-place pointer choreography; dummy node discipline |