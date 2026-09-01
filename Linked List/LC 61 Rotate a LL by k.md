# LC 61 — Rotate List

**Pattern:** Linked List → close the cycle, walk to the cut, break it
**Related:** LC 19 (Remove Nth From End), LC 86 (Partition List), LC 141/142 (Cycle Detection)
**Difficulty:** Medium

---

## Problem

Given the head of a linked list, rotate the list to the right by `k` places.

```
Input:  1 → 2 → 3 → 4 → 5,  k = 2
Output: 4 → 5 → 1 → 2 → 3
```

Constraints: `0 <= n <= 500`, `0 <= k <= 2 * 10^9`
(`k` is huge, `n` is tiny — that mismatch is the whole hint.)

---

## Core intuition

**Nothing actually moves.**

A right-rotation by `k` looks like elements shifting, but the nodes never change
their relative order. The list is just *cut at one point* and the two halves swap
places:

```
1 → 2 → 3 | 4 → 5        cut after index 2
           ↓
4 → 5 → 1 → 2 → 3        halves swapped
```

So the entire problem is: **find the cut point, relink once.**

Every rotation, no matter how large `k` is, is one single relink. If you find
yourself doing `k` operations, you've misread the problem.

### Second intuition: `k` wraps

Rotating by `n` returns the original list. So only `k % n` matters.
`k = 2_000_000_000` with `n = 5` is really `k = 0`.

---

## Approach 1 — Rotate one node at a time (brute force)

### Idea
Repeat `k` times: walk to the last node, detach it, stick it on the front.

```java
public ListNode rotateRight(ListNode head, int k) {
    if (head == null || head.next == null || k == 0) return head;

    int size = 1;
    ListNode temp = head;
    while (temp.next != null) { temp = temp.next; size++; }

    k = k % size;
    if (k == 0) return head;

    while (k > 0) {
        ListNode lastNode = head, prev = lastNode;
        while (lastNode.next != null) {
            prev = lastNode;
            lastNode = lastNode.next;
        }
        prev.next = null;         // detach tail
        lastNode.next = head;     // put it at the front
        head = lastNode;
        k--;
    }
    return head;
}
```

### Complexity
| | |
|---|---|
| Size pass | O(n) |
| Outer loop | runs `k % n` times |
| Inner loop | O(n) each |
| **Time** | O(n + n·(k mod n)) → **O(n²)** worst case |
| **Space** | **O(1)** |

Since `k mod n <= n - 1`, the bound in terms of input size alone is O(n²).
Write it as: *"O(n·(k mod n)), which is O(n²) worst case."*

### Bugs I hit writing this

