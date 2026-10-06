# LC 92 — Reverse Linked List II

**Difficulty:** Medium · **Pattern:** Reverse a sub-segment in place, then re-attach both ends

---

## 1. Problem

Given a linked list and two **1-indexed** positions `left ≤ right`, reverse the nodes from position
`left` to position `right` and return the list.

```
1→2→3→4→5,  left=2, right=4   →   1→4→3→2→5
1→2→3→4→5,  left=1, right=5   →   5→4→3→2→1
[5],        left=1, right=1   →   [5]
```

**Follow-up:** do it in one pass.

> ⚠️ **Positions are 1-indexed**, and `left` can be **1** — which means the head can change. That
> single fact decides the shape of the solution (§3).

---

## 2. The idea

Reversing a whole list is routine. The work here is at the two **seams**:

```
        ┌─ reverse this ─┐
1  →    2  →  3  →  4    →  5
↑                           ↑
nodeBeforeLeft          nodeAfterRight
```

After reversing, three links must be correct:

1. `nodeBeforeLeft.next` → the segment's **new head** (the old node at `right`)
2. the segment's **new tail** (the old node at `left`) → `nodeAfterRight`
3. everything inside the segment, flipped

> ### 🔑 The fact
> **Seed the reversal's `prev` with `nodeAfterRight` instead of `null`, and link 2 happens for
> free.** The first node reversed — the old node at `left`, which becomes the segment's tail — gets
> its `next` set to `nodeAfterRight` on the very first iteration.

That leaves only link 1 to do by hand afterwards.

Same trick as LC 25: when reversing a *sub*-segment rather than a whole list, seeding `prev` with
the continuation makes the segment come out already attached on one side.

---

## 3. Why a dummy node — decide this before writing any loop

`left` can be 1. When it is, the node at position 1 moves into the middle of the reversed segment,
so **the head changes**. There is then no "node before `left`" to attach anything to.

> **The trigger is in the problem statement, not in the debugging.** *Any linked-list problem where
> the head can change gets a dummy node, written before the loop.*

With `dummy.next = head` and `nodeBeforeLeft` starting at `dummy`:

- a node before position `left` always exists, including when `left == 1`
- the `left == 1` branch disappears entirely
- the answer is `dummy.next`, which is correct whether or not the head moved

---

## 4. Solution

```java
class Solution {
    public ListNode reverseBetween(ListNode head, int left, int right) {
        ListNode dummy = new ListNode(-1);
        dummy.next = head;

        // walk to the boundaries, both starting from dummy
        ListNode nodeBeforeLeft = dummy;
        for (int i = 0; i < left - 1; i++) nodeBeforeLeft = nodeBeforeLeft.next;

        ListNode nodeAfterRight = dummy;
        for (int i = 0; i < right + 1; i++) nodeAfterRight = nodeAfterRight.next;

        // reverse the segment, seeded with the continuation
        ListNode newSegmentHead = reverse(nodeBeforeLeft.next, nodeAfterRight, right - left);
        nodeBeforeLeft.next = newSegmentHead;

        return dummy.next;
    }

    // reverses k+1 nodes starting at `node`, with `stop` seeded as the initial prev
    private ListNode reverse(ListNode node, ListNode stop, int k) {
        ListNode curr = node;
        ListNode prev = stop;

        while (curr != null && k >= 0) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
            k--;
        }
        return prev;        // new head of the segment
    }
}
```

**O(n) time · O(1) space.** Two walks to the boundaries plus one reversal — all linear, single
traversal's worth of work.

### The index arithmetic

Both pointers start at **`dummy`**, which sits at conceptual position 0. So the hop count is just
the target position:

| target | position | hops from `dummy` |
|---|---|---|
| `nodeBeforeLeft` | `left − 1` | `left − 1` |
| `nodeAfterRight` | `right + 1` | `right + 1` |

> ⚠️ **Derive these from scratch when you change the starting point.** Moving the start from `head`
> to `dummy` shifts every count by one, and adjusting the old bounds by feel is how off-by-ones get
> in. Ask directly: *starting at `dummy`, how many `.next` hops reach position `left − 1`?*

