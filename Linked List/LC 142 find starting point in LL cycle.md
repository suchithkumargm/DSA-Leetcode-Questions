# LC 142 — Linked List Cycle II/Find starting point in LL cycle

**Topic:** Linked List · Two Pointers · Floyd's Cycle Detection (Tortoise & Hare)
**Difficulty:** Medium
**Pattern:** Fast/slow pointer + reset-to-head phase
**Related:** LC 141 (detect only), LC 287 (Find the Duplicate Number — same algorithm on an implicit list)

---

## 1. Problem

Given the head of a linked list, return the node **where the cycle begins**.
If there is no cycle, return `null`.

Explicit follow-up constraint: **solve it using O(1) memory.**

```
head -> 3 -> 2 -> 0 -> -4
             ^           |
             |___________|

answer = the node with value 2 (the entry point)
```

---

## 2. The naive approach and why it's rejected

### HashSet of visited nodes

```java
// Works, but violates the follow-up constraint
Set<ListNode> seen = new HashSet<>();
while (head != null) {
    if (!seen.add(head)) return head;   // first repeat IS the entry
    head = head.next;
}
return null;
```

This is genuinely correct, and the first node you revisit *is* the cycle entry
(you enter the loop from outside exactly once). Time `O(n)`.

**Why it's not the answer:** space is `O(n)`. The problem explicitly asks for
`O(1)`. In an interview, offering this first is fine as a warm-up — but you must
follow it with Floyd's, or the question hasn't been answered.

> Note: store **node references**, not values. Values can repeat legitimately.

---

## 3. What is Floyd's Cycle Detection Algorithm?

Also called the **tortoise and hare algorithm**, published by Robert W. Floyd in the
1960s. It answers two questions about any sequence that eventually repeats, using
only **two pointers and no extra memory**:

1. **Is there a cycle?**
2. **Where does the cycle begin, and how long is it?**

### The general setting

It applies to any **deterministic successor function** `f` on a finite set of states.
Start at some `x0` and iterate:

```
x0 -> f(x0) -> f(f(x0)) -> f(f(f(x0))) -> ...
```

Because the state space is finite but the sequence is infinite, some state must
repeat. Once it does, the sequence is locked into a loop forever. Every such
sequence therefore has a **rho shape** (ρ) — a straight tail feeding into a ring:

```
        tail (length L1)          cycle (length C)
    x0 -> o -> o -> o -> ENTRY -> o -> o
                          ^             |
                          |_____________|
```

For a linked list, `f(node) = node.next`. But `f` can be anything:
- `f(i) = nums[i]` → array-index chains (LC 287)
- `f(n) = sum of squares of digits of n` → LC 202
- `f(x) = (x² + c) mod n` → Pollard's rho factorisation

The linked list is just the most literal instance of the pattern.

### The two phases

**Phase 1 — detection.** Move `slow` by 1 and `fast` by 2 each step. If `fast`
reaches the end (`null`), there is no cycle. If they land on the same node, a cycle
exists. Naive alternative: compare every node against every earlier node — `O(n²)`
time. Floyd's does it in `O(n)` with `O(1)` space.

**Phase 2 — location.** Reset one pointer to the start, then advance both by 1 step
each. Where they meet is the cycle entry. The proof is in the next section.

### Why 1 and 2 specifically?

Any speeds `a < b` will work for *detection* — the gap changes by `b - a` per step
and must hit a multiple of `C`. But the **`2x` ratio is what makes phase 2's algebra
collapse to `L1 = (n-1)C + (C - L2)`**. Change the ratio and the reset trick stops
landing on the entry. So: other speeds detect, only 1-and-2 locates this cleanly.

### Where the name shows up

- **Cycle detection in linked lists** — LC 141, 142
- **Duplicate detection in constrained arrays** — LC 287
- **Happy Number** — LC 202
- **Pollard's rho** — integer factorisation in cryptanalysis
- **Detecting infinite loops** in state machines and interpreters

