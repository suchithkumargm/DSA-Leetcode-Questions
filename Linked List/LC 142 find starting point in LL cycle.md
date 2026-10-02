# LC 142 — Linked List Cycle II / Find the Starting Point of a Cycle

**Topic:** Linked List · Two Pointers · Floyd's Cycle Detection (Tortoise & Hare)
**Difficulty:** Medium
**Pattern:** Fast/slow pointer + reset-to-head phase
**Related:** LC 141 (detect only), LC 287 (same algorithm on an implicit list)

---

## 1. Problem

Given the head of a linked list, return the node **where the cycle begins**. If there is no cycle,
return `null`.

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

Genuinely correct, and the first node you revisit *is* the cycle entry — you enter the loop from
outside exactly once. Time `O(n)`.

**Why it's not the answer:** space is `O(n)` and the problem explicitly asks for `O(1)`. Offering it
first as a warm-up is fine; stopping there means the question hasn't been answered.

> Store **node references**, not values. Values can repeat legitimately. This works because
> `ListNode` doesn't override `equals`/`hashCode`, so the set uses reference identity.

---

## 3. What is Floyd's cycle detection?

Also called the **tortoise and hare**. It answers two questions about any sequence that eventually
repeats, using only **two pointers and no extra memory**:

1. Is there a cycle?
2. Where does it begin (and how long is it)?

> **On the name:** universally attributed to Robert W. Floyd, though he appears never to have
> published it himself — the attribution comes from Knuth's *TAOCP*. Don't make a point of the
> history in an interview; just know "Floyd's" is the expected name.

### The general setting

It applies to any **deterministic successor function** `f` on a finite set of states. Start at `x0`
and iterate:

```
x0 -> f(x0) -> f(f(x0)) -> f(f(f(x0))) -> ...
```

The state space is finite but the sequence is infinite, so some state must repeat — and once it
does, the sequence is locked in forever. Every such sequence has a **rho shape** (ρ): a straight
tail feeding into a ring.

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

The linked list is just the most literal instance.

### The two phases

**Phase 1 — detection.** Move `slow` by 1 and `fast` by 2. If `fast` reaches `null`, no cycle. If
they land on the same node, a cycle exists. (Naive alternative: compare every node against every
earlier one — `O(n²)` time.)

**Phase 2 — location.** Reset one pointer to the start, advance both by 1. Where they meet is the
entry. Proof in §5.

### ⚠️ Why speeds 1 and 2 specifically

**For detection**, what matters is that **`slow` moves at speed 1**. Then after slow enters the
cycle, its position at time `t` is `(t − L1) mod C` and fast's is `(bt − L1) mod C`; they coincide
when `(b−1)t ≡ 0 (mod C)`, and since `t` takes *every* integer value, `t = C` always works. So any
`fast` speed `b > 1` detects — as long as slow moves by 1.

**With both speeds greater than 1, detection can fail.** Counterexample: `C = 2`, slow at speed 2,
fast at speed 4, starting one apart. Each step slow advances 2 ≡ 0 (mod 2) and fast advances
4 ≡ 0 (mod 2), so the gap stays 1 forever and they never meet.

**For location**, the `2×` ratio is load-bearing — it's what collapses the algebra in §5 to
`L1 = (n−1)C + (C − L2)`. Change the ratio and the reset trick no longer lands on the entry.

> **So: other speeds may detect; only 1-and-2 locates.**

### Where it shows up

- Cycle detection in linked lists — LC 141, 142
- Duplicate detection in constrained arrays — LC 287
- Happy Number — LC 202
- Pollard's rho — integer factorisation
- Detecting infinite loops in state machines and interpreters

> **One-line summary:** Floyd's finds a repeat in an iterated sequence in `O(n)` time and `O(1)`
> space, by racing two pointers at different speeds instead of remembering where you've been.

---

## 4. Intuition — why fast/slow works

Two runners on a track, one at 2× the other's speed.

- **No cycle:** the fast runner reaches the end and the race is over.
- **Cycle:** both eventually end up inside the loop. Once both are inside, fast closes the gap by
  exactly **1 node per iteration**. A gap that shrinks by 1 every step inside a finite ring must
  reach 0 — **it cannot skip over zero.** They must meet.

The meeting proves a cycle exists — but the meeting point is **not** the entry. That's phase 2's job.

---