`nodeAfterRight` can legitimately be **null** (when `right == n`). That's fine and is exactly what
you want seeded as `prev` — it terminates the reversed segment.

### Why `k = right - left` and `k >= 0`

The segment holds `right − left + 1` nodes. With `k` starting at `right − left` and the loop
running while `k >= 0`, the body executes `right − left + 1` times. Equivalent and arguably
clearer: pass the count directly and loop `while (k-- > 0)`.

---

## 5. Common pitfalls

### 🚩 Reversing from `nodeBeforeLeft` instead of `nodeBeforeLeft.next` — creates a cycle

```java
reverse(nodeBeforeLeft, nodeAfterRight, right - left);   // 🚩
```

The reversal starts one node too early, so `nodeBeforeLeft` is dragged *into* the reversed segment.

Trace `1→2→3→4→5`, `left=2`, `right=4`, with `nodeBeforeLeft` = node 1:

```
iter 1:  curr = 1, prev = 5  →  1.next = 5
...
returns node 3
then:    nodeBeforeLeft.next = 3   →   1.next = 3
```

But node 2 now points back at node 1. Result: `1 → 3 → 2 → 1 → 3 → …` — an infinite loop, and the
symptom (a hang or TLE) looks nothing like the cause.

**Pass the first node that should actually move:** `nodeBeforeLeft.next`.

### 🚩 `return head` instead of `return dummy.next`

This defeats the dummy entirely, and it fails exactly in the case the dummy was added for.

Trace `left=1, right=3` on `1→2→3→4→5`. The segment becomes `3→2→1→4`, `dummy.next = 3`, and the
full list is correct — but `head` is node **1**, which now sits third. You return `1→4→5`: a pointer
into the middle of a correct list.

**The dummy only pays off if you return `dummy.next`.**

### 🚩 Not re-deriving the loop bounds after adding the dummy

Keeping `for (int i = 1; i < left - 1; i++)` from a `head`-based version leaves `nodeBeforeLeft` one
node short. See the table in §4.

### 🚩 Seeding `prev = null` in the reversal

Then the segment's new tail terminates the list and everything after `right` is lost. Seed it with
`nodeAfterRight`.

### 🚩 Leftover guards and debug output

`if (head == null || head.next == null) return head;` is dead code once the dummy is in place — a
single-node list with `left == right == 1` flows through correctly on its own. And strip
`System.out.println` and commented-out blocks before submitting.

---

## 6. One-pass alternative — head insertion

The follow-up's intended answer. Instead of reversing a detached segment, repeatedly pull the node
*after* the current one to the front of the segment.

```java
public ListNode reverseBetween(ListNode head, int left, int right) {
    ListNode dummy = new ListNode(-1);
    dummy.next = head;

    ListNode prev = dummy;
    for (int i = 0; i < left - 1; i++) prev = prev.next;

    ListNode curr = prev.next;                 // stays put; becomes the segment's tail

    for (int i = 0; i < right - left; i++) {
        ListNode moved = curr.next;            // the node to pull forward
        curr.next  = moved.next;               // unlink it
        moved.next = prev.next;                // insert at the segment's front
        prev.next  = moved;
    }
    return dummy.next;
}
```

**The idea:** `curr` never moves. Each iteration lifts `curr`'s successor out and splices it in
right after `prev`. After `right − left` lifts, the segment is reversed.

Same O(n)/O(1), but it never walks to `right + 1` and never needs a separate reversal helper. The
three-line splice is easy to get wrong, though — **write down which node each of the three
assignments is moving** before typing them.

Both approaches are a single pass in the asymptotic sense. The §4 version is easier to explain; this
one is tighter.

---

## 7. Dry run — `1→2→3→4→5`, `left=2`, `right=4`

```
dummy → 1 → 2 → 3 → 4 → 5
```