**1. Off-by-one on `size`.**
```java
while (temp.next != null) { temp = temp.next; size++; }
```
This loop counts **edges**, not **nodes**. Starting `size = 0` gives `n - 1`.
Fix: start at `size = 1` (you're already standing on a node), or use
`while (temp != null)` starting from `size = 0` — but then you lose the tail
reference, which Approach 2 needs. **Prefer `size = 1` + `temp.next != null`.**

**2. Flipped the modulo operands.**
```java
k = k > size ? size % k : k;   // WRONG
```
`size % k` when `size < k` returns `size` — nonsense. You want to shrink `k`,
so `k` must be on the left: `k % size`.

**3. The ternary misses `k == size`.**
`k > size` is false when `k == size`, so `k` stays at `n` and you do a full
`n` rotations to land back exactly where you started. Correct answer, `n²` wasted
work. **Drop the ternary entirely** — a bare `k %= size` handles all three cases
(`k < n`, `k == n`, `k > n`) in one line.

---

## Approach 2 — Circle and cut (optimal)

### Idea
1. Walk to the tail, counting size. **Keep the tail reference.**
2. `tail.next = head` → the list is now a **circle**. Nothing lost, nothing broken.
3. Walk from `head` to the **new tail**.
4. `newHead = newTail.next`, then `newTail.next = null` → break the circle.

```java
public ListNode rotateRight(ListNode head, int k) {
    if (head == null) return head;

    // 1. size + tail in one pass
    int size = 1;
    ListNode tail = head;
    while (tail.next != null) {
        tail = tail.next;
        size++;
    }

    // 2. reduce k; k == 0 means no work at all
    k %= size;
    if (k == 0) return head;

    // 3. close the circle
    tail.next = head;

    // 4. walk to the new tail
    ListNode newTail = head;
    for (int i = 0; i < size - k - 1; i++) {
        newTail = newTail.next;
    }

    // 5. break the circle
    ListNode newHead = newTail.next;
    newTail.next = null;
    return newHead;
}
```

### Complexity
| | |
|---|---|
| **Time** | **O(n)** — two traversals (`2n` → constant dropped) |
| **Space** | **O(1)** — a fixed number of local references |

Say *"O(n), two passes"* — not "one pass." Being precise reads well.

---

## The index arithmetic (where everyone slips)

This is the part worth re-deriving rather than memorising.

Work in **1-based positions** first, then convert to a **step count**.

New tail is at **position `size - k`** (1-based).

Verify with `[1,2,3,4,5]`:

| k | `size - k` | node at that position | expected output | new tail ✓ |
|---|---|---|---|---|
| 1 | 4 | `4` | `5,1,2,3,4` | ✓ |
| 2 | 3 | `3` | `4,5,1,2,3` | ✓ |
| 3 | 2 | `2` | `3,4,5,1,2` | ✓ |

**Position → step count:** `head` is already position 1, so reaching position `p`
takes `p - 1` steps. Hence:

```
steps = (size - k) - 1 = size - k - 1
```

**You never need the new head's index.** Once you're standing on `newTail`,
`newTail.next` hands you the new head for free.

> **Singly linked list rule:** you can never get `prev` from a node — there's no
> back pointer. So always walk to the node *before* the cut, never to the cut
> itself. This is the same reason LC 19 uses a dummy node.

---

## Dry run — `[1,2,3,4,5]`, `k = 2`

```
Start:      1 → 2 → 3 → 4 → 5 → null
                                tail

size = 5,  k = 2 % 5 = 2

Close circle:
            ┌───────────────────┐
            ↓                   │
            1 → 2 → 3 → 4 → 5 ──┘

steps = size - k - 1 = 5 - 2 - 1 = 2

newTail walk:
  i=0: newTail = 1
  i=1: newTail = 2
  i=2: newTail = 3   ← loop ends (i < 2 fails at i=2... )
```
Careful — the loop is `for (i = 0; i < 2; i++)`, so it runs **twice**:
```
  start:  newTail = 1
  i=0:    newTail = 2
  i=1:    newTail = 3   ← 2 steps taken, landed on position 3 ✓
```
```
newHead = newTail.next = 4
newTail.next = null

Result:     4 → 5 → 1 → 2 → 3 → null   ✓
```

---

## Edge cases

| Case | Handled by |
|---|---|
| `head == null` | explicit guard (the only load-bearing one) |
| single node (`n == 1`) | `k % 1 == 0` → early return |
| `k == 0` | `0 % n == 0` → early return |
| `k == n` | `n % n == 0` → early return |
| `k >> n` (e.g. 2·10⁹) | `k %= size` |

Note the guards `head.next == null` and `k == 0` in Approach 1 are **redundant**
once `k %= size` and `if (k == 0)` are in place. Keeping them is a harmless fast
path; knowing they're dead code is the point.

---

## "But LeetCode says 0ms / beats 100%"

The brute force also reports 0ms. That does **not** mean it's optimal.

- `n <= 500`, so worst case is ~500 × 499 ≈ **250k pointer hops**. A modern CPU
  does that in well under a millisecond. O(n²) and O(n) both round to 0ms here.
- When hundreds of submissions all report 0ms, the percentile is a tie-break on
  **measurement jitter**, not a signal about the algorithm. Resubmit the same
  code and watch it swing.

**The real test:** *would it still pass if `n` were 10⁵?*
- Brute force: ~10¹⁰ operations → TLE.
- Circle-and-cut: 2 × 10⁵ → fine.

That gap is what an interviewer is probing when they ask "can you do better?"

---

## On "isn't the extra pointer wasted memory?"

Keeping a separate `newTail` walker instead of mutating `head`:

- Costs **one word** (4–8 bytes) on the stack. Space is O(1) either way — it
  doesn't grow with `n`.
- Buys **readability**: `head` means one thing for the whole method. A reader
  tracing `head = head.next` in a loop has to keep asking "is `head` still the
  head right now?"
- Buys **debuggability**: you still have a handle on the original list.

> **Rule of thumb:** optimise memory when it *scales with input*. A fixed number
> of local references never does. Where it *would* matter: allocating a new node
> per element, or a `HashMap` sized to `n` — that's genuine O(n) auxiliary space.

---

## Takeaways

1. **Look for the single relink.** If a list operation looks like `k` repeated
   moves, ask whether it's really one cut. Repeated traversal is the smell.
2. **`k %= n` before anything else.** Never a ternary — the bare modulo covers
   `k < n`, `k == n`, `k > n` in one line.
3. **`size = 1` + `while (tail.next != null)`** counts nodes *and* leaves you
   holding the tail. Both are needed. Starting at 0 counts edges.
4. **Derive indices in 1-based positions, then subtract 1 for step count.**
   This is where the off-by-ones live.
5. **Walk to the node *before* the cut** — singly linked lists have no `prev`.
6. **Judge runtime ≠ optimality.** Reason from constraints, not the leaderboard.

### Next problems using this arithmetic
- **LC 19** — Remove Nth From End. Same position→step conversion, but a
  **two-pointer gap trick skips the size pass entirely**. Good contrast to this one.
- **LC 86** — Partition List. Cut and relink with two dummy heads.
- **LC 142** — Cycle detection. Here you *build* a cycle deliberately; there you
  detect an accidental one.