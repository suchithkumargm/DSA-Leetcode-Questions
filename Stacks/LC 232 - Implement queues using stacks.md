# LC 232 — Implement Queue using Stacks

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE
**Pattern:** Two-stack inbox/outbox · amortised analysis

---

## 1. Problem

Implement a FIFO queue using only two stacks. Support `push(x)`, `pop()`, `peek()`, `empty()`. You
may only use standard stack operations — push to top, peek/pop from top, size, isEmpty.

**Follow-up:** can you make each operation **amortised O(1)**?

Constraints: at most 100 calls; `pop`/`peek` are only called on a non-empty queue.

---

## 2. The core idea

A stack is LIFO, a queue is FIFO — **exact opposites**, which is the whole reason this works:

> ### 🔑 The fact
> **Reversing a reversal restores the original order.**

Pour stack A into stack B one element at a time and B holds A's contents **backwards** — i.e. in
insertion order. Popping B pops the oldest element first. That's a queue.

```
push 1,2,3   →   A: [3,2,1]          (top on the left)
                 B: []

transfer     →   A: []
                 B: [1,2,3]           ← 1 is now on top = queue front

pop()        →   returns 1 ✅
```

**The second half of the idea, and the one that makes it efficient:** an element only needs flipping
**once**. Once it's in B it's already in the right order — leave it there. New pushes go to A and
wait their turn.

---

## 3. The naive version and why it's bad

The obvious first instinct: transfer **on every operation**.

```java
public int pop() {
    while (!stackA.isEmpty()) stackB.push(stackA.pop());   // A → B
    int result = stackB.pop();
    while (!stackB.isEmpty()) stackA.push(stackB.pop());   // B → A  🚩 pointless
    return result;
}
```

The second transfer undoes all the work. Every `pop` is O(n), every time, forever — no averaging
saves you.

> **The fix is laziness: only transfer when B is empty.** Elements sitting in B are already in the
> correct order; moving them back and forth is pure waste.

This is the entire difference between O(n)-always and amortised O(1).

---

