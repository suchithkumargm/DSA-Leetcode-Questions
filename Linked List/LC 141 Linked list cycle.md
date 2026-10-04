# LC 141 — Linked List Cycle

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** Fast & slow pointers (Floyd's cycle detection)

---

## 1. Problem

Given `head` of a linked list, return `true` if the list has a cycle — i.e. some node can be
reached again by continuously following `next`.

**Follow-up:** solve it in **O(1) memory**.

Constraints: `0 ≤ n ≤ 10⁴`.

---

## 2. Approaches

| # | Approach | Time | Aux Space | Verdict |
|---|---|---|---|---|
| 1 | HashSet of visited nodes | O(n) | O(n) | Works. Fails the follow-up. |
| 2 | Mutate nodes to mark them visited | O(n) | O(1) | Destroys the input — see §6 |
| 3 | Count steps, bail after n iterations | O(n) | O(1) | Needs a known `n`; hacky |
| **4** | **Floyd's tortoise & hare** | **O(n)** | **O(1)** | ✅ the answer |

### HashSet version

State this first in an interview, then improve on it.

```java
Set<ListNode> seen = new HashSet<>();
while (head != null) {
    if (!seen.add(head)) return true;   // add() returns false if already present
    head = head.next;
}
return false;
```

> ⚠️ **It must be a set of nodes, not of values.** Duplicate *values* are legal in a cycle-free
> list. This works because `ListNode` doesn't override `equals`/`hashCode`, so the set uses
> **reference identity** — the same Java property that makes LC 138's `HashMap<Node, Node>` work.

---

## 3. Floyd's cycle detection

Two pointers start at the head. `slow` advances 1 node per step; `fast` advances 2.

- **No cycle** → `fast` runs off the end and hits `null`.
- **Cycle** → neither pointer can ever exit, so both are trapped circling forever.

### The objection every interviewer raises

> *"They're both in the loop — but couldn't `fast` keep jumping **over** `slow` and never land on
> it?"*

### The answer — one sentence, no formula

> Once both pointers are inside the cycle, `fast` gains **exactly one position** on `slow` per
> iteration (2 − 1 = 1).
>
> **A gap that changes by exactly 1 each step cannot skip over zero.** It has to land on it.

That's the entire proof. `fast` can't leap past `slow` because the gap shrinks by 1, never by 2.

> ⚠️ **There is no formula for LC 141.** If you find yourself reaching for arithmetic, you're
> thinking of **LC 142**, which asks *where* the cycle starts — see §7.

### Complexity follows directly from the same argument

- **No cycle:** `fast` reaches the end in n/2 steps → **O(n)**.
- **Cycle:** `slow` walks at most the **tail length** to enter the loop, then at most the **cycle
  length** to be caught. Since `tail + cycle ≤ n` → **O(n)**.

**O(n) time · O(1) space**, either way.

---

## 4. Solution

```java
public class Solution {
    public boolean hasCycle(ListNode head) {
        ListNode slow = head;
        ListNode fast = head;

        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) return true;
        }
        return false;
    }
}
```

**O(n) time · O(1) space.** No null guard, no post-loop branch.

### Why the loop condition is `fast != null && fast.next != null`

`fast` takes **two** hops per iteration, so **both** must be checked:

| list | drop `fast != null` | drop `fast.next != null` |
|---|---|---|
| `[]` | NPE immediately | NPE immediately |
| `[1]` | — | `fast.next.next` on the only node → NPE |
| `[1,2]` | NPE on the 2nd iteration | NPE |

⚠️ **Order matters.** Java's `&&` short-circuits, so `fast != null` must come first — flip them and
you NPE on an empty list.

`slow` needs no check: it can never pass `fast`, so if `fast` is alive, `slow` is.

---

## 5. Common pitfalls

### Checking for the meeting *before* moving

The `slow == fast` test must come **after** both pointers advance. Check it first and
`slow == fast == head` on entry, so every list returns `true`.

### Putting the meeting check in the loop condition

```java
while (fast != null && fast.next != null && slow != fast) {   // 🚩
    slow = slow.next;
    fast = fast.next.next;
}
if (slow == fast) return true; else return false;             // 🚩
```

This is correct but forces an ambiguous exit — you can't tell from the loop ending *why* it ended,
so you need a post-loop `if` to disambiguate. Keep the check **inside** the body and the two exits
are unambiguous: returning from inside means cycle, falling out means no cycle.

### `if (x) return true; else return false;`

Converting a boolean into a boolean. Write `return slow == fast;`.

### Starting `fast` at `head.next`

```java
ListNode fast = head.next;                              // 🚩
if (head == null || head.next == null) return false;    // ...now this guard is mandatory
```

The offset start is *correct*, but it NPEs on a null head — which is why it forces the guard. Start
both pointers at `head` and the guard becomes unnecessary: `fast != null` fails on the first
evaluation and the loop never runs.

**This is the head-special-case pattern in disguise** — a hand-written guard patching a choice made
two lines earlier.

---

## 6. Why mutating the list is a trap

A tempting O(1)-space "trick": as you walk, set each visited node's `next` to a sentinel (or to
itself, or to null). Arriving at the sentinel means there's a cycle.