**Boundaries:** `nodeBeforeLeft` = 1 hop from dummy = node **1**.
`nodeAfterRight` = 5 hops from dummy = node **5**.

**Reversal** — `reverse(node 2, node 5, k = 2)`, so `prev` starts as node 5:

| iter | curr | next | `curr.next = prev` | prev after | k after |
|---|---|---|---|---|---|
| 1 | 2 | 3 | `2.next = 5` ← **link 2, free** | 2 | 1 |
| 2 | 3 | 4 | `3.next = 2` | 3 | 0 |
| 3 | 4 | 5 | `4.next = 3` | 4 | −1 |

Loop exits (`k >= 0` fails). Returns `prev` = node **4**.

**Re-attach:** `nodeBeforeLeft.next = 4` → `1.next = 4`.

Final: `1 → 4 → 3 → 2 → 5` ✅ — and note `2.next = 5` was set on the very first iteration by the
seeded `prev`.

### The head-changing case — `left=1, right=3`

`nodeBeforeLeft` = 0 hops = **dummy**. `nodeAfterRight` = 4 hops = node **4**.
Reversal returns node 3; `dummy.next = 3`.

Final: `3 → 2 → 1 → 4 → 5` ✅ — returned via `dummy.next`, since `head` (node 1) is now third.

---

## 8. Edge cases

| case | expected | ✓ |
|---|---|---|
| `left == right` | unchanged | ✅ `k = 0`, one iteration, the node's `next` is reset to the same successor |
| `left == 1` | head changes | ✅ `nodeBeforeLeft` is the dummy; `dummy.next` carries it |
| `right == n` | tail changes | ✅ `nodeAfterRight` is null, which correctly terminates the segment |
| `left == 1 && right == n` | whole list reversed | ✅ both boundaries are the sentinels |
| single node, `left=right=1` | unchanged | ✅ no guard needed |
| two nodes, `left=1, right=2` | swapped | ✅ |

---

## 9. Interview Q&A

**Q: Why the dummy node?**
`left` can be 1, so the head can change. The dummy guarantees a node before position `left` exists
and gives a stable handle to return.

**Q: Why seed `prev` with the node after `right` rather than `null`?**
It attaches the reversed segment's tail to the remainder on the first iteration, so only one seam
has to be fixed by hand.

**Q: What if `right == n`?**
`nodeAfterRight` is null, which is exactly the right value to seed — the reversed segment
terminates the list.

**Q: Can you do it in one pass?**
Yes — head insertion (§6): keep the segment's first node fixed and repeatedly splice its successor
to the front.

**Q: Why does reversing from `nodeBeforeLeft` create a cycle?**
It pulls that node into the reversed segment, so it ends up pointing forward into the segment while
a segment node points back at it.

**Q: What's the space complexity?**
O(1) — a handful of pointers. (A recursive version would be O(n) stack.)

---

## 10. Pattern

> **Reverse the sub-segment with `prev` seeded to the node after it, so the back seam attaches
> itself — then fix only the front seam.**

The transferable half: **a sub-segment reversal has two seams, and seeding `prev` with the
continuation closes one of them for free.** Same move in LC 25.

And the standing rule this problem is a direct test of: **if the head can change, write the dummy
node before the loop, not after the bug.**

---

## 11. Takeaways

1. **`left == 1` is the tell.** Head can change ⇒ dummy node, decided from the problem statement.
2. **Return `dummy.next`, never `head`.** Returning `head` fails in exactly the case the dummy
   exists for.
3. **Reverse from `nodeBeforeLeft.next`**, not `nodeBeforeLeft`. The latter builds a cycle.
4. **Seed `prev` with `nodeAfterRight`** so the back seam attaches itself.
5. **Re-derive loop bounds when you change the starting point.** From `dummy`, hops = target
   position.
6. **`nodeAfterRight` being null is correct**, not an error case.
7. **A cycle shows up as a hang or TLE**, which looks nothing like a wiring bug — trace the first
   two iterations when a list problem hangs.
8. **Delete leftover guards and debug prints** before submitting.
9. 