## 5. The key insight — why resetting to head finds the entry

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

Read the last line out loud:

> **The distance from `head` to the entry equals the distance from the meeting point forward to the
> entry, plus some whole number of extra laps.**

Extra laps don't matter — a pointer looping the cycle lands back where it was.

### The consequence

Put one pointer back at `head`, leave the other at the meeting point, advance **both one step at a
time**. After exactly `L1` steps:

- pointer A (from head) is at the entry — it walked `L1`
- pointer B (from the meeting point) is at the entry — it walked `(n−1)` full laps then `C − L2`

They collide precisely at the entry. ∎

> **Recall trigger:** `nC = L1 + L2`. Everything else falls out of that one line.

### Why slow never laps more than once

A sanity check that the derivation is sound. By the time slow enters the cycle, fast is already
inside it. The gap then shrinks by 1 per iteration, so they meet within at most `C` iterations.
Slow therefore travels **less than one full lap** inside the cycle — which is why
`slow travelled = L1 + L2` with no `+ kC` term.

---

## 6. Solution

Two structures, both correct. The second is preferable.

### Version A — break, then disambiguate after the loop

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) break;
    }

    // Loop exited naturally => fast hit the end => no cycle
    if (fast == null || fast.next == null) return null;

    slow = head;
    while (slow != fast) {
        slow = slow.next;
        fast = fast.next;
    }
    return fast;
}
```

The post-loop test is correct: if they broke on a meeting, `fast` is inside the cycle so
`fast.next` cannot be null; if the loop exited naturally, one of the two is null.

### Version B — return from inside, `null` after the loop ⭐

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;

    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;

        if (slow == fast) {                 // cycle found
            slow = head;
            while (slow != fast) {
                slow = slow.next;
                fast = fast.next;
            }
            return slow;                    // the entry
        }
    }
    return null;                            // fell out => no cycle
}
```

**Why this is better:** falling out of the loop *means* no cycle, with nothing to re-derive. Version
A's correctness depends on the reader checking that `fast.next != null` after a break — a step
Version B removes entirely. Same shape as the clean-exit structure in LC 141.

### Details worth defending

- **`fast != null && fast.next != null`** — both checks mandatory, in that order. `fast.next.next`
  dereferences two links ahead, so you must confirm one link ahead exists first; `&&`
  short-circuits.
- **Both pointers start at `head`**, not `fast = head.next`. The §5 algebra assumes equal start; the
  offset variant meets at a different point and the reset trick no longer lands on the entry.
- **Empty list:** `head == null` → `fast != null` fails → returns `null`. No special case needed.
- **Self-loop (single node pointing to itself):** phase 1 meets at that node, phase 2's
  `while (slow != fast)` is immediately false, returns it. `L1 = 0` works.

---

## 7. Common pitfalls

### 🚩 Guarding only `fast.next != null`

```java
while (fast.next != null) {        // 🚩 NPE
    slow = slow.next;
    fast = fast.next.next;
    if (slow == fast || fast == null) break;
}
```

`fast` itself becomes null inside the body, and the *next* evaluation of `fast.next` throws.

Trace `1→2→null`: iteration 1 computes `fast = fast.next.next` where `fast.next` is already the last
node, so `fast` becomes null; the loop condition then dereferences it. On `1→2` specifically it's
worse — `2.next` is null, so `fast.next.next` NPEs inside the body before any check runs.

> **You cannot null-check `fast` *after* already dereferencing it twice.** Both guards go in the
> loop condition, before either hop.

### 🚩 Using `slow == fast` as the post-loop cycle test

```java
while (fast != null && fast.next != null) { ...; if (slow == fast) break; }
if (slow == fast) { /* phase 2 */ }     // 🚩 conflates two situations
```

`slow == fast` is true in **two** different cases:
- they met → cycle ✅
- **the loop never ran** → both are still `head`

Trace `[1]` with no cycle: `fast.next` is null, so the loop body never executes;
`slow == fast == head`; phase 2's inner loop is immediately false; you **return `head`** instead of
`null`.

Use `fast == null || fast.next == null` (Version A) or return from inside (Version B).

### Other slips

- **Returning the meeting point.** It's almost never the entry — only when `L1 = 0`.
- **Leaving `fast` at 2× in phase 2.** Both move one step at a time; this is the classic break.
- **Starting the pointers at different offsets.** Breaks the §5 algebra.
- **Stopping at the HashSet** when the prompt says O(1) space.

