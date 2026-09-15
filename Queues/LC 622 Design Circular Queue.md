# LC 622 — Design Circular Queue

**Difficulty:** Medium
**Topic:** Design, Array, Queue
**Link:** https://leetcode.com/problems/design-circular-queue/
**Sheets:** NeetCode 150 (—), Striver SDE (Stack & Queue), Apna College (Queue)
**Status:** Solved — beats 100%
**Date:** Sep 2026

---

## 1. Problem

Design a fixed-capacity FIFO queue backed by an array. Operations:

| Method | Contract |
|---|---|
| `MyCircularQueue(k)` | capacity `k` |
| `enQueue(value)` | insert at rear, `false` if full |
| `deQueue()` | remove from front, `false` if empty |
| `Front()` | front element, `-1` if empty |
| `Rear()` | rear element, `-1` if empty |
| `isEmpty()` / `isFull()` | booleans |

**Return-contract trap:** mutators return `boolean`, accessors return `-1`. Nothing throws. Most WAs on this problem are contract violations, not logic bugs.

**Constraints:** `1 <= k <= 1000`, `0 <= value <= 1000`, up to `3000` calls. Note `k = 1` is legal — it's the boundary that breaks naive designs.

---

## 2. Why this problem exists

A queue needs O(1) insert at one end and O(1) remove at the other. An array gives O(1) indexed access, so this should be trivial — and it is, *until the indices run off the end of the array*. The whole problem is the wrap-around, and the whole difficulty is one question:

> **When `front` and `rear` collide, is the queue full or empty?**

Everything below is a different answer to that one question.

---

## 3. Approach 1 — Naive linear queue (BROKEN)

Keep `front` and `rear` as plain counters. `enQueue` does `rear++`, `deQueue` does `front++`. Reject when `rear == capacity`.

```java
// DO NOT SUBMIT — illustrative only
public boolean enQueue(int value) {
    if (rear == capacity) return false;
    queue[rear++] = value;
    return true;
}
public boolean deQueue() {
    if (front == rear) return false;
    front++;
    return true;
}
```

### Why it fails — the slot leak

`k = 3`:

```
enQueue(1), enQueue(2), enQueue(3)
  queue = [1, 2, 3]   front = 0   rear = 3

deQueue() x3
  queue = [1, 2, 3]   front = 3   rear = 3     <- logically EMPTY

enQueue(4)
  rear == capacity (3 == 3) -> return false    <- REJECTED
```

The queue is empty but refuses insertions. Slots `0..2` are permanently dead. After `k` enqueues the structure is bricked forever, regardless of how many dequeues happened.

- **Time:** O(1) per op
- **Space:** O(k)
- **Verdict:** wrong. This is the motivation for "circular".

---

## 4. Approach 2 — Shift-on-dequeue (works, too slow)

Patch the leak by keeping `front` pinned at index 0 — on every `deQueue`, shift everything left one position.

```java
public boolean deQueue() {
    if (size == 0) return false;
    for (int i = 1; i < size; i++) queue[i - 1] = queue[i];
    size--;
    return true;
}
```

Correct, no leak, no wrap-around logic needed. But:

- **enQueue:** O(1)
- **deQueue:** O(k)  ← the killer
- **Space:** O(k)

With 3000 calls and `k = 1000` this is ~3·10⁶ moves. It passes LeetCode, but it defeats the point of a queue — dequeue is supposed to be the cheap operation. **If you offer this in an interview, offer it as "the obvious fix, but it turns an O(1) op into O(n) — let me do better."**

The insight it hands you: the elements don't need to move. Only the *window* into the array needs to move. So let the window wrap instead.

---

## 5. Approach 3 — Circular, sacrifice one slot

Wrap both pointers with modulo: `rear = (rear + 1) % capacity`. Now indices recycle and the leak is gone. But immediately the ambiguity appears.

Let `rear` point at the **next write position**. Start `front = rear = 0`.

```
EMPTY state (k=3):   front = 0, rear = 0        -> front == rear
after 3 enqueues:    front = 0, rear = (2+1)%3 = 0  -> front == rear
```

**Two different states, identical variables.** `front == rear` cannot distinguish full from empty. This is the core algorithmic obstacle of the problem.

**Fix:** refuse to ever fill the last slot. Then `front == rear` means empty, and `(rear + 1) % capacity == front` means full.

```java
boolean isEmpty() { return front == rear; }
boolean isFull()  { return (front + 1) % capacity == rear; } // rear = next-write flavour
```

- **Time:** O(1) all ops
- **Space:** O(k + 1) — must allocate `k + 1` to actually store `k` elements
- **Cost:** one wasted slot. Also, `k = 1` needs an array of size 2.
- **Verdict:** classic textbook answer, valid, slightly wasteful.