**Never lead with this.** It destroys the input list, which:
- is disqualifying in any real codebase,
- breaks if the caller reuses the list,
- fails outright if nodes are shared or the structure is immutable.

If you mention it, attach the objection yourself.

---

## 7. LC 142 — where the arithmetic actually lives

**LC 141** asks *"is there a cycle?"* → the +1-gap argument suffices.
**LC 142** asks *"which node starts the cycle?"* → this genuinely needs a derivation.

**The result:** after `slow` and `fast` meet, reset one pointer to `head` and advance **both one
step at a time**. They meet at the cycle's entry point.

**Why.** Let `L` = tail length, `C` = cycle length, `x` = distance from cycle entry to the meeting
point.

- `slow` travelled `L + x`; `fast` travelled `2(L + x)`.
- The difference, `L + x`, must be a whole number of laps: `L + x ≡ 0 (mod C)`.
- So walking `L` more steps from the meeting point lands on the entry — and walking `L` from the
  head lands there too.

LC 142 is asked considerably more often than 141. Do it while this is warm.

---

## 8. Dry run

### Cycle: `3 → 2 → 0 → -4 → (back to 2)`

Nodes A(3) → B(2) → C(0) → D(-4) → B.

| iter | slow | fast | meet? |
|---|---|---|---|
| start | A | A | not checked before the first move |
| 1 | B | C | no |
| 2 | C | B | no |
| 3 | D | D | ✅ return true |

Note at iteration 2 the pointers **cross without landing together** — the gap is still closing by 1
each step, so they meet on the next iteration. That crossing is exactly what "can't skip zero"
covers.

### No cycle: `1 → 2 → 3 → 4 → null`

| iter | slow | fast | next loop condition |
|---|---|---|---|
| 1 | 2 | 3 | `fast=3` ok, `fast.next=4` ok → continue |
| 2 | 3 | **null** | `fast != null` fails → exit |

Return `false` ✅

---

## 9. Edge cases

| case | result | why |
|---|---|---|
| `head == null` | false | `fast != null` fails immediately — no guard needed |
| single node, no cycle | false | `fast.next` is null → exit |
| single node, self-cycle (`A.next = A`) | true | both advance to A, meet on iteration 1 |
| two nodes, cycle | true | meet within 2 iterations |
| long tail, tiny cycle | true | slow walks the tail, then is caught within one cycle length |

---

## 10. Interview Q&A

**Q: Why does `fast` have to catch `slow`?**
Once both are in the cycle, the gap closes by exactly 1 per iteration. A quantity changing by 1
can't skip zero, so they must coincide.

**Q: Why move 2 and 1 — would 3 and 1 work?**
Yes, any pair with different speeds eventually meets, but the gap then changes by 2 per step and
can skip a value. It still terminates (the gap cycles through residues mod `C`), but the "can't
skip zero" argument no longer applies directly and the proof gets messier. 2-and-1 is the clean
choice, and it's required for LC 142's entry-point derivation.

**Q: Time complexity when there *is* a cycle?**
Still O(n). `slow` covers at most `tail + cycle` nodes, and `tail + cycle ≤ n`.

**Q: Can you do it without extra space and without modifying the list?**
That's exactly what Floyd's gives you.

**Q: What if you needed the cycle's length?**
Once they meet, keep one pointer fixed and walk the other around until it returns. The number of
steps is `C`.

---

## 11. Pattern

> **A gap that changes by exactly 1 each step cannot skip over zero — so a faster pointer inside a
> bounded region must eventually land on the slower one.**

Note this says nothing about linked lists, which is why it transfers to any "functional graph"
cycle question: LC 142, LC 202 (Happy Number), LC 287 (Find the Duplicate Number).

---

## 12. Takeaways

1. **Floyd's proof is one sentence, not a formula.** The gap closes by exactly 1 ⇒ it must hit zero.
2. **Cycle-case complexity follows from the same argument:** `tail + cycle ≤ n` ⇒ O(n).
3. **Start both pointers at `head`.** The offset start creates the null guard you then have to write.
4. **Check `fast != null && fast.next != null`, in that order** — `&&` short-circuits and `fast`
   takes two hops.
5. **Test for the meeting after moving, and inside the loop body**, not in the loop condition.
6. **`return slow == fast;`** — never `if (x) return true; else return false;`.
7. **The HashSet version must key on nodes, not values** — it relies on reference identity.
8. **Never offer a solution that mutates the input** without attaching the objection yourself.

---