## 4. Solution

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

    private void shiftIfNeeded() {
        if (stackB.isEmpty()) {
            while (!stackA.isEmpty()) stackB.push(stackA.pop());
        }
    }

    public int pop()  { shiftIfNeeded(); return stackB.pop(); }
    public int peek() { shiftIfNeeded(); return stackB.peek(); }

    public boolean empty() {
        return stackA.isEmpty() && stackB.isEmpty();
    }
}
```

### Why extract `shiftIfNeeded()`

Written inline, `pop()` and `peek()` contain **the same five lines**. Two methods that must stay in
sync is exactly where a later edit breaks one and not the other. Extracting it makes both one line
and puts the lazy-transfer rule in exactly one place.

### The invariant

> **The queue's front element is the top of B if B is non-empty; otherwise it is the bottom of A.**
> `shiftIfNeeded()` exists to guarantee the first case before any read.

---

## 5. Why `ArrayDeque` and not `Stack`

| | |
|---|---|
| `java.util.Stack` | extends `Vector` → **synchronised on every call**, legacy since Java 1.0. Also iterates bottom-to-top, which is surprising for a stack. |
| `ArrayDeque` | unsynchronised, array-backed, faster. The class the JDK docs explicitly recommend for stack use. |
| `LinkedList` | works (implements `Deque`) but allocates a node per element — worse locality. |

> **Interview line:** *"I used `ArrayDeque` rather than `Stack` because `Stack` extends `Vector` and
> is synchronised on every operation — the JDK docs recommend `ArrayDeque` for LIFO use."*

⚠️ On `ArrayDeque` as a stack: `push`/`pop`/`peek` all operate on the **head**. Don't mix in `add()`
or `offer()`, which append to the **tail** — that silently gives you a queue instead.

⚠️ `ArrayDeque` **rejects null elements**. Irrelevant for `int`, but it matters with
`Deque<SomeObject>` if you rely on null as a sentinel.

---

## 6. ⭐ Complexity — and the word that matters

| operation | complexity |
|---|---|
| `push` | **O(1)** always |
| `pop` | **amortised O(1)**, worst case O(n) for a single call |
| `peek` | **amortised O(1)**, worst case O(n) for a single call |
| `empty` | **O(1)** |
| space | **O(n)** total across both stacks |

### ⚠️ "Sometimes fast, sometimes slow" is a non-answer

Describing the behaviour — *"O(1) when B isn't empty, O(n) when it's not"* — is accurate and stops
one step short of the actual claim. The claim is **amortised O(1)**.

### The proof — this is what the interviewer is checking

> **Every element is touched exactly three times in its entire lifetime:**
> 1. pushed onto A (once),
> 2. moved A → B (once),
> 3. popped off B (once).
>
> Three constant-time operations per element, **ever**. So `n` queue operations cost O(n) total work,
> and the average per operation is **O(1)** — even though one individual `pop()` can cost O(n).
>
> An expensive pop is *paid for in advance* by all the cheap pushes that preceded it. **No element
> is ever transferred twice.**

> **Say both halves:** *"Amortised O(1) per operation, with a worst case of O(n) on any single
> `pop` — because each element is transferred at most once."*

### 🚩 `O(2n)` is not a thing

Never write constants inside big-O — it's `O(n)`. And the constant isn't even 2: an element lives in
**A or B, never both**, so the two stacks together hold exactly `n` elements.

### Where else "amortised" applies

Same word, same reasoning: HashMap operations (LC 169), `ArrayList` add (doubling on resize). **If
an operation is usually cheap and occasionally expensive, and the expensive case is *paid for* by
the cheap ones, the word is amortised.**

---

## 7. Follow-up — do it with ONE stack

Asked to see whether you understand what the second stack is buying you.

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

**The ordering detail:** `stack.push(top)` goes **after** the recursive call, not before. The rebuild
happens as the recursion unwinds.

> ⚠️ **Name the objection before it's made:** the call stack *is* a second stack. This is O(n)
> explicit + O(n) implicit stack space, so "one stack" is a fiction. Saying so yourself is stronger
> than being corrected.

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

## 8. Common pitfalls

**Transferring when A is non-empty rather than when B is empty.** This is the classic bug. It
destroys ordering — you'd interleave newer elements on top of older ones in B. **The condition is
about B being empty, always.**

**Transferring back after popping** (§3). Turns amortised O(1) into O(n)-always.

**`empty()` checking only one stack.** Both must be empty.

**Duplicating the transfer in `pop()` and `peek()`** instead of extracting a helper.

**Using `Stack` instead of `ArrayDeque`**, or mixing `push`/`add` on an `ArrayDeque`.

---

## 9. Edge cases

| case | handled? |
|---|---|
| `pop` on empty queue | not required by constraints. Would throw `NoSuchElementException`; mention you'd throw or return a sentinel |
| interleaved push/pop (`push 1, pop, push 2, pop`) | ✅ transfer happens only when B is empty |
| pushes arriving **while B still has elements** | ✅ they queue up in A and transfer later — exactly why the check is `if (stackB.isEmpty())` |
| all pushes, then all pops | ✅ one O(n) transfer, then n O(1) pops |

---

## 10. Interview Q&A

**Q: What's the complexity of `pop`?**
Amortised O(1); worst case O(n) on a single call. Each element is pushed, transferred, and popped
exactly once — three operations per element over its whole lifetime.

**Q: Why don't you transfer back after popping?**
Elements left in B are already in queue order. Transferring back would mean re-flipping them later,
which is what makes the naive version O(n) every time rather than amortised O(1).

**Q: What if `push` also needed to be O(1) worst case, not just amortised?**
You can't have both with two stacks — one operation has to absorb the transfer. A genuinely O(1)
worst-case queue needs a different structure (e.g. a linked list with head and tail pointers, or an
incrementally-rebuilding "real-time queue").

**Q: Do it with one stack.**
Recursion (§7), and name the objection that the call stack is a second stack.

**Q: Why `ArrayDeque` over `Stack`?**
`Stack` extends `Vector` and is synchronised on every call; the JDK docs recommend `ArrayDeque`.

---

## 11. Pattern

> **Reversing a reversal restores the original order — and an element only needs flipping once.**

Both halves earn their place: the first gives correctness (why two stacks make a queue), the second
gives the complexity (why it's amortised O(1) rather than O(n) every time).

---

## 12. Takeaways

1. **"Usually fast, sometimes slow" has a name: amortised.** Reach for the word, then justify it by
   counting **total work across all operations**, not per-call.
2. **The argument here is "each element is transferred at most once."** Three touches per element,
   ever.
3. **Be lazy: only do the expensive transfer when you must.** Transferring back is what breaks the
   amortisation.
4. **Never write `O(2n)`.** Drop constants — and here the constant was wrong anyway.
5. **Extract duplicated logic between two methods that must stay in sync.**
6. **`ArrayDeque` over `Stack`** — have the one-liner ready.
7. **Name the objection to your own follow-up answer** — recursion uses the call stack.

---

## 13. Related problems

| # | Problem | Connection |
|---|---|---|
| **225** | Implement Stack using Queues | ⭐ the mirror image — note the amortisation argument does **not** transfer |
| 155 | Min Stack | auxiliary stack carrying extra state alongside the main one |
| 622 | Design Circular Queue | queue mechanics from the other direction |
| 946 | Validate Stack Sequences | reasoning about achievable push/pop orders |
| 20 | Valid Parentheses | the base "stack = reversal" instinct |
| 1381 | Design a Stack With Increment Operation | amortised reasoning on a designed data structure |