---

## 6. Approach 4 — Circular with a `size` counter

Drop `rear` entirely. Store `front` and `size` (element count). Derive rear.

```java
private int front = 0, size = 0;

boolean isEmpty() { return size == 0; }
boolean isFull()  { return size == capacity; }

boolean enQueue(int v) {
    if (isFull()) return false;
    queue[(front + size) % capacity] = v;   // next write slot, derived
    size++;
    return true;
}
boolean deQueue() {
    if (isEmpty()) return false;
    front = (front + 1) % capacity;
    size--;
    return true;
}
int Rear() { return isEmpty() ? -1 : queue[(front + size - 1) % capacity]; }
```

**Key identity:** `nextWrite = (front + size) % capacity`. You never store `rear`, so it can never desync from `front`.

- **Time:** O(1) all ops
- **Space:** O(k) — full capacity used
- **Bonus:** `count()` is free. `size` is already the answer.
- **Trap:** `Rear()` is `(front + size - 1) % capacity`. Guard `isEmpty()` **before** the `- 1`, or `size = 0` gives you index `-1`.
- **Verdict:** the most robust design. **Use this one for LC 641 (deque).**

---

## 7. Approach 5 — Circular with `-1` sentinel (SUBMITTED)

Keep both `front` and `rear` (pointing at the **last written element**, not next-write). Encode empty as the impossible state `front == rear == -1`.

Because `-1` is unreachable during normal operation, it's an unambiguous empty marker — so `front == rear >= 0` safely means "exactly one element", and `(rear + 1) % capacity == front` safely means full.

### Final code

```java
class MyCircularQueue {

    private final int[] queue;
    private final int capacity;
    private int front = -1, rear = -1;

    public MyCircularQueue(int k) {
        this.queue = new int[k];
        this.capacity = k;
    }

    public boolean enQueue(int value) {
        if (isFull()) return false;
        if (isEmpty()) front = 0;
        rear = (rear + 1) % capacity;   // -1 -> 0 on first insert, free
        queue[rear] = value;
        return true;
    }

    public boolean deQueue() {
        if (isEmpty()) return false;
        if (front == rear) {            // last element leaving -> reset sentinel
            front = rear = -1;
        } else {
            front = (front + 1) % capacity;
        }
        return true;
    }

    public int Front() { return isEmpty() ? -1 : queue[front]; }
    public int Rear()  { return isEmpty() ? -1 : queue[rear];  }

    public boolean isEmpty() { return front == -1; }
    public boolean isFull()  { return (rear + 1) % capacity == front; }
}
```

- **Time:** O(1) all six methods — a constant number of arithmetic ops, no loops.
- **Space:** **O(k)** for the storage, which is the data structure the problem asks for; **O(1) auxiliary** beyond it (3 scalar fields). Full capacity used, no sacrificed slot.

> **How to state complexity out loud:** "O(1) time for every operation. O(k) space, which is the structure itself and is required by the problem — O(1) auxiliary." Saying bare "O(1) space" invites pushback.

### Two things that make it work

**(a) `rear = (rear + 1) % capacity` handles the first insert for free.**
`rear` starts at `-1`, so `(-1 + 1) % capacity == 0`. No special-case write path. That's why `enQueue` has no third branch.

**(b) `isFull()` needs no empty-check.**
When empty, `rear = -1`, so `(-1 + 1) % capacity == 0`, and `front == -1`. `0 == -1` is false. The empty state can never be misread as full. Proof once, then delete the guard.

---

## 8. Dry runs

### Dry run A — fill, reject, wrap (`k = 3`)

Legend: `·` = stale/unreachable slot.

```
init            queue = [·, ·, ·]   front = -1  rear = -1

enQueue(1)
  isFull? (-1+1)%3 = 0 == -1 ? no
  isEmpty? yes -> front = 0
  rear = (-1+1)%3 = 0 ; queue[0] = 1
                queue = [1, ·, ·]   front =  0  rear =  0   -> true

enQueue(2)
  isFull? (0+1)%3 = 1 == 0 ? no
  rear = 1 ; queue[1] = 2
                queue = [1, 2, ·]   front =  0  rear =  1   -> true

enQueue(3)
  isFull? (1+1)%3 = 2 == 0 ? no
  rear = 2 ; queue[2] = 3
                queue = [1, 2, 3]   front =  0  rear =  2   -> true

enQueue(4)
  isFull? (2+1)%3 = 0 == front 0 ? YES
                                                            -> false

Rear()          isEmpty? no -> queue[2] = 3                 -> 3
isFull()                                                    -> true

deQueue()
  isEmpty? no ; front(0) == rear(2)? no
  front = (0+1)%3 = 1
                queue = [·, 2, 3]   front =  1  rear =  2   -> true

enQueue(4)      <- THE WRAP
  isFull? (2+1)%3 = 0 == front 1 ? no
  isEmpty? no
  rear = (2+1)%3 = 0 ; queue[0] = 4
                queue = [4, 2, 3]   front =  1  rear =  0   -> true

Front()         queue[1] = 2                                -> 2
Rear()          queue[0] = 4                                -> 4
```