> **One-line summary:** Floyd's algorithm finds a repeat in an iterated sequence in
> `O(n)` time and `O(1)` space, by racing two pointers at different speeds instead of
> remembering where you've been.

---

## 4. Intuition — why fast/slow works

Two runners on a track, one at 2× the speed of the other.

- **No cycle:** the fast runner reaches the end (`null`) and the race is over.
- **Cycle:** both runners eventually end up inside the loop. Once both are inside,
  fast closes the gap by exactly **1 node per iteration**. A gap that shrinks by 1
  every step and lives in a finite ring must reach 0. They *must* meet.

```
        entry
head ----> O ------> O
           ^         |
           |         v
           O <------ O
              (meeting happens somewhere in here)
```

The meeting proves a cycle exists — but the meeting point is **not** the entry.
That's phase 2's job.

---

## 5. The key insight — why resetting to head finds the entry

This is the part to actually memorise, because it looks like magic otherwise.

### Setup

```
L1 = distance from head to the cycle entry
L2 = distance from entry to the meeting point (in travel direction)
C  = cycle length

head ---- L1 ----> ENTRY ---- L2 ----> MEET
                     ^                   |
                     |___ (C - L2) ______|
```

### The algebra

At the moment they meet:

```
slow travelled = L1 + L2
fast travelled = L1 + L2 + nC        (fast lapped the cycle n times, n >= 1)
```

Fast moves twice as far as slow:

```
L1 + L2 + nC = 2 * (L1 + L2)
          nC = L1 + L2
          L1 = nC - L2
             = (n-1)C + (C - L2)
```

Read that last line out loud:

> **The distance from `head` to the entry equals the distance from the meeting
> point forward to the entry, plus some whole number of extra laps.**

Extra laps don't matter — a pointer looping the cycle lands back where it was.

### The consequence

Put one pointer back at `head`, leave the other at the meeting point, and advance
**both one step at a time**. After exactly `L1` steps:

- pointer A (from head) is at the entry — it walked `L1`
- pointer B (from meeting) is at the entry — it walked `(n-1)` full laps then `C - L2`

They collide precisely at the cycle entry. ∎

> **Recall trigger:** `nC = L1 + L2`. Everything else falls out of that one line.

### Why slow never laps more than once

Sanity check that the derivation is sound: by the time slow enters the cycle, fast
is already inside it. The gap then shrinks by 1 per iteration, so they meet within
at most `C` iterations. Slow therefore travels **less than one full lap** inside the
cycle, which is why `slow travelled = L1 + L2` with no `+ kC` term.

---

## 6. Solution

```java
public class Solution {
    public ListNode detectCycle(ListNode head) {
        ListNode slow = head;
        ListNode fast = head;

        // Phase 1: detect a meeting point
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) {
                break;
            }
        }

        // Loop exited naturally => hit the end => no cycle
        if (fast == null || fast.next == null) {
            return null;
        }

        // Phase 2: reset one pointer, advance both at the same speed
        slow = head;
        while (slow != fast) {
            slow = slow.next;
            fast = fast.next;
        }

        return fast;   // == slow == cycle entry
    }
}
```

### Details worth defending in an interview

- **`fast != null && fast.next != null`** — both checks are mandatory. `fast.next.next`
  dereferences two links ahead, so you must confirm one link ahead exists first.
  Java's `&&` short-circuits, so the order of the two checks matters.
- **Both start at `head`, not `slow = head; fast = head.next`.** The algebra above
  assumes equal start. The offset variant meets at a different point and the
  reset trick no longer lands on the entry.
- **The check after the loop distinguishes the two exit routes.** The loop can end
  by `break` (they met → cycle) or by the condition failing (fast fell off the end
  → no cycle). Testing `fast == null || fast.next == null` identifies the second.
