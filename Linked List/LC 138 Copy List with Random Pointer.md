# LC 138 — Copy List with Random Pointer

**Difficulty:** Medium
**Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** Hashing on node *identity* / Linked list deep copy

---

## 0. Revision history

| date | what happened |
|---|---|
| **Sep (first solve)** | HashMap version accepted. O(1) follow-up **deferred** — §7 was a sketch only. |
| **Sep 18** | Cold revisit. HashMap solved in **11 min** unaided. Then reached the interleaving insight unaided on a technique never seen before — but the restore pass took **~90 min** of guided debugging. **Note closed.** |
| **Sep 22 (R1)** | Full revisit, O(1) weave only. **Solved unaided in 30 min** (20-min cap → requeue). Reached for a **dummy node unprompted** in pass 3 — the habit flagged 7× in earlier sessions. |

**Self-diagnosis on Sep 22, and it's the accurate one:**
> *"Got the idea very fast, writing the code took a while."*

That is the gap: **not concept recall — clean code under time pressure.** The caps exist to measure
exactly this, which is why 30 min against a 20-min cap is information, not failure.

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

### Complexity

- **Time: O(n²).** For each of `n` nodes we do up to two O(n) walks.
- **Space: O(n)** for the output (unavoidable), **O(1)** auxiliary.

### Why it's bad

Every single `random` resolution is a linear search. We're doing `n` linear searches.

> **The optimisation trigger:** *"I am repeatedly searching a collection for a specific element."* That sentence should immediately make you reach for a hash table. Same reflex as Two Sum going O(n²) → O(n).

---

## 5. Approach 2 — Two-pass HashMap ✅

### Idea

Replace the linear search with an O(1) lookup. Build `old node → new node` in one pass, then use it to wire everything in a second pass.

The reason it needs two passes: when you're at node `X` in the wiring pass, `X.random` might point **forward** to a node you haven't reached yet. You can't link to a copy that doesn't exist. So pass 1 guarantees *every* copy exists before any linking begins.

> ### 🔑 How to phrase this in an interview
> Not *"the random can be added later in the list"* (vague, and the node isn't "added" — it already exists).
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

        // Pass 2: wire up next and random on the copies
        cur = head;
        while (cur != null) {
            Node clone  = map.get(cur);
            clone.next   = map.get(cur.next);
            clone.random = map.get(cur.random);
            cur = cur.next;
        }

        return map.get(head);
    }
}
```

> 🚩 **The cleanup I needed prompting on (Sep 18).** My cold version hand-rolled the second loop
> with `newHead` / `prev` and an `if (newHead == null)` branch for the first node. All of it was
> unnecessary: the map already knows the clone of `cur.next`, so both pointers come from lookups,
> the return value is `map.get(head)`, and the null-input case disappears. ~14 lines → 4.
>
> This is the **head special-case habit** — see §16.

### The invariant

> **At the top of every iteration, `clone == map.get(cur)`.**

If you keep a separate `newTemp` pointer advancing in lockstep (a valid variant), that invariant is
what justifies it — and it saves one hash lookup per iteration.

### Why `map.get(null)` returning `null` is load-bearing

Both `cur.next` and `cur.random` can be `null`. `map.get(null)` returns `null` (key absent), which is *exactly* the value we want to assign. So the null cases handle themselves — no `if` needed.

> ⚠️ **Know that you're relying on it.** A `Map` implementation that rejects null keys on lookup —
> `TreeMap`, `ConcurrentHashMap` — would throw here. Worth saying out loud; it shows you know
> *why* it works rather than that it happened to.

### Complexity

- **Time: O(n).** Two passes, **amortised** O(1) hash ops. (Say "amortised" — Java 8+ treeifies
  long buckets, so true worst case per op is O(log n), not O(n).)
- **Space: O(n)** aux for the map. The output's O(n) is the deliverable, not overhead.

---

## 6. Approach 2b — One-pass HashMap (variant, worth knowing)

Fuse the passes by creating a copy **on demand** instead of requiring it to already exist. The
forward-reference problem evaporates: whoever points at a node first creates its clone.

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

**Three things that make this correct, and all three are easy to get wrong:**

1. A clone created on demand is **val-only** — you do *not* wire its `next`/`random` at creation.
   Those get filled when the main traversal reaches its original. Otherwise you'd recurse down the
   whole list from one node.
2. When the traversal later reaches a node whose clone already exists, you must **reuse** it, never
   create a second one. That "create only if absent" check is the entire reason one pass works.
3. ⚠️ **`computeIfAbsent` + null key:** `HashMap` allows a `null` key, so the lambda runs with
   `n = null` and **NPEs on `n.val`**. The guard in `getOrCreate` is not optional.

**Verdict:** same O(n)/O(n), more hash lookups per node. Know it exists; **ship the two-pass** — it's
easier to explain on a whiteboard.

---

## 7. Approach 3 — O(1) space interleaving ✅ **SOLVED**

*(Was deferred for months. Closed Sep 18, re-solved unaided Sep 22.)*

### Idea

The map exists only to answer *"given old node `Y`, where is `Y'`?"* So store the answer **in the
list itself**. Weave each copy in directly after its original:

```
A → A' → B → B' → C → C' → null
```

> ### 🔑 The whole technique in one sentence
> **Placing each clone directly after its original turns the original's `next` field into the map —
> the clone of any node `X` is reachable as `X.next`, in one hop, with no extra memory.**

That sentence reconstructs the code. *"Create a new node and add it next to the original, then point
the randoms, then separate"* does **not** — that's the steps, restated. (§17)

### Three passes — and they cannot be merged

1. **Interleave.** For each old `X`: `X' = new Node(X.val); X'.next = X.next; X.next = X';`
   Advance with `cur = cur.next.next` — **not** `cur = cur.next`, which is now the clone.
2. **Set randoms.** For each old `X`: `if (X.random != null) X.next.random = X.random.next;`
   Read it as *clone-of-X dot random = clone-of (X's random)*. Both `.next`s mean "jump to the clone".
3. **Unzip**, restoring the originals' `next` pointers. The problem forbids modifying the input, and
   **LeetCode checks it** — leaving them mutated is a wrong answer, not just an interview nicety.

> ### 🔴 THE INVARIANT — the bug a dry run cannot catch
> **Pass 2 depends on every original still pointing at its clone.** So nothing may be restored until
> pass 2 has finished the *whole* list.
>
> On Sep 18 I merged pass 2 and pass 3. If a node's `random` points **backward** to an
> already-restored node, `cur.random.next` returns that node's real successor — an *original* —
> instead of a clone.
>
> **A two-node trace can never catch this**, because a two-node list has no backward random. This is
> why tracing is necessary but not sufficient: *before writing a loop, state what the data must look
> like for that loop to be correct.*

### ✅ Final solution (Sep 22, unaided)

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

        // PASS 3 — unzip, restoring the originals. Dummy node = no head case, no tail guard.
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

### 🔑 Why the dummy node in pass 3 is the real improvement

The Sep 18 version built the clone list **in place** and needed two patches the dummy version
doesn't:

| | Sep 18 (no dummy) | Sep 22 (dummy) |
|---|---|---|
| last clone's `next` | needed `if (clone.next == null) break;` | falls out — nothing is appended after it |
| `head == null` | needed `if (head == null) return null;` at the top | all 3 loops skip, `dummy.next` is null ✅ |
| first clone | `newHead = head.next` captured before restoring | `dummy` is already the anchor |

**Appending onto a dummy means there is no first case and no last case.** That's the payoff, and
reaching for it unprompted on Sep 22 is the single biggest improvement across these sessions.

### The Sep 18 bug catalogue (all in the weave version)

| # | bug | why |
|---|---|---|
| 1 | `cur` never advanced in pass 1 | infinite loop → OOM. The advance is `cur.next.next`, not `cur.next` |
| 2 | `clone.random = cur.random` in pass 1 | points the clone at an **original**; and it can't be fixed in pass 1 anyway (forward references). Also a dead write — pass 2 overwrites it |
| 3 | merged restore into pass 2 | ⚠️ **the invariant bug above** |
| 4 | `clone.next.next` on the last clone | NPE — `clone.next` is already null |
| 5 | advance overshoot after restoring | once `cur.next = clone.next` runs, `cur.next` *is* the next original, so `cur = cur.next.next` jumps one too far and lands on a clone |
| 6 | `if (newHead == null)` branch in the unzip | the head special-case, again (§16) |
| 7 | pass 3 starting at `head.next` | node A never restored → *"next pointer of node 7 was modified"* |

### Complexity

- **Time: O(n)** — three linear passes.
- **Space: O(1)** auxiliary. ✅ (Output is O(n) but that's the deliverable.)

---

## 8. Dry run — HashMap version

**Input:** `[[7,null],[13,0],[11,4],[10,2],[1,0]]`

| Node | val | next | random |
|---|---|---|---|
| A | 7 | B | null |
| B | 13 | C | A |
| C | 11 | D | E |
| D | 10 | E | C |
| E | 1 | null | A |

### Pass 1 — build the map

After the loop: `{A→A', B→B', C→C', D→D', E→E'}`. A'..E' are **five disconnected islands** — every
`next` and `random` is still null. This is the state my first broken version returned from.

### Pass 2 — wire them

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

---

## 8b. Dry run — interleaving version, `A → B`

The two-node case, which is the one that broke repeatedly on Sep 18.

**After pass 1:** `A → A' → B → B' → null`

**Pass 2:** clone-of-A's random = `A.random.next`; same for B. (Guard the nulls.)

**Pass 3**, `dummy`, `newCur = dummy`, `cur = A`:

| iter | clone | `cur.next = clone.next` | `newCur.next = clone` | `cur = clone.next` | `newCur` |
|---|---|---|---|---|---|
| 1 | A' | `A.next = B` ✅ restored | `dummy.next = A'` | `cur = B` | A' |
| 2 | B' | `B.next = null` ✅ restored | `A'.next = B'` | `cur = null` | B' |
| exit | | | | | |

`B'.next` is never assigned → stays null ✅ (this is the guard the dummy version doesn't need).

Return `dummy.next` = `A' → B' → null`. Originals: `A → B → null`. ✅

> ⚠️ **Always extend to three nodes mentally.** A two-node trace uses `prev`/`newCur` only once and
> can hide bugs. And it has no backward random, so it cannot catch the §7 invariant bug at all.

### Edge cases

| case | HashMap | Interleaving (dummy) |
|---|---|---|
| `head == null` | ✅ loops skip, `map.get(null)` = null | ✅ all 3 loops skip, `dummy.next` = null |
| single node | ✅ | ✅ |
| self-random `A.random = A` | ✅ `map.get(A)` = A' | ✅ `A.random.next` = A' |
| all randoms null | ✅ `map.get(null)` = null | ✅ guard skips |
| randoms pointing backward | ✅ | ✅ **only because passes 2 and 3 stay separate** |

---

## 9. The critical Java detail: identity vs equals

`HashMap` uses `hashCode()` and `equals()` to find keys. `Node` overrides **neither**, so it inherits `Object`'s versions — which are **reference identity**. That's exactly what we need: two different nodes both holding `val = 7` are *different keys*.

**If `Node` had overridden `equals`/`hashCode` by `val`**, this solution would break — duplicate-valued nodes would collide into one entry and produce a mangled copy.

> **Interview answer:** "It works because `Node` doesn't override `equals`, so `HashMap` falls back to reference equality — which is the semantics I want. If I needed to guarantee that regardless of the class, I'd use `IdentityHashMap`, which uses `==` and `System.identityHashCode()` by contract."

Mentioning `IdentityHashMap` unprompted is a strong signal.

---

## 10. Complexity summary

| Approach | Time | Aux Space | Status |
|---|---|---|---|
| Brute force (index walk) | O(n²) | O(1) | mentioned only |
| **Two-pass HashMap** | **O(n)** | O(n) | ✅ ship this by default — clearest to explain |
| One-pass HashMap (`computeIfAbsent`) | O(n) | O(n) | know it, don't lead with it |
| **Interleaving** | **O(n)** | **O(1)** | ✅ **closed** — the follow-up answer |

Output space is O(n) in all cases — that's the deliverable, not overhead. "Auxiliary space" is what's being compared, and stating that distinction unprompted is worth a point.

---

## 11. Common bugs checklist

**HashMap version**
- [ ] Assigning `map.get(...)` to a **local** instead of to `node.field` → nothing gets built
- [ ] `map.get(temp)` instead of `map.get(temp.next)` → self-loop → **infinite loop**
- [ ] Returning `head` instead of `map.get(head)`
- [ ] Single pass without on-demand creation → NPE / nulls on forward references
- [ ] `computeIfAbsent` with a `null` key → NPE inside the lambda
- [ ] Hand-rolled `newHead`/`prev` bookkeeping in pass 2 — unnecessary (§16)
- [ ] Redundant `map.put(null, null)` — harmless but invites the question

**Interleaving version**
- [ ] `cur = cur.next` in pass 1 instead of `cur.next.next` → infinite loop
- [ ] Setting `clone.random` in pass 1 → dead write **and** points at an original
- [ ] ⚠️ **Merging the restore into the random pass** → breaks on backward randoms (§7)
- [ ] Not null-guarding `X.random` before `X.random.next`
- [ ] `clone.next.next` on the final clone → NPE
- [ ] Advance overshoot after restoring `cur.next`
- [ ] Starting pass 3 at `head.next` → first original never restored
- [ ] Forgetting to restore the originals at all → **LeetCode fails you**

**Process**
- [ ] 🚩 `System.out.println` left in submitted code — **3 sessions running**
- [ ] 🚩 Dead traversal loop left in after removing the println body
- [ ] 🚩 Two variables holding the same value (`newHead` == `dummy`) — one is redundant

---

## 12. Interview Q&A

**Q: Why two passes? Why can't you do it in one?**
Because a pointer can reference a node that appears later in traversal order, so its clone doesn't exist when I need it. True of `next` as well as `random`. Pass 1 guarantees every clone exists. *(You can do one pass with lazy creation — §6 — but the two-pass version is cleaner.)*

**Q: Why does `HashMap<Node, Node>` work when `Node` has no `equals`/`hashCode`?**
It inherits identity semantics from `Object`, which is precisely the lookup I want. §9.

**Q: Can you do it in O(1) extra space?**
Yes — interleave clones into the original list so `X.next` is `X'`, set randoms via `X.next.random = X.random.next`, then unzip and restore. §7.

**Q: Why can't the unzip and the random pass be combined?**
Because the random pass reads `X.random.next` to find a clone — which only works while every original still points at its clone. Restoring as you go breaks that for any backward-pointing random. **This is the follow-up that separates people who memorised the code from people who understand it.**

**Q: What if the `next` pointers contain a cycle?**
The HashMap version doesn't terminate as written — but neither does the problem's definition of a list. If cycles were allowed, guard pass 1 with `if (map.containsKey(cur)) break;`. The interleaving approach breaks badly under cycles.

**Q: Recursive solution?**
A memoised DFS. `copy(node)` returns the memo if present, else creates the clone, **memoises before recursing** (critical — otherwise `random` cycles infinitely), then recurses into `next` and `random`. O(n) time, O(n) space plus O(n) stack. The memo-before-recurse ordering is the whole trick.

**Q: Does this generalise?**
Directly. This is **graph cloning**. LC 133 (Clone Graph) is the same algorithm with an adjacency list instead of two fixed pointers.

---

## 13. Pattern extraction

**HashMap version:**
> **When you need to translate references from one object graph into references in a parallel graph, build an `original → copy` map first, then wire edges using it.**

**Interleaving version:**
> **Place each clone directly after its original so the list becomes its own map — the clone of `X` is `X.next`, one hop, no extra memory.**

**The meta-lesson, and the transferable one:**
> **When you need a side map keyed by nodes, ask whether the mapping can be embedded into the structure itself.** Interleaving here; parent pointers, in-place marking, index-as-hash and sign-flipping elsewhere.

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

**Next up:** LC 133 (Clone Graph) while this is warm — same solution, different container.

---

## 15. Final accepted code

### Ship-by-default: two-pass HashMap

```java
class Solution {
    public Node copyRandomList(Node head) {
        HashMap<Node, Node> map = new HashMap<>();

        Node cur = head;
        while (cur != null) {
            map.put(cur, new Node(cur.val));
            cur = cur.next;
        }

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
**O(n) time · O(n) aux**

### Follow-up answer: O(1) interleaving — see §7 for the full version.
**O(n) time · O(1) aux**

---

## 16. 🔴 Recurring habit — the hand-rolled head special case

Across LC 138 and LC 25 I wrote this shape **seven times**:

```java
if (newHead == null) { newHead = x; prev = newHead; }
else { prev.next = x; prev = x; }
```

Every single time it was deletable. It showed up in **both** LC 138 versions — the HashMap pass 2
and the interleave unzip.

> **Standing rule: on any linked-list problem where the head can change, or where you're building a
> new list, reach for the dummy node FIRST — before writing the loop, not as cleanup afterwards.**

**Causal chain:** branchy loop → too many moving parts to hold in your head → you stop tracing and
start guessing → hours lost. The Sep 18 session is the proof: four rounds of edit-and-resubmit on a
loop that was hard to trace *because* of the branch.

**Sep 22 status: ✅ applied unprompted.** First session where the dummy came first. Keep it.

Applies to: **19, 21, 24, 25, 61, 82, 83, 86, 92, 138, 203.**

---

## 17. 🔴 Stating the pattern — reason, not steps

Asked for the one-line pattern after the Sep 22 revisit, I said:

> *"create a new node and add it next to the original node, then point the randoms to their correct
> positions, then separate the new nodes into a new list"*

That's **the steps, restated**. It describes what my hands did. It does not let me reconstruct the
code, and it's the version that's gone in a day.

The pattern is the **reason**:

> **Placing each clone directly after its original turns the original's `next` field into the map.**

From that one fact, everything follows: why `cur.random.next` finds the right clone, why the
original list must be restored afterwards, and why the restore can't share a pass with the randoms.

> **Same failure recorded in the LC 169 and LC 1552 docs.** This is now a three-problem pattern:
> when asked for the one-liner, I default to mechanics. **Memorise the reason, not the steps.**

---

*Sep 22 R1 — O(1) weave solved unaided in 30 min (20-min cap → requeued). Dummy node reached
unprompted for the first time. Remaining gap, self-diagnosed and accurate: **idea fast, code slow.***