**Read the final state carefully:** `front = 1`, `rear = 0`. `rear < front` and that is perfectly valid — the logical queue is `2 -> 3 -> 4`, physically stored as `[4, 2, 3]`. The array order is not the queue order. This is the mental picture to hold.

### Dry run B — drain to empty, sentinel reset (`k = 2`)

```
init            queue = [·, ·]      front = -1  rear = -1
enQueue(5)      front = 0 ; rear = 0
                queue = [5, ·]      front =  0  rear =  0   -> true
deQueue()
  isEmpty? no ; front(0) == rear(0)? YES -> both = -1
                queue = [5, ·]      front = -1  rear = -1   -> true
                            ^ stale 5 is unreachable, never read

isEmpty()                                                   -> true
Rear()          isEmpty? yes                                -> -1
Front()         isEmpty? yes                                -> -1
deQueue()       isEmpty? yes                                -> false
```

The `front == rear -> reset` branch is what makes the sentinel design work. **Forget it and the queue becomes permanently non-empty**, because `front` would walk past `rear` and `isEmpty()` would never fire again.

### Dry run C — the `k = 1` boundary

```
init            queue = [·]         front = -1  rear = -1
enQueue(7)
  isFull? (-1+1)%1 = 0 == -1 ? no
  isEmpty? yes -> front = 0
  rear = (-1+1)%1 = 0 ; queue[0] = 7
                queue = [7]         front =  0  rear =  0   -> true
isFull()        (0+1)%1 = 0 == front 0 ? YES                -> true
enQueue(8)                                                  -> false
deQueue()       front == rear -> both = -1                  -> true
isEmpty()                                                   -> true
```

With `k = 1`, "exactly one element" and "full" are the same state, and the sentinel keeps them distinct from empty. Approach 3 (sacrificed slot) would need `new int[2]` here.

---

## 9. Bugs hit / caught in review

| # | Bug | Fix |
|---|---|---|
| 1 | Field named `size` while holding the **capacity** | Renamed to `capacity`. `queue[(rear+1) % size]` reads like a bug to a reviewer. |
| 2 | `isFull()` had a redundant `front == -1 && rear == -1` pre-check | Dead code — `(-1+1)%capacity = 0` never equals `front = -1`. Deleted. |
| 3 | `enQueue` had 3 branches with a duplicated `queue[rear] = value; return true;` | `(rear+1) % capacity` handles `-1 -> 0`, so the first-insert branch collapses. 3 branches -> 2. |
| 4 | Raw sentinel conditions inlined in `deQueue`/`Front`/`Rear` instead of calling `isEmpty()` | Two versions of the same invariant that can drift. Call the predicate everywhere. |
| 5 | `Rear()` tested `rear == -1` instead of `isEmpty()` | Correct only because both sentinels move together — relies on the invariant instead of stating the intent. |
| 6 | `queue` / `capacity` not `final` | Constructor-only assignment; `final` is free and documents it. |

**The meta-lesson (recurring gap — fast clean code under time pressure):**
> **Write `isEmpty()` and `isFull()` FIRST, before any mutator.** Then every method is one predicate call plus one action. This is faster to type, faster to debug, and impossible to desync. On the first pass I wrote the conditions inline and only extracted them later — extracting them up front costs nothing and removes a whole class of bug.

---

## 10. Complexity summary

| Approach | enQueue | deQueue | Front/Rear | Space | Capacity usable |
|---|---|---|---|---|---|
| 1. Naive linear | O(1) | O(1) | O(1) | O(k) | **broken** |
| 2. Shift on dequeue | O(1) | **O(k)** | O(1) | O(k) | k |
| 3. Circular, slot sacrificed | O(1) | O(1) | O(1) | O(k+1) | k |
| 4. Circular + `size` counter | O(1) | O(1) | O(1) | O(k) | k |
| 5. Circular + `-1` sentinel ← **submitted** | O(1) | O(1) | O(1) | O(k) | k |

---

## 11. Follow-ups to expect

**Q: "Distinguish full from empty — how many ways?"**
Three: sacrifice a slot, keep a `size` counter, or use an impossible sentinel. Name all three and the trade-off (one wastes a slot; one costs an extra field but gives `count()` free; one costs nothing but needs care when the last element leaves).

