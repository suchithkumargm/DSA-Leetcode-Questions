# LC 225 — Implement Stack using Queues

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE
**Pattern:** Order inversion by rotation — and the problem that proves LC 232's amortisation doesn't generalise

---

## 1. Problem

Implement a LIFO stack using only queues. Support `push(x)`, `pop()`, `top()`, `empty()`.

> **You may use only standard queue operations: push to back, peek/pop from front, size, is empty.**

Constraints: at most 100 calls; `pop`/`top` are only called on a non-empty stack. You may use one or
two queues.

That constraint line is not boilerplate — §3 shows it rules out the obvious fix for `top()`.

---

## 2. Why this is harder than LC 232

The mirror problem (queue from stacks) has a clean answer: **reversing a reversal restores the
original order.** Pour stack A into stack B and B holds A's contents backwards, which is exactly what
you wanted. Each element is flipped once and then sits in its final position.

**A queue cannot reverse anything.** Pouring queue A into queue B preserves the order — `offer` adds
to the back and `poll` takes from the front, so the oldest stays oldest. You can only ever **rotate**.

> ### 🔑 The fact
> **A stack's top is the newest element, and a queue's front is the oldest. The only way to reconcile
> them is to move every *other* element out of the way — and that work isn't reusable, because the
> next push has to do it again.**

---

## 3. ⚠️ `top()` and the constraint

The natural implementation of `top()` is "read the back of the queue". **That operation does not
exist** — the constraint allows peeking only from the **front**.

This bites specifically if you're holding elements in insertion order, where the front is the stack's
**bottom**:

```java
return queue1.peek();     // 🚩 ArrayDeque.peek() is peekFirst — the OLDEST element
```

And the fix you'd reach for is illegal:

```java
return queue1.peekLast(); // 🚩 not a queue operation
```

> ### 🔑 Let the type enforce the constraint
> Declare the fields as **`Queue<Integer>`**, not `Deque<Integer>`:
>
> ```java
> Queue<Integer> queue = new ArrayDeque<>();
> ```
>
> `Queue` exposes only `offer`/`poll`/`peek`/`size`/`isEmpty`. `peekLast` and `addFirst` won't
> compile, so you cannot accidentally cheat. `ArrayDeque` still provides the implementation.
>
> Same "program to the interface" instinct as LC 232 — except here the interface *is* the problem
> statement.

---

## 4. ⭐ The headline result: no amortised O(1) exists

LC 232's answer is **amortised O(1)**, justified by: *every element is pushed once, transferred once,
and popped once — three touches over its whole lifetime.* An expensive pop is paid for by the cheap
pushes before it.

**That argument does not survive here, and it's worth knowing exactly why.**

In LC 232, moving an element puts it in its **final position** — once it's in stack B it gets popped
directly, never touched again. The work is *spent*.

In LC 225, moving elements during a push puts them nowhere permanent. The next push has to move them
all again, because a new element must get in front of them.

Count it on the rotation design:

```
push #1:  0 rotations
push #2:  1
push #3:  2
...
push #n:  n−1
          ────────────
total:    n(n−1)/2  =  O(n²)
```

**n pushes cost O(n²).** So even averaged over the whole sequence, push is O(n) — not O(1).

The drain-on-pop design is no better: n pushes are cheap, but then n pops drain n, n−1, n−2 … = the
same O(n²). And `top()` is worse still, because it drains without consuming, so 100 consecutive
`top()` calls on 100 elements is 10,000 transfers.

> **Every design for LC 225 pays O(n) somewhere, every time. The only choice is which operation
> absorbs it.** That asymmetry with LC 232 is the entire reason the two are asked as a pair.

### Which operation should absorb it?

| design | push | pop | top |
|---|---|---|---|
| drain on **pop** | O(1) | O(n) | O(n) |
| drain on **push** ⭐ | O(n) | **O(1)** | **O(1)** |

**Make `push` expensive.** On a stack, `top()` is the hot read, and making *both* reads O(n) to save
on writes is the wrong trade. Say this out loud — recognising that the choice exists is most of what's
being tested.

---

## 5. Solution A — one queue, rotate on push ⭐

```java
class MyStack {
    private final Queue<Integer> queue = new ArrayDeque<>();

    public void push(int x) {
        queue.offer(x);
        for (int i = queue.size() - 1; i > 0; i--) {   // rotate everything else behind x
            queue.offer(queue.poll());
        }
    }

    public int pop()      { return queue.poll(); }
    public int top()      { return queue.peek(); }
    public boolean empty(){ return queue.isEmpty(); }
}
```

**O(n) push · O(1) pop, top, empty · O(n) space.**

### The invariant

> **The front of the queue is always the top of the stack.**