---

## 8. Dry run — `[3,2,0,-4]`, tail connects to index 1

Nodes: `A(3) → B(2) → C(0) → D(-4) → back to B`. So `L1 = 1`, `C = 3`.

**Phase 1:**

| iter | slow | fast | met? |
|------|------|------|------|
| start| A    | A    | —    |
| 1    | B    | C    | no   |
| 2    | C    | B    | no   |
| 3    | D    | D    | **yes** |

Meeting point = `D`, so `L2 = 2` (B → C → D).
Check the identity: `nC = L1 + L2` → `1 × 3 = 1 + 2` ✓

Note at iteration 2 the pointers **cross without landing together** — the gap is still closing by 1,
so they meet on the next step. That crossing is exactly what "can't skip zero" covers.

**Phase 2:** reset `slow = A`, `fast` stays at `D`.

| iter | slow | fast | equal? |
|------|------|------|--------|
| start| A    | D    | no     |
| 1    | B    | B    | **yes** |

Return `B` — the node with value 2 ✓

---

## 9. Edge cases

| case | result | why |
|---|---|---|
| `head == null` | null | `fast != null` fails immediately |
| single node, no cycle | null | loop never runs — ⚠️ the case that breaks the `slow == fast` test |
| single node, self-loop | that node | meet on iteration 1, `L1 = 0`, phase 2 exits immediately |
| two nodes, cycle | correct entry | |
| entire list is the cycle (`L1 = 0`) | head | phase 2's inner loop never runs |
| long tail, tiny cycle | correct entry | slow walks the tail, is caught within one lap |

---

## 10. Complexity

| | |
|---|---|
| **Time** | `O(n)` |
| **Space** | `O(1)` |

**Time, precisely:** phase 1 runs at most `L1 + C` iterations (slow reaches the cycle in `L1` steps,
then is caught within `C` more). Phase 2 runs exactly `L1`. Both bounded by `n`.

**Space:** two pointers. No allocation, no recursion stack.

---

## 11. Interview talking points

**"Prove it terminates."** In the cycle, the gap between fast and slow decreases by exactly 1 each
iteration. A quantity changing by 1 cannot skip zero.

**"Why does resetting to head work?"** Recite `nC = L1 + L2 ⇒ L1 = (n−1)C + (C − L2)`. Don't
hand-wave — it's the entire point of the question.

**"Can you also return the cycle length?"** Yes — from the meeting point, walk forward counting
until you return to it. `O(C)` time, `O(1)` space.

**"What if you could modify the list?"** Mark visited nodes (null out `next`, flip a flag).
Destructive, and the wrong answer unless explicitly permitted — say so yourself rather than waiting
to be told.

**"How does this relate to LC 287?"** Find the Duplicate treats the array as `f(i) = nums[i]`,
forming an implicit linked list. Two indices holding the same value point at the same node, which
creates a cycle — and the duplicate **is** the entry. Identical algorithm, different disguise.

---

## 12. Pattern

> **Floyd's = detect + locate, in two phases.**
> Phase 1: `slow += 1`, `fast += 2` until they meet (or fast falls off).
> Phase 2: reset one to the start, advance both by 1 — the meeting point is the entry.

Applies to any **deterministic successor function on a finite state space** — linked lists,
array-index chains, iterated number functions.

- LC 141 — Linked List Cycle *(phase 1 only)*
- LC 142 — this one *(both phases)*
- LC 287 — Find the Duplicate Number *(implicit list over array indices)*
- LC 202 — Happy Number *(implicit list over digit-square sums; phase 1 only)*

---

## 13. Takeaways

1. **Both null checks in the loop condition, in order.** You can't null-check a pointer you've
   already dereferenced twice.
2. **Don't use `slow == fast` as the post-loop cycle test** — it's also true when the loop never ran.
3. **Return from inside the loop; `return null` after it.** Removes the need to disambiguate exits.
4. **The meeting point is not the entry.** Phase 2 is mandatory.
5. **Phase 2 moves both pointers at speed 1.**
6. **Both pointers start at `head`** — the algebra depends on it.
7. **`nC = L1 + L2`** is the one line to memorise.
8. **Slow moving at speed 1 is what guarantees detection**; the 2× ratio is what makes *location*
   work.