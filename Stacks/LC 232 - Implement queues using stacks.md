# LC 232 — Implement Queue using Stacks (Easy)

> **Problem:** Implement a FIFO queue using only two stacks. Support `push(x)`, `pop()`, `peek()`,
> `empty()`. You may only use standard stack operations — push to top, peek/pop from top, size,
> isEmpty.
>
> **Follow-up:** can you make each operation **amortised O(1)**?
>
> Constraints: `1 ≤ x ≤ 9`, at most 100 calls, and `pop`/`peek` are only called on a non-empty queue.

**Session record — Sep 24**
| | |
|---|---|
| Outcome | **solved cold, unaided, 7 min** (10-min target) |
| Got right | `ArrayDeque` over `Stack`; lazy transfer only when B is empty |
| Missed | the word **"amortised"** and the argument behind it |
| Missed | wrote `O(2n)`; duplicated the transfer across `pop()` and `peek()` |
| 🔴 Habit | stated the pattern as a problem restatement — **6th consecutive problem** (§8) |

---

## 1. The core idea

A stack is LIFO, a queue is FIFO. They are **exact opposites** — which is the whole reason this
works:

> ### 🔑 The fact
> **Reversing a reversal restores the original order.**

Pour stack A into stack B one element at a time, and B now holds A's contents **backwards** —
i.e. in insertion order. Popping B is popping the oldest element first. That's a queue.

```
push 1,2,3   →   A: [3,2,1]          (top on the left)
                 B: []

transfer     →   A: []
                 B: [1,2,3]           ← 1 is now on top = queue front

pop()        →   returns 1 ✅
```

**The second half of the idea, and the one that makes it efficient:** an element only ever needs
flipping **once**. Once it's in B, it's already in the right order — leave it there. New pushes go
to A and wait their turn.

---

## 2. The naive version and why it's bad

The obvious first instinct: transfer **on every operation**.

```java
public int pop() {
    while (!stackA.isEmpty()) stackB.push(stackA.pop());   // A → B
    int result = stackB.pop();
    while (!stackB.isEmpty()) stackA.push(stackB.pop());   // B → A  ❌ pointless
    return result;
}
```

That second transfer undoes all the work. Every `pop` is O(n), every time, forever — no averaging
saves you.

> **The fix is to be lazy: only transfer when B is empty.** Elements left sitting in B are already
> in the correct order. Moving them back and forth is pure waste.

This is the entire difference between O(n)-always and amortised O(1).

---

## 3. Solution (what I wrote, 7 min)

```java
class MyQueue {

    private Deque<Integer> stackA;   // inbox  — new pushes land here
    private Deque<Integer> stackB;   // outbox — pops/peeks come from here

    public MyQueue() {
        stackA = new ArrayDeque<>();
        stackB = new ArrayDeque<>();
    }

    public void push(int x) {
        stackA.push(x);
    }

    public int pop() {
        if (!stackB.isEmpty()) return stackB.pop();
        while (!stackA.isEmpty()) stackB.push(stackA.pop());
        return stackB.pop();
    }

    public int peek() {
        if (!stackB.isEmpty()) return stackB.peek();
        while (!stackA.isEmpty()) stackB.push(stackA.pop());
        return stackB.peek();
    }

    public boolean empty() {
        return stackA.isEmpty() && stackB.isEmpty();
    }
}
```

### 🚩 The one real flaw — duplicated transfer logic

`pop()` and `peek()` contain **the same five lines**. Two methods that must stay in sync is exactly
where a later edit breaks one and not the other. Extract it:

```java
private void shiftIfNeeded() {
    if (stackB.isEmpty()) {
        while (!stackA.isEmpty()) stackB.push(stackA.pop());
    }
}

public int pop()  { shiftIfNeeded(); return stackB.pop(); }
public int peek() { shiftIfNeeded(); return stackB.peek(); }
```

Both methods become one line each, and the lazy-transfer rule now lives in exactly one place.
**This is the version to write in an interview** — it makes the invariant visible.

### The invariant, stated

> **The queue's front element is the top of B if B is non-empty; otherwise it is the bottom of A.**
> `shiftIfNeeded()` exists to guarantee the first case before any read.

---

## 4. Why `ArrayDeque` and not `Stack`

I chose `ArrayDeque` and should be able to defend it:

| | |
|---|---|
| `java.util.Stack` | extends `Vector` → **synchronised on every call**, legacy since Java 1.0. Also iterates bottom-to-top, which is surprising for a stack. |
| `ArrayDeque` | unsynchronised, array-backed, faster, and the class the JDK docs explicitly recommend for stack use. |
| `LinkedList` | works (implements `Deque`) but allocates a node per element — worse locality. |

> **Interview line:** *"I used `ArrayDeque` rather than `Stack` because `Stack` extends `Vector` and
> is synchronised on every operation — the JDK docs recommend `ArrayDeque` for LIFO use."*

⚠️ On `ArrayDeque` as a stack: `push`/`pop`/`peek` all operate on the **head**. Don't mix in
`add()` / `offer()`, which append to the **tail** — that silently gives you a queue instead.

⚠️ `ArrayDeque` **rejects null elements**. Not an issue for `int`, but it matters if you ever use
`Deque<SomeObject>` and rely on null as a sentinel.

---

## 5. ⭐ Complexity — and the word I missed

| operation | complexity |
|---|---|
| `push` | **O(1)** always |
| `pop` | **amortised O(1)**, worst case O(n) for a single call |
| `peek` | **amortised O(1)**, worst case O(n) for a single call |
| `empty` | **O(1)** |
| space | **O(n)** total across both stacks |

### 🔴 What I said vs what to say

**I said:** *"best case O(1) when B isn't empty, but if I get lots of pushes without pops, it's
O(n) again."*

That's an accurate *description* and a non-answer. It stops one step short of the actual claim.

**Say:** **amortised O(1).**

### The proof — this is what the interviewer is checking

> **Every element is touched exactly three times in its entire lifetime:**
> 1. pushed onto A (once),
> 2. moved A → B (once),
> 3. popped off B (once).
>
> Three constant-time operations per element, **ever**. So `n` queue operations cost O(n) total
> work, and the average per operation is **O(1)** — even though one individual `pop()` can cost
> O(n).
>
> An expensive pop is *paid for in advance* by all the cheap pushes that preceded it. No element is
> ever transferred twice.

> **Say both halves:** *"Amortised O(1) per operation, with a worst case of O(n) on any single
> `pop` — because each element is transferred at most once."*

### 🚩 `O(2n)` is not a thing

Never write constants inside big-O — it's `O(n)`. And the constant isn't even 2: an element lives in
**A or B, never both**, so the two stacks together hold exactly `n` elements.

### Where else "amortised" applies

Same word, same reasoning — it came up in **LC 169** (HashMap ops are amortised O(1), not
guaranteed) and it's the standard answer for **`ArrayList` add** (doubling on resize). If an
operation is usually cheap and occasionally expensive, and the expensive case is *paid for* by the
cheap ones, the word is **amortised**.

---

## 6. Follow-up — do it with ONE stack

Asked in interviews to see whether you understand what the second stack is actually buying you.

### Variant A — expensive `pop`, via recursion

The call stack does the reversing.

```java
public int pop() {
    int top = stack.pop();
    if (stack.isEmpty()) return top;   // bottom of the stack = queue front
    int result = pop();                // recurse into the rest
    stack.push(top);                   // ⚠️ rebuild on the UNWIND, after the call returns
    return result;
}
```

**The ordering detail:** `stack.push(top)` goes **after** the recursive call, not before. The
rebuild happens as the recursion unwinds.

> ⚠️ **Know the objection before it's made:** the call stack *is* a second stack. This is
> O(n) explicit + O(n) implicit stack space. It's not really a one-stack solution, and saying so
> yourself is stronger than being corrected.

### Variant B — expensive `push`, cheap everything else

Mirror the tradeoff: keep the single stack **permanently in queue order** by paying on insert.

```java
public void push(int x) {
    Deque<Integer> tmp = new ArrayDeque<>();
    while (!stack.isEmpty()) tmp.push(stack.pop());
    stack.push(x);
    while (!tmp.isEmpty()) stack.push(tmp.pop());
}
// pop() and peek() are now plain O(1) stack operations
```

O(n) push, O(1) pop/peek. (Also arguably uses a second stack, just a temporary one.)

### Verdict

| version | push | pop | amortised O(1)? |
|---|---|---|---|
| Two stacks, lazy transfer ⭐ | O(1) | O(n) worst | ✅ **yes** |
| One stack, recursive pop | O(1) | O(n) **every time** | ❌ |
| One stack, expensive push | O(n) **every time** | O(1) | ❌ |