**Q: "Add an O(1) `count()`."**
Sentinel design has to derive it:
```java
public int count() {
    return isEmpty() ? 0 : (rear - front + capacity) % capacity + 1;
}
```
The `+ capacity` is mandatory — when `rear < front` (wrapped), `rear - front` is negative, and **Java's `%` returns a negative result for negative operands** (unlike Python). Check against Dry Run A's final state: `front = 1, rear = 0, capacity = 3` → `(0 - 1 + 3) % 3 + 1 = 2 + 1 = 3`. ✓
With Approach 4, `count()` is just `return size;`. **This is the strongest argument for the counter design.**

**Q: "Make it grow instead of rejecting."**
On full, allocate `2 * capacity`, then **copy in logical order, not physical** — `for (i = 0; i < count; i++) bigger[i] = queue[(front + i) % capacity];` — then reset `front = 0, rear = count - 1`. Amortized O(1). A straight `Arrays.copyOf` is wrong here: it preserves the physical layout and shatters a wrapped queue.

**Q: "Make it thread-safe."**
Naive: `synchronized` on every method. Better: `ReentrantLock` with two `Condition`s (`notFull`, `notEmpty`) → that's a bounded blocking queue, i.e. hand-rolling `ArrayBlockingQueue`. Single-producer/single-consumer can go lock-free with `AtomicInteger` head/tail. *(Ties directly into the concurrency arc — Lectures 51/52 monitor locks + inter-thread communication, and 53 advanced locks.)*

**Q: "Where is this used in production?"**
Ring buffers: log rotation, audio/video sample buffers, Disruptor (LMAX), Kafka producer record accumulator, TCP sliding window. The fixed-capacity + wrap-around design is the whole reason ring buffers are allocation-free in steady state.

---

## 12. Related problems

| Problem | Why next |
|---|---|
| **641. Design Circular Deque** | Direct sequel. Insert/delete at both ends. **Use the `size` counter (Approach 4)** — the sentinel gets messy across two ends. Watch `front = (front - 1 + capacity) % capacity` for `insertFront` — the `+ capacity` is mandatory. |
| **155. Min Stack** | Array-backed stack. Parallel `minSoFar[]` array, `minSoFar[i] = min` of everything at/below `i` → O(1) `getMin()`. Top index is always `size - 1`; don't keep a separate `top` field. |
| **232. Implement Queue using Stacks** | Amortized O(1) via two stacks; only transfer when the out-stack is empty. |
| **225. Implement Stack using Queues** | The reverse. One-queue solution: after enqueue, rotate `size - 1` elements. |
| **933. Number of Recent Calls** | Sliding-window queue, real use of the pattern. |
| **239. Sliding Window Maximum** | Monotonic deque — the deque from 641 put to work. |
| **146. LRU Cache** (done) | Same "design" muscle: pick the state representation so every op is O(1). |

---

## 13. Recall checklist

Before looking at the code again, can I answer these?

- [ ] What breaks in a naive linear array queue? *(slot leak — full-forever after k enqueues even when empty)*
- [ ] Why is shift-on-dequeue unacceptable? *(O(k) dequeue; elements shouldn't move, the window should)*
- [ ] What is the single core ambiguity? *(`front == rear` means both full and empty)*
- [ ] Name the three resolutions. *(sacrifice a slot / `size` counter / `-1` sentinel)*
- [ ] Sentinel version: why does `rear = (rear + 1) % capacity` handle the first insert with no special case? *(`-1 + 1 = 0`)*
- [ ] Sentinel version: why does `isFull()` need no empty guard? *(`(-1+1)%cap = 0 ≠ -1`)*
- [ ] Sentinel version: what must `deQueue` do when `front == rear`? *(reset BOTH to -1, else never empty again)*
- [ ] Counter version: what's the next-write index? *(`(front + size) % capacity`)*
- [ ] Counter version: what's `Rear()`? *(`(front + size - 1) % capacity`, guard empty first)*
- [ ] Why `+ capacity` before every `%` that can go negative? *(Java `%` returns negative for negative operands)*
- [ ] Can `rear < front`? *(yes — wrapped; array order ≠ queue order)*
- [ ] Complexity, stated properly? *(O(1) per op; O(k) storage required by the problem, O(1) auxiliary)*
- [ ] What does `k = 1` test? *(one-element state must stay distinguishable from empty)*
- [ ] How do you resize a wrapped queue? *(copy in logical order via `(front + i) % capacity`, not `Arrays.copyOf`)*

---

## 14. One-line takeaway

> A circular queue is a **fixed window sliding over a wrapped array** — the elements never move, only the window does. The entire problem reduces to disambiguating `front == rear`, and the entire clean-code lesson is: **define `isEmpty()`/`isFull()` first, then every method is one predicate plus one action.**