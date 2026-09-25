# LC 141 — Linked List Cycle (Easy on LC / Tier A)

> **Problem:** Given `head` of a linked list, determine if it has a cycle. A cycle exists if some
> node can be reached again by continuously following `next`.
>
> **Follow-up:** solve it using **O(1) memory**.
>
> Constraints: `0 ≤ n ≤ 10⁴`

**Session record — Sep 25**
| | |
|---|---|
| Outcome | **solved cold, unaided, 13 min**, traced before submitting ✅ |
| Got right | Floyd's two pointers, O(1) space, wrote the survey as comments before coding |
| Missed | **why** fast is guaranteed to catch slow — reached for "some formula" |
| Missed | time complexity in the cycle case (follows from the same argument, no formula) |
| Style | `if (x) return true; else return false;`; meeting check mixed into the loop condition |
| 🟡 Pattern line | **first time any causal content appeared** — 7 problems in (§7) |

---

## 1. The approaches — survey this before coding

| # | Approach | Time | Aux Space | Verdict |
|---|---|---|---|---|
| 1 | HashSet of visited nodes | O(n) | **O(n)** | works, fails the follow-up |
| 2 | Mutate nodes (mark visited / null out `next`) | O(n) | O(1) | ❌ **destroys the input** — never offer this first |
| 3 | Count steps, bail after n iterations | O(n) | O(1) | needs a known `n`; hacky |
| **4** | **Floyd's tortoise & hare** ⭐ | **O(n)** | **O(1)** | ✅ the answer |

### Approach 1 — HashSet (mention it, then improve it)

Walk the list, insert each **node reference** into a `HashSet<ListNode>`. If you ever try to insert
one that's already there, you've looped.

```java
Set<ListNode> seen = new HashSet<>();
while (head != null) {
    if (!seen.add(head)) return true;   // add() returns false if already present
    head = head.next;
}
return false;
```

> ⚠️ **It must be a set of nodes, not of values.** Duplicate *values* are legal in a cycle-free
> list. This relies on `ListNode` not overriding `equals`/`hashCode`, so the set uses **reference
> identity** — the same Java property that makes **LC 138**'s `HashMap<Node, Node>` work.

O(n) space. This is the version to state in 15 seconds and then say *"but the follow-up asks for
O(1), so:"*

---

## 2. Floyd's cycle detection — the fact that makes it work

Two pointers from the head: `slow` moves 1 node per step, `fast` moves 2.

- **No cycle** → `fast` runs off the end and hits `null`. Done.
- **Cycle** → neither can ever exit, so they're both trapped going round forever.

The obvious objection, and the one an interviewer will raise:

> *"Sure they're both in the loop — but couldn't `fast` keep **jumping over** `slow` and never
> land on it?"*

### 🔑 The answer — no formula required

> **Once both pointers are inside the cycle, `fast` gains exactly ONE position on `slow` per
> iteration** (it moves 2, slow moves 1, net +1).
>
> **A gap that changes by exactly 1 each step cannot skip over zero.** It has to land on it.

That's the whole proof. `fast` can't leap past `slow` because the gap shrinks by 1, never by 2.

> 🔴 **I reached for "there's some formula I can't remember."** There isn't one — not for *this*
> problem. The +1-gap argument is the entire justification. (The arithmetic you're thinking of
> belongs to **LC 142**, which asks *where* the cycle starts — see §6.)

### Time complexity — falls straight out of the same argument

- **No cycle:** `fast` reaches the end in n/2 steps → **O(n)**.
- **Cycle:** `slow` walks at most the **tail length** to enter the loop, then at most the **cycle
  length** to be caught. `tail + cycle ≤ n` → **O(n)**.

Same bound either way. **O(n) time, O(1) space.** No formula — just "each pointer covers a bounded
portion of the list."

---

## 3. My solution and the cleanups

### What I submitted first

```java
public boolean hasCycle(ListNode head) {
    if (head == null || head.next == null) return false;    // guard needed only because of ↓

    ListNode slow = head;
    ListNode fast = head.next;                              // offset start

    while (fast != null && fast.next != null && slow != fast) {   // 🚩 3 conditions
        slow = slow.next;
        fast = fast.next.next;
    }

    if (slow == fast) return true;                          // 🚩 boolean → boolean
    else return false;
}
```

Correct — the offset start is a valid variant, verified against a 3-cycle and a cycle-free list.
Three things to tighten:

### 🚩 1. `if (x) return true; else return false;`

You're converting a boolean into a boolean. **`return slow == fast;`**

### 🚩 2. The null guard exists *only* because of the offset start

`fast = head.next` NPEs on a null head, which is what forces the guard. Start `fast = head` and the
guard becomes unnecessary — `head == null` makes the loop condition fail on the first evaluation.

> **This is the head-special-case habit in a different costume** (§16 of the LC 138 doc, §8 of LC
> 25). Same shape: a hand-written guard that exists to patch a choice made two lines earlier.

### 🚩 3. The meeting check doesn't belong in the loop condition

Mixing `slow != fast` into the `while` is what forces the awkward post-loop `if`. Move it inside
and the two exits become unambiguous: **return from inside ⇒ cycle; fall out ⇒ no cycle.**

### ✅ Final

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

**O(n) time · O(1) space.** No guard, no post-loop branch, no offset.

### Why `fast != null && fast.next != null` and not something shorter

`fast` takes **two** hops per iteration, so **both** must be checked. Dropping either gives an NPE:

| list | drop `fast != null` | drop `fast.next != null` |
|---|---|---|
| `[]` | NPE immediately | NPE immediately |
| `[1]` | — | `fast.next.next` on the only node → NPE |
| `[1,2]` | NPE on the 2nd iteration | NPE |

⚠️ **Order matters** — `fast != null` must come first. Java's `&&` short-circuits, so the second
check is never evaluated when `fast` is null. Flip them and you NPE.

`slow` needs no check: it can never pass `fast`, so if `fast` is alive, `slow` is.

---

## 4. Dry run

### Cycle: `3 → 2 → 0 → -4 → (back to 2)`

Name them A(3) → B(2) → C(0) → D(-4) → B.

| iter | slow | fast | meet? |
|---|---|---|---|
| start | A | A | — (not checked before the first move) |
| 1 | B | C | no |
| 2 | C | B | no |
| 3 | D | D | ✅ **return true** |

Note at iter 2 the pointers **cross without landing together** — but the gap is still closing by 1
each step, so they meet on the very next iteration. That crossing is exactly what the "can't skip
zero" argument covers.

### No cycle: `1 → 2 → 3 → 4 → null`

| iter | slow | fast | loop condition next time |
|---|---|---|---|
| 1 | 2 | 3 | `fast=3` ok, `fast.next=4` ok → continue |
| 2 | 3 | **null** | `fast != null` **fails** → exit |

Return false ✅

### Edge cases

| case | result | why |
|---|---|---|
| `head == null` | false | `fast != null` fails immediately — **no guard needed** |
| single node, no cycle | false | `fast.next` is null → exit |
| single node, self-cycle (`A.next = A`) | true | both advance to A, meet on iteration 1 |
| two nodes, cycle | true | meet within 2 iterations |
| long tail + tiny cycle | true | slow walks the tail, then gets caught within one cycle length |

⚠️ **The `slow == fast` check must come AFTER both moves**, not before. Checking first means
`slow == fast == head` at the start and you'd return true for every list.

---

## 5. Approach 2 — why mutating the list is a trap

A tempting O(1)-space "trick": as you walk, set each visited node's `next` to point at a sentinel
(or to itself, or null). If you ever arrive at the sentinel, there's a cycle.

**Never lead with this.** It **destroys the input list**, which:
- is disqualifying in any real codebase,
- breaks if the caller reuses the list,
- and fails outright if nodes are shared or the structure is immutable.

If you mention it at all, mention it *with* the objection attached. Same reasoning as LC 138, where
LeetCode explicitly fails you for leaving the originals mutated.

---

## 6. LC 142 — where the formula actually lives

**LC 141** asks *"is there a cycle?"* → the +1-gap argument is enough.
**LC 142** asks *"which node starts the cycle?"* → this is where arithmetic is genuinely required.

The result: after `slow` and `fast` meet, reset one pointer to `head` and advance **both one step at
a time**. They meet at the cycle's entry point.