**The two-stack version is strictly better**, and that's the point of the problem: the second stack
is what lets the O(n) cost be *amortised away* instead of paid repeatedly.

---

## 7. Edge cases checklist

| case | handled? |
|---|---|
| `pop` on empty queue | not required — constraints guarantee non-empty. Would NPE/NoSuchElement; mention you'd throw or return a sentinel |
| interleaved push/pop (`push 1, pop, push 2, pop`) | ✅ each transfer happens only when B is empty |
| pushes arriving **while B still has elements** | ✅ they queue up in A and transfer later — this is exactly why the lazy check is `if (stackB.isEmpty())` and not `if (stackA.isEmpty())` |
| `empty()` must check **both** stacks | ✅ — checking only one is the classic bug here |
| all pushes, then all pops | ✅ one O(n) transfer, then n O(1) pops |

> 🚩 **The classic bug in this problem:** transferring when A is non-empty rather than when B is
> **empty**. That destroys ordering — you'd be interleaving newer elements on top of older ones in
> B. The condition is about **B being empty**, always.

---

## 8. 🔴 Habit — stating the pattern as a problem restatement

**Sixth consecutive problem.** What I said this time:

> *"if we use 2 stacks and transfer between them, we can mimic the queue"*

That is the **problem title, restated**. Delete it and I've lost nothing.

### The test (apply it before speaking)

> **Cross out every word that came from the problem statement. What survives is the pattern.**
> If nothing survives, keep thinking.

Run it on mine: *"use 2 stacks"* — from the title. *"mimic the queue"* — from the title. Nothing
left.

### The record so far

| problem | ❌ what I said (the steps) | ✅ the fact |
|---|---|---|
| 169 | "increment if same, decrement if not, reset at 0" | more than n/2 copies can't be fully cancelled by pairing |
| 1552 | "find where monotonicity turns" | if a gap is achievable, every smaller gap is too |
| 138 | "make a new node next to the original, then separate" | the clone of X **is** X.next |
| 25 | "reverse the group and add it to the list" | reversing a segment turns its head into its tail |
| 232 | "use 2 stacks and transfer between them" | reversing a reversal restores the original order |

Every ✅ entry is **one observation you could forget**, and forgetting it costs you the solution.
Every ❌ entry is a description of finger movements.

> **A pattern line = the non-obvious fact, with the problem statement stripped out.**

---

## 9. One-line pattern

> **Reversing a reversal restores the original order — and an element only needs flipping once.**

Both halves earn their place: the first gives you correctness (why two stacks make a queue), the
second gives you the complexity (why it's amortised O(1) rather than O(n) every time).

---

## 10. Takeaways

1. **"Usually fast, sometimes slow" has a name: amortised.** Reach for the word, then justify it by
   counting **total work across all operations**, not per-call.
2. **The amortisation argument here is "each element is transferred at most once."** Three touches
   per element, ever.
3. **Be lazy: only do the expensive transfer when you must.** Transferring back is what turns
   amortised O(1) into O(n)-always.
4. **Never write `O(2n)`.** Drop constants. (And here the constant was wrong anyway — an element is
   in A *or* B.)
5. **Extract duplicated logic between two methods that must stay in sync.** `shiftIfNeeded()`.
6. **`ArrayDeque` over `Stack`** — `Stack` extends `Vector` and is synchronised. Have the one-liner
   ready.
7. **Name the objection to your own follow-up answer** — recursion uses the call stack, so
   "one stack" is a fiction.
8. **🔴 Strip the problem statement out of the pattern line.** Six problems running.

---

## 11. Related problems

| # | Problem | Connection |
|---|---|---|
| 225 | Implement Stack using Queues | ⭐ the mirror image — do it next; note the tradeoff lands differently |
| 155 | Min Stack | auxiliary stack carrying extra state alongside the main one |
| 622 | Design Circular Queue | queue mechanics from the other direction |
| 946 | Validate Stack Sequences | reasoning about achievable push/pop orders |
| 20 | Valid Parentheses | the base "stack = reversal" instinct |
| 1381 | Design a Stack With Increment Operation | amortised reasoning on a designed data structure |

**Next: LC 225 (Implement Stack using Queues)** — the mirror. Worth doing while this is warm,
because the amortisation argument does **not** transfer, and noticing that is the lesson.

---

*Sep 24 — solved cold in 7 min, unaided. Code was right; the complexity vocabulary was the gap.
The word to have reached for was **amortised**, and the proof is that **each element is transferred
at most once.***