- **Empty list:** `head == null` → both pointers null → loop never runs → `fast == null`
  → returns `null`. Handled without a special case.
- **Self-loop (single node pointing to itself):** phase 1 meets at that node,
  phase 2's `while (slow != fast)` is immediately false, returns it. `L1 = 0` works.

---

## 7. Dry run — `[3,2,0,-4]`, tail connects to index 1

Nodes: `A(3) -> B(2) -> C(0) -> D(-4) -> back to B`
Here `L1 = 1`, `C = 3`.

**Phase 1:**

| iter | slow | fast | met? |
|------|------|------|------|
| start| A    | A    | —    |
| 1    | B    | C    | no   |
| 2    | C    | B    | no   |
| 3    | D    | D    | **yes → break** |

Meeting point = `D`. So `L2 = 2` (B → C → D).
Check the identity: `nC = L1 + L2` → `1 * 3 = 1 + 2` ✓

**Phase 2:** reset `slow = A` (head), `fast` stays at `D`.

| iter | slow | fast | equal? |
|------|------|------|--------|
| start| A    | D    | no     |
| 1    | B    | B    | **yes → exit** |

Return `B` — the node with value 2. ✓

---

## 8. Complexity

| | |
|---|---|
| **Time** | `O(n)` |
| **Space** | `O(1)` |

**Time, more precisely:** phase 1 runs at most `L1 + C` iterations (slow reaches the
cycle in `L1` steps, then is caught within `C` more). Phase 2 runs exactly `L1`
iterations. Total is linear in the number of nodes — no constant-factor surprises
worth mentioning beyond "two passes, both bounded by n".

**Space:** two pointers, nothing else. No allocation, no recursion stack.

---

## 9. Interview talking points

- **"Prove it terminates."** In the cycle, the gap between fast and slow decreases
  by exactly 1 each iteration. A strictly decreasing non-negative integer in a
  finite ring reaches 0.
- **"Why does resetting to head work?"** Recite `nC = L1 + L2 ⇒ L1 = (n-1)C + (C - L2)`.
  Don't hand-wave this; it's the entire point of the question.
- **"Can you also return the cycle length?"** Yes — from the meeting point, walk
  forward counting until you return to it. That's `C`, in `O(C)` time, `O(1)` space.
- **"What if you're allowed to modify the list?"** You could mark visited nodes
  (e.g. null out `next`, or flip a flag). Destructive, and usually the wrong answer
  to give unless explicitly permitted.
- **"How does this relate to LC 287?"** Find the Duplicate Number treats the array
  as a function `i -> nums[i]`, forming an implicit linked list. A duplicate value
  means two indices point to the same node, which creates a cycle — and the
  duplicate *is* the cycle entry. Identical algorithm, different disguise.

---

## 10. Pattern to carry forward

> **Floyd's cycle detection = detect + locate, in two phases.**
> Phase 1: `slow += 1`, `fast += 2` until they meet (or fast falls off).
> Phase 2: reset one to start, advance both by 1, meeting point is the entry.

Applies whenever you have a **deterministic successor function** on a finite state
space — linked lists, array-index chains, iterated number functions.

- LC 141 — Linked List Cycle *(phase 1 only)*
- LC 142 — Linked List Cycle II *(this one — both phases)*
- LC 287 — Find the Duplicate Number *(implicit list over array indices)*
- LC 202 — Happy Number *(implicit list over digit-square-sum; phase 1 only)*

---

## 11. Mistakes to not repeat

1. Don't return the meeting point. It is almost never the entry — only when `L1 = 0`.
2. Don't check `fast.next != null` before `fast != null`. NPE.
3. Don't start the pointers at different offsets; the phase-2 math assumes both start at `head`.
4. Don't use a HashSet and stop there when the prompt says `O(1)` space.
5. In phase 2, both pointers move **one step at a time**. Leaving `fast` at 2× is the
   classic slip and breaks everything.