Sketch of why — let `L` = tail length, `C` = cycle length, `x` = distance from cycle entry to the
meeting point:
- slow travelled `L + x`; fast travelled `2(L + x)`; the difference `L + x` is a whole number of
  cycle laps.
- So `L + x ≡ 0 (mod C)`, i.e. **`L` and `x` are complementary within the cycle** — walking `L`
  more from the meeting point lands on the entry, and walking `L` from the head does too.

> **Do LC 142 next while this is warm.** It's the same two pointers plus the one derivation, and it
> is far more commonly asked than 141.

---

## 7. 🟡 Pattern line — first causal content in 7 problems

**What I said:**
> *"two pointer slow fast approach, when one moves one step and the other moves double, if there is
> a loop they will eventually meet"*

Run the test — **cross out every word that came from the problem statement**: "two pointer slow
fast", "if there is a loop" → gone. What survives is *"they will eventually meet"*, which **asserts
the conclusion** rather than giving the reason.

**But** this is the first time in seven problems that any causal content appeared at all. That's
movement — the previous six were pure mechanics.

### The fact

> **A gap that changes by exactly 1 each step cannot skip over zero.**

Delete that sentence and you can't answer the "couldn't fast jump over slow?" follow-up. Delete
mine and nothing is lost.

### Running record

| problem | ❌ what I said | ✅ the fact |
|---|---|---|
| 169 | "increment if same, decrement if not" | >n/2 copies can't be fully cancelled by pairing |
| 1552 | "find where monotonicity turns" | if a gap is achievable, every smaller one is too |
| 138 | "add a node next to the original, then separate" | the clone of X **is** X.next |
| 25 | "reverse the group and add it to the list" | reversing a segment turns its head into its tail |
| 232 | "use 2 stacks and transfer between them" | reversing a reversal restores the original order |
| **141** | "they will eventually meet" 🟡 | **a gap changing by 1 can't skip zero** |

---

## 8. One-line pattern

> **A gap that changes by exactly 1 each step cannot skip over zero — so a faster pointer inside a
> bounded region must eventually land on the slower one.**

Note it says nothing about linked lists. That's why it transfers to LC 142, LC 202 (Happy Number),
LC 287 (Find the Duplicate), and any "functional graph" cycle question.

---

## 9. Takeaways

1. **Floyd's proof is one sentence, not a formula.** Gap closes by exactly 1 per step ⇒ it must hit
   zero. Have this ready — it's the standard follow-up.
2. **Cycle-case complexity follows from the same argument:** tail + cycle ≤ n ⇒ O(n). No formula.
3. **`return slow == fast;`** — never `if (x) return true; else return false;`.
4. **Keep the meeting check *inside* the loop.** Mixing it into the loop condition forces an
   ambiguous post-loop branch.
5. **Start both pointers at `head`.** The offset start (`fast = head.next`) is correct but *creates*
   the null guard you then have to write. Same head-special-case habit as LC 25 / 138.
6. **Check `fast != null && fast.next != null`, in that order** — `&&` short-circuits, and `fast`
   takes two hops.
7. **Check for the meeting AFTER moving**, never before.
8. **HashSet version must key on nodes, not values** — relies on reference identity, same property
   as LC 138.
9. **Never offer a solution that mutates the input** without attaching the objection yourself.

---

## 10. Related problems

| # | Problem | Connection |
|---|---|---|
| **142** | Linked List Cycle II | ⭐ **do next** — same pointers, plus the entry-point derivation. Asked far more often. |
| 287 | Find the Duplicate Number | ⭐ Floyd's on an **array** treated as a function graph — the non-obvious application |
| 202 | Happy Number | Floyd's on a number sequence; no list at all |
| 876 | Middle of the Linked List | same two-speed pointers, different payoff |
| 234 | Palindrome Linked List | find middle with two pointers, then reverse half |
| 19 | Remove Nth Node From End | two pointers with a **fixed** gap rather than a closing one |
| 138 | Copy List with Random Pointer | reference-identity hashing (the HashSet version here) |

---

*Sep 25 — solved cold in 13 min, unaided, traced. Code was essentially right; the gap was the
**justification**. "There's some formula" was the wrong instinct: the proof is one sentence, and the
complexity follows from it.*