`push` restores it by moving every element that was ahead of `x` to behind it:

```
queue = [1]
push(2):   offer      →  [1, 2]
           rotate 1×  →  [2, 1]          1 goes front → back
push(3):   offer      →  [2, 1, 3]
           rotate 2×  →  [3, 2, 1]
```

**Rotation count is `size − 1`**, read *after* the offer. Off by one and the wrong element is at the
front. (The size doesn't change during rotation — a `poll` and an `offer` cancel — so hoisting it or
re-reading it are equivalent; hoisting just says so.)

Three lines of logic, **no branching anywhere**, and `pop`/`top`/`empty` are one-liners because the
invariant has already done the work.

---

## 6. Solution B — two queues, drain on push

Correct, and a legitimate answer. The key is **which** queue receives the new element.

```java
class MyStack {
    private final Queue<Integer> queue1 = new ArrayDeque<>();
    private final Queue<Integer> queue2 = new ArrayDeque<>();

    private Queue<Integer> active()   { return queue1.isEmpty() ? queue2 : queue1; }  // holds the data
    private Queue<Integer> inactive() { return queue1.isEmpty() ? queue1 : queue2; }  // the empty one

    public void push(int x) {
        Queue<Integer> to = inactive(), from = active();
        to.offer(x);                                        // x lands at the FRONT
        while (!from.isEmpty()) to.offer(from.poll());      // everything else queues behind it
    }

    public int pop()       { return active().poll(); }
    public int top()       { return active().peek(); }
    public boolean empty() { return queue1.isEmpty() && queue2.isEmpty(); }
}
```

**The invariant:** one queue holds everything in top-to-bottom order; the other is empty.

**The load-bearing detail:** `x` goes into the **empty** queue *first*, and the old contents are
appended **behind** it. Reverse that order and nothing is reversed at all (§7).

### Why A is preferable

The second queue is **redundant state** — the same lesson as LC 641, where `rear` was redundant
alongside `front` and `size`. Here it forces a "which queue holds the data?" question that must be
answered correctly in `push`, `pop`, `top` and `empty`. The `active()`/`inactive()` helpers collapse
four branch sites into two, which is the right move if you keep this design.

The rotation version doesn't have the question, because there's only one queue.

> **Same complexity, fewer places to be wrong.**

---

## 7. Common pitfalls

### 🚩 Draining into the other queue and expecting a reversal

```java
queue1.offer(x);                                    // new element into the queue that has data
while (!queue1.isEmpty()) queue2.offer(queue1.poll());   // 🚩 appends to the BACK of queue2
```

```
push(1):  q1=[1]  → drain →  q2=[1]
push(2):  q1=[2]  → drain →  q2=[1, 2]
                                ↑ front
top() → 1          should be 2  ✗
```

The queue has moved from one variable to another and the order is untouched — the oldest element is
still at the front. **A transfer between queues preserves order; only the insertion point changes
anything.**

The fix is which queue receives `x`: it must go into the **empty** one, so the old contents land
behind it.

### 🚩 Inverted loop condition

```java
if (!queue1.isEmpty()) {
    while (queue1.isEmpty()) { … }     // 🚩 just established it isn't
```

The body never executes and the method returns its initialiser (typically `0`). Silent and total —
every `pop()` returns the same wrong value.

### 🚩 `peek()` for `top()` when the front is the bottom

`ArrayDeque.peek()` is `peekFirst`. If elements sit in insertion order, that's the stack's bottom.
And `peekLast()` is not a legal queue operation — §3.

### 🚩 A boolean flag switching a helper between two behaviours

```java
private int drain(boolean isPopOperation) { … }
drain(true);   // ← tells a reader nothing
```

The classic flag-argument smell. When the two behaviours differ only in whether the last element is
kept, express it as composition instead:

```java
public int top() {
    int x = pop();
    push(x);        // put it back — it's the newest, so it returns to the front
    return x;
}
```

### 🚩 A dead branch after the design changes

A "which queue is non-empty" check becomes unreachable the moment `push` unconditionally targets one
of them. Re-check your branches after moving work between methods.

### 🚩 Dropping a method during a refactor

`empty()` is easy to lose when restructuring the other three. The compile error names the missing
symbol, but it's worth re-reading the required interface after any rewrite.

---

## 8. Dry runs

### Solution A — rotation

```
push(1):  offer      →  [1]              rotate 0× (size−1 = 0)
push(2):  offer      →  [1, 2]           rotate 1× →  [2, 1]
push(3):  offer      →  [2, 1, 3]        rotate 2× →  [3, 2, 1]

top()  →  peek  =  3  ✓
pop()  →  poll  =  3  ✓        queue = [2, 1]
top()  →  peek  =  2  ✓
push(4): offer →  [2, 1, 4]              rotate 2× →  [4, 2, 1]
pop()  →  4  ✓
```

The front is the top at every single point — that's the whole design.

### Solution B — two queues, alternating

```
push(1):  q1 empty → q1=[1];  drain q2 (empty)        q1=[1]      q2=[]
push(2):  q1 has data → q2=[2];  drain q1 → q2=[2,1]  q1=[]       q2=[2,1]
push(3):  q1 empty → q1=[3];  drain q2 → q1=[3,2,1]   q1=[3,2,1]  q2=[]

top()  →  active() = q1,  peek  =  3  ✓
pop()  →  q1.poll() = 3  ✓                            q1=[2,1]    q2=[]
push(4): q1 has data → q2=[4]; drain q1 → q2=[4,2,1]  q1=[]       q2=[4,2,1]
pop()  →  active() = q2,  poll  =  4  ✓
```

Note the queues **alternate** roles with each push, and `pop` leaves the roles unchanged — which is
why `active()` has to be computed, not remembered.

---

## 9. Edge cases

| case | behaviour |
|---|---|
| `pop`/`top` on an empty stack | not required by constraints; would return `null` from `poll`/`peek` (and NPE on unboxing). Mention you'd throw |
| single push then pop | A: 0 rotations, poll returns it. B: lands in `queue1`, nothing to drain |
| push after draining the stack empty | B: both queues empty, `inactive()` returns `queue1`, drain is a no-op ✓ |
| interleaved push/pop/top | ✅ both designs — the invariant is restored by every push |
| all pushes, then all pops | A: O(n²) pushing, then n O(1) pops |
| 100 consecutive `top()` calls | A and B: O(1) each. The drain-on-pop design: O(n) each — 10,000 transfers |

---

## 10. Interview Q&A

**Q: Why can't you use the LC 232 trick?**
Because transferring between queues preserves order — a queue can't reverse. Two stacks reverse
twice and land you back where you started, which is exactly what you want; two queues don't.

**Q: Complexity?**
O(n) push, O(1) pop/top/empty. Space O(n).

**Q: Is it amortised O(1) like LC 232?**
**No.** In 232 a transferred element reaches its final position and is never touched again — three
touches per element, ever. Here, work done during one push is undone by the next, so n pushes cost
O(n²). Every LC 225 design pays O(n) somewhere, every time.

**Q: Could you make push O(1) instead?**
Yes — drain on pop rather than on push. But then `pop` *and* `top` are both O(n), and `top` is
usually the hot operation on a stack. Wrong trade.

**Q: One queue or two?**
One. The second queue is redundant state that forces a "which queue holds the data?" branch in four
places. Same complexity, more surface area.

**Q: Why declare the field as `Queue` rather than `Deque`?**
So the compiler enforces the problem's restriction — `peekLast` and `addFirst` won't compile.

**Q: How many rotations per push?**
`size − 1`, measured after the new element is offered.

**Q: Could a single queue do it with O(1) push and O(n) pop?**
Yes — push plainly, then on pop rotate `size − 1` elements to bring the newest to the front, poll it,
and leave the rest rotated. Same no-amortisation story, just relocated.

---

## 11. Pattern

> **A queue can only rotate, never reverse — so making its front behave like a stack's top means
> moving every other element out of the way, on every single push.**

The transferable half: **work is only "amortised away" if it moves something to its final position.**
LC 232's transfer does; LC 225's rotation doesn't, because the next push invalidates it. Before
claiming amortised O(1) anywhere, ask whether the expensive step is *progress* or just *rearrangement*.

---

## 12. Takeaways

1. **Transferring between queues preserves order.** Only the choice of *which* queue receives the new
   element reverses anything.
2. **The new element goes into the empty queue first**, with the old contents appended behind it.
3. **Declare it `Queue<Integer>`, not `Deque`** — the type then enforces the problem's restriction.
4. **`peek()` is the front, which is the stack's bottom** in insertion order. `peekLast()` is illegal.
5. **There is no amortised O(1) design.** Every version is O(n) somewhere, every time — the contrast
   with LC 232 is the point of the problem.
6. **Work amortises only when it's progress.** 232's transfer is final; 225's rotation is redone next
   push.
7. **Make `push` the expensive one.** `top()` is the hot read on a stack.
8. **Rotation count is `size − 1`**, read after the offer.
9. **One queue, not two.** The second is redundant state — LC 641's lesson in a different costume.
10. **A boolean flag that switches a method's behaviour** is usually two methods, or composition
    (`top()` = `pop()` then `push()`).
11. **After a `while` loop's condition, re-read it.** `if (!empty) while (empty)` never runs and fails
    silently.