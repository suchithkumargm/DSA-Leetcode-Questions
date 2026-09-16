# LC 641 — Design Circular Deque

**Difficulty:** Medium
**Topic:** Design, Array, Queue, Deque
**Link:** https://leetcode.com/problems/design-circular-deque/
**Sheets:** Striver SDE (Stack & Queue), Apna College (Queue)
**Status:** Solved
**Date:** Sep 2026
**Prereq:** `LC622-Design-Circular-Queue.md` — read that first; this doc assumes the wrap-around and full-vs-empty ambiguity are already understood.

---

## 1. Problem

Fixed-capacity **double-ended** queue on an array. Insert and delete at *both* ends, all O(1).

| Method | Contract |
|---|---|
| `MyCircularDeque(k)` | capacity `k` |
| `insertFront(v)` / `insertLast(v)` | `false` if full |
| `deleteFront()` / `deleteLast()` | `false` if empty |
| `getFront()` / `getRear()` | `-1` if empty |
| `isEmpty()` / `isFull()` | booleans |

**Constraints:** `1 <= k <= 1000`, `0 <= value <= 1000`, up to `2000` calls.

---

## 2. What's new versus LC 622

622 had one moving pointer per operation and both pointers only ever moved **forward**. A deque doubles everything:

| | 622 queue | 641 deque |
|---|---|---|
| Insert points | rear only | **both ends** |
| Delete points | front only | **both ends** |
| `front` moves | forward only | forward *and* backward |
| `rear` moves | forward only | forward *and* backward |

That single change — pointers now move in **two directions** — is the entire source of difficulty, and it produced both bugs below. The wrap-around itself is already solved knowledge from 622.

**The rule to burn in:**

> `front` is the index of the **first** element. `rear` is the index of the **last** element.
> - `insertFront` claims the slot **before** `front` → `front--`
> - `deleteFront` releases `front` → `front++`
> - `insertLast` claims the slot **after** `rear` → `rear++`
> - `deleteLast` releases `rear` → `rear--`
>
> **Inserting and deleting at the same end move that pointer in opposite directions.** Two of the four go backward.

---

## 3. Approach 0 — Copy LC 622 verbatim (BROKEN)

The tempting move: the sentinel design from 622 was clean, so reuse it and just add the two new methods. But I copied the *forward* formula into both new methods:

```java
// WRONG
public boolean insertFront(int value) {
    if (isEmpty()) rear = 0;
    front = (front + 1) % capacity;   // <-- wrong direction
    queue[front] = value;
    return true;
}
public boolean deleteLast() {
    ...
    rear = (rear + 1) % capacity;     // <-- wrong direction
    return true;
}
```

### Failure trace — LeetCode's own example, `k = 3`

```
insertLast(1)    isEmpty -> front=0 ; rear=(-1+1)%3=0 ; q[0]=1
                 [1,·,·]  f=0  r=0                        -> true   ok
insertLast(2)    rear=1 ; q[1]=2
                 [1,2,·]  f=0  r=1                        -> true   ok
insertFront(3)   front=(0+1)%3=1 ; q[1]=3
                 [1,3,·]  f=1  r=1    <-- OVERWROTE the 2 -> true   CORRUPT
insertFront(4)   isFull? (1+1)%3=2 == front 1? no
                 front=2 ; q[2]=4
                 [1,3,4]  f=2  r=1                        -> true   WRONG (expected false)
```

Wrong by the fourth call. `front++` walks **toward** `rear`, so it overwrites live elements instead of claiming free ones, and the deque silently shrinks while reporting space it doesn't have.

Independent proof for `deleteLast`: fill `[1,2,3]` (`f=0, r=2`), then `deleteLast()`. The wrong formula sets `rear = (2+1)%3 = 0`, giving `f=0, r=0` — a **one**-element deque holding `1`. Two elements vanished and the survivor is wrong; `getRear()` returns `1` instead of `2`.

**Verdict:** wrong. Lesson: the direction of pointer movement is per-*operation*, not per-*pointer*.

---

## 4. Approach 1 — Fix the directions (STILL BROKEN)

Correct both formulas:

```java
front = (front - 1 + capacity) % capacity;   // insertFront
rear  = (rear  - 1 + capacity) % capacity;   // deleteLast
```

> ### The `+ capacity` rule
> **Java's `%` returns a negative result for negative operands** — `-1 % 3 == -1`, not `2`. (Python differs; don't carry that intuition over.) Any modulo whose operand can go negative must be written `(x - 1 + capacity) % capacity`. Omit it and you get `ArrayIndexOutOfBoundsException: -1`.

This fixes Approach 0's overwrite. But a new bug appears, and it's a direct consequence of the sentinel:

### Failure trace — `insertFront` into an EMPTY deque, `k = 3`

```
fresh deque      f=-1  r=-1

insertFront(9)   isEmpty? yes -> rear = 0
                 front = (-1 - 1 + 3) % 3 = 1        <-- expected 0
                 q[1] = 9
                 [·,9,·]  f=1  r=0

getFront()  -> queue[1] = 9   ok
getRear()   -> queue[0] = 0   WRONG — should be 9, reads an uninitialised slot
```

### Why — the sentinel does not rehydrate backward

In 622 the sentinel was elegant *because* `(-1 + 1) % capacity == 0` — the forward formula turned `-1` into `0` for free, so no first-insert special case was needed.

**That coincidence is one-directional.** Backward: `(-1 - 1 + capacity) % capacity == capacity - 2`. For `k = 3` that's `1`; for `k = 1000` it's `998`. Nowhere near `0`.

So `front` and `rear` land on *different* slots for a one-element deque and `rear` points at garbage.

**Why it hides in casual testing:** `k = 1` gives `(-1-1+1) % 1 = 0` and `k = 2` gives `(-1-1+2) % 2 = 0` — both accidentally correct. **The bug only appears at `k >= 3`.** Classic case of small test cases passing a broken implementation.

---

## 5. Approach 2 — Sentinel + `count` + symmetric empty branch (SUBMITTED)

Two changes: give both inserts an **explicit, identical** first-insert branch, and add a `count` field so `isEmpty`/`isFull` stop depending on pointer arithmetic entirely.

```java
class MyCircularDeque {

    private int[] queue;
    private int capacity;
    private int front = -1, rear = -1;
    private int count = 0;

    public MyCircularDeque(int k) {
        this.queue = new int[k];
        this.capacity = k;
    }

    public boolean insertFront(int value) {
        if (isFull()) return false;
        if (isEmpty()) front = rear = 0;                     // sentinel can't rehydrate backward
        else front = (front - 1 + capacity) % capacity;
        queue[front] = value;
        count++;
        return true;
    }

    public boolean insertLast(int value) {
        if (isFull()) return false;
        if (isEmpty()) front = rear = 0;                     // symmetric, though not strictly needed
        else rear = (rear + 1) % capacity;
        queue[rear] = value;
        count++;
        return true;
    }

    public boolean deleteFront() {
        if (isEmpty()) return false;
        if (front == rear) front = rear = -1;                // last element leaving
        else front = (front + 1) % capacity;
        count--;
        return true;
    }

    public boolean deleteLast() {
        if (isEmpty()) return false;
        if (front == rear) front = rear = -1;
        else rear = (rear - 1 + capacity) % capacity;
        count--;
        return true;
    }

    public int getFront() { return isEmpty() ? -1 : queue[front]; }
    public int getRear()  { return isEmpty() ? -1 : queue[rear];  }

    public boolean isEmpty() { return count == 0; }
    public boolean isFull()  { return count == capacity; }
}
```

**Complexity:** O(1) time for all eight methods — fixed arithmetic, no loops. O(k) space for the storage (required by the problem), O(1) auxiliary (4 scalar fields).

### Why `insertLast` gets the branch too, even though it doesn't need it

`insertLast`'s forward formula *would* still rehydrate `-1 → 0` correctly. But writing both inserts identically means **you never have to remember which direction tolerates the sentinel.** That memory burden is exactly what caused the bug. Symmetry costs one line and removes the question.

### Why `front == rear` safely means "one element left"

For `capacity > 1`, a full deque has `front != rear` (they're `capacity - 1` apart). So `front == rear` can only hold when `count == 1`. For `capacity == 1`, full *is* one element, so `front == rear` and `count == 1` agree anyway. The branch is sound at every capacity.

---

## 6. Approach 3 — `front` + `count` only, derive rear (BEST REPRESENTATION)

Approach 2 works, but look at what it tracks: `front`, `rear`, `count`, **and** a `-1` sentinel convention. `count` alone already answers empty and full — which makes `rear` and the sentinel *redundant state that can disagree with it*. Both bugs above were exactly that: pointers desyncing from reality.

**Eliminate `rear`. Derive it.**

```java
class MyCircularDeque {

    private final int[] queue;
    private final int capacity;
    private int front = 0, count = 0;

    public MyCircularDeque(int k) {
        queue = new int[k];
        capacity = k;
    }

    public boolean insertFront(int value) {
        if (isFull()) return false;
        front = (front - 1 + capacity) % capacity;
        queue[front] = value;
        count++;
        return true;
    }

    public boolean insertLast(int value) {
        if (isFull()) return false;
        queue[(front + count) % capacity] = value;     // next free slot after rear
        count++;
        return true;
    }

    public boolean deleteFront() {
        if (isEmpty()) return false;
        front = (front + 1) % capacity;
        count--;
        return true;
    }

    public boolean deleteLast() {
        if (isEmpty()) return false;
        count--;                                       // that's the whole method
        return true;
    }

    public int getFront() { return isEmpty() ? -1 : queue[front]; }
    public int getRear()  { return isEmpty() ? -1 : queue[(front + count - 1) % capacity]; }

    public boolean isEmpty() { return count == 0; }
    public boolean isFull()  { return count == capacity; }
}
```

### The two identities that replace `rear`

```
rear            = (front + count - 1) % capacity
next free slot  = (front + count)     % capacity
```

### Why this is strictly better

- **No sentinel.** `front = 0` is a valid index; emptiness lives in `count`.
- **No empty special-case in either insert.** Fresh deque, `insertFront(9)`, `k=3`: `front = (0-1+3)%3 = 2`, `queue[2] = 9`, and `getRear()` reads `queue[(2+1-1)%3] = queue[2] = 9`. **Consistent for free**, because rear is *derived* rather than *maintained*.
- **`deleteLast()` is one decrement.** No pointer to move ⇒ no direction to get backwards ⇒ **Approach 0's `deleteLast` bug is structurally impossible here.**
- **Same O(1) / O(k).** Strictly less state, identical complexity.
- Only 2 fields survive against 4.

**Interview line:** *"I'll track `front` and `count` and derive rear — that's one source of truth, so there are fewer invariants to keep in sync."*

---

## 7. Alternative — doubly linked list

Legitimate and sometimes expected. Head/tail sentinel nodes, insert/delete by relinking.

- **Time:** O(1) all ops
- **Space:** O(k), but with a large constant — each node is object header + value + 2 references (~32–40 bytes vs 4 bytes for an `int` slot)
- **Downside:** no cache locality (nodes scattered on the heap), per-insert allocation, GC pressure
- **Upside:** capacity need not be fixed; no wrap-around arithmetic at all

**For a *fixed*-capacity deque the array wins** — contiguous memory, zero allocation in steady state. This is the same reasoning behind ring buffers in production (see §9). Worth saying aloud: it's a real trade-off answer, not a dodge.

---

## 8. Dry runs (Approach 2, the submitted code)

### Dry run A — the official LeetCode example, `k = 3`

`·` = stale/unreachable slot.

```
init                    [·,·,·]  f=-1  r=-1  count=0

insertLast(1)
  isFull? 0==3 no ; isEmpty? yes -> f = r = 0
  q[0] = 1 ; count=1
                        [1,·,·]  f= 0  r= 0  count=1        -> true

insertLast(2)
  isEmpty? no -> rear = (0+1)%3 = 1
  q[1] = 2 ; count=2
                        [1,2,·]  f= 0  r= 1  count=2        -> true

insertFront(3)
  isFull? 2==3 no ; isEmpty? no
  front = (0-1+3)%3 = 2              <-- wraps BACKWARD to the tail
  q[2] = 3 ; count=3
                        [1,2,3]  f= 2  r= 1  count=3        -> true
  logical order: q[2],q[0],q[1] = 3 -> 1 -> 2

insertFront(4)
  isFull? 3==3 YES                                          -> false

getRear()               q[1] = 2                            -> 2
isFull()                                                    -> true

deleteLast()
  isEmpty? no ; f(2)==r(1)? no
  rear = (1-1+3)%3 = 0 ; count=2
                        [1,·,3]  f= 2  r= 0  count=2        -> true
  logical order: 3 -> 1

insertFront(4)
  isFull? 2==3 no ; isEmpty? no
  front = (2-1+3)%3 = 1
  q[1] = 4 ; count=3
                        [1,4,3]  f= 1  r= 0  count=3        -> true
  logical order: q[1],q[2],q[0] = 4 -> 3 -> 1

getFront()              q[1] = 4                            -> 4
```

All outputs match the expected LeetCode output. **Note the final state: `f=1, r=0`, so `rear < front`.** Perfectly valid — the deque is wrapped. Physical array `[1,4,3]`, logical deque `4 → 3 → 1`. **Array order is not deque order.**

### Dry run B — the `k >= 3` empty-insertFront boundary (the bug that was fixed)

This is the case Approach 1 failed. Verifying the fix:

```
fresh deque, k = 3      [·,·,·]  f=-1  r=-1  count=0

insertFront(9)
  isFull? 0==3 no
  isEmpty? yes -> f = r = 0          <-- explicit branch, NOT the backward formula
  q[0] = 9 ; count=1
                        [9,·,·]  f= 0  r= 0  count=1        -> true

getFront()              q[0] = 9                            -> 9   ✓
getRear()               q[0] = 9                            -> 9   ✓  (was 0 before the fix)
```

Both pointers on the same slot, as a one-element deque requires.

### Dry run C — full drain, sentinel reset

```
[·,·,·]  f=-1 r=-1 count=0
insertLast(5)     f=r=0 ; q[0]=5 ; count=1     [5,·,·]  f=0 r=0 count=1
deleteFront()
  isEmpty? no ; f(0)==r(0)? YES -> f = r = -1 ; count=0
                                               [5,·,·]  f=-1 r=-1 count=0
                                                ^ stale 5, never read again
isEmpty()                                      -> true
getFront() / getRear()                         -> -1 / -1
deleteLast()                                   -> false
insertFront(6)
  isEmpty? yes -> f = r = 0 ; q[0]=6 ; count=1  <-- recovers correctly
                                               [6,·,·]  f=0 r=0 count=1
```

The delete methods reset the sentinel, so the deque returns to a fresh state and the *next* `insertFront` hits the empty branch again. **That is why Approach 1's bug wasn't a one-time init issue — it recurred after every full drain.**

### Dry run D — `k = 1`

```
init            [·]  f=-1 r=-1 count=0
insertFront(7)  isEmpty -> f=r=0 ; q[0]=7 ; count=1   [7] f=0 r=0 count=1  -> true
isFull()        1==1                                                        -> true
insertLast(8)                                                               -> false
getFront()/getRear()                                                        -> 7 / 7
deleteLast()    f==r -> f=r=-1 ; count=0                                    -> true
isEmpty()                                                                   -> true
```

With `k = 1`, "one element" and "full" are the same state; `count` keeps it distinct from empty.

---

## 9. Bugs hit

| # | Bug | Root cause | Fix |
|---|---|---|---|
| 1 | `insertFront` overwrote live elements; `insertFront` after full returned `true` | Copied 622's **forward** formula into an operation that must move **backward** | `front = (front - 1 + capacity) % capacity` |
| 2 | `deleteLast` deleted two elements and kept the wrong one | Same directional error, mirrored | `rear = (rear - 1 + capacity) % capacity` |
| 3 | `insertFront` into an empty deque left `front != rear`; `getRear()` read an uninitialised slot. **Only reproduces at `k >= 3`** | The `-1` sentinel rehydrates `-1 → 0` **only forward**; backward it gives `capacity - 2` | Explicit `if (isEmpty()) front = rear = 0;` in **both** inserts |
| 4 | Three sources of truth (`front`, `rear`, `count` + sentinel) | Redundant state that can disagree | Approach 3: keep `front` + `count`, derive rear |
| 5 | `int count = 0;` was package-private | Missing modifier | `private int count = 0;` |
| 6 | `queue` / `capacity` not `final` | Constructor-only assignment | `private final` |

### The two lessons, stated plainly

> **1. Direction is per-operation, not per-pointer.** Both `front` and `rear` move in *both* directions in a deque. Inserting and deleting at the same end move that pointer opposite ways. Write all four formulas down before writing any code.
>
> **2. Every redundant field is a bug waiting to happen.** Bugs 1–3 were all pointer-desync bugs, and they are *unreachable* when there is a single source of truth. **Choose the representation that makes bugs impossible, not the one that makes them fixable.** This is the same skill as LC 146 (LRU Cache): the design work is picking the state, not writing the methods.

---

## 10. Complexity summary

| Approach | All ops | Space | Fields | Status |
|---|---|---|---|---|
| 0. 622 copied verbatim | O(1) | O(k) | 3 | **broken** — wrong directions |
| 1. Directions fixed, sentinel kept | O(1) | O(k) | 3 | **broken** at `k >= 3` — sentinel can't rehydrate backward |
| 2. Sentinel + `count` + symmetric branch ← **submitted** | O(1) | O(k) | 4 | correct |
| 3. `front` + `count`, rear derived | O(1) | O(k) | 2 | correct, **best representation** |
| 4. Doubly linked list | O(1) | O(k), big constant | — | correct, worse for fixed capacity |

---

## 11. Follow-ups to expect

**Q: "How does `java.util.ArrayDeque` do this?"**
Same ring buffer, with one optimisation worth knowing: it **rounds capacity up to a power of two** so it can replace `% capacity` with `& (capacity - 1)` — a single AND instruction instead of an integer division, which is one of the slowest ALU ops. It also has no `count` field; it stores `head` and `tail` and grows (doubles) rather than rejecting, which is why it can afford `head == tail` meaning empty. Strong answer — it shows you know the library, not just the exercise.

**Q: "Implement `deleteFront`/`insertLast` without `%` at all."**
```java
front = (front + 1 == capacity) ? 0 : front + 1;
```
A compare-and-branch beats a division, and it is well predicted since the wrap is rare. This is what real ring buffers do when capacity isn't a power of two.

**Q: "Make it grow instead of rejecting."**
Allocate `2 * capacity`, then **copy in logical order**: `for (i = 0; i < count; i++) bigger[i] = queue[(front + i) % capacity];` then `front = 0`. A plain `Arrays.copyOf` is **wrong** — it preserves the physical layout and shatters a wrapped deque. Amortised O(1).

**Q: "Thread-safe version?"**
`synchronized` on every method is the naive answer. Better: `ReentrantLock` with two `Condition`s (`notFull`, `notEmpty`) → that's a bounded blocking deque, i.e. hand-rolling `LinkedBlockingDeque` / `ArrayBlockingQueue`. Single-producer/single-consumer can go lock-free with atomic head/tail. *(Ties into concurrency lectures 51/52 — monitor locks and inter-thread communication — and 53 advanced locks.)*

**Q: "Where do ring buffers show up in production?"**
Log rotation, audio/video sample buffers, LMAX Disruptor, Kafka's producer record accumulator, TCP sliding windows. Fixed capacity + wrap-around is precisely what makes them **allocation-free in steady state** — no GC pressure on the hot path.

---

## 12. Related problems

| Problem | Link to this one |
|---|---|
| **622. Design Circular Queue** (done) | Prereq. Single-ended, pointers only move forward. See `LC622-Design-Circular-Queue.md`. |
| **239. Sliding Window Maximum** | **Do this next.** The monotonic deque put to work — this is *why* the deque matters. Deque of indices, evict from front when out of window, evict from rear while smaller than the incoming element. O(n). |
| **862. Shortest Subarray with Sum ≥ K** | Monotonic deque over a prefix-sum array. Hard, and the natural sequel to 239. |
| **1696. Jump Game VI** | Monotonic deque + DP. |
| **155. Min Stack** | Array-backed stack, parallel `minSoFar[]`. `top` index is always `size - 1` — don't keep a separate field (same "derive, don't maintain" lesson). |
| **232 / 225. Queue via Stacks / Stack via Queues** | Same design muscle, different substrate. |
| **146. LRU Cache** (done) | The general lesson: pick the state representation so every op is O(1) *and* invariants can't desync. |
| **933. Number of Recent Calls** | Simplest real application of a sliding queue. |

---

## 13. Recall checklist

- [ ] Which of the four deque operations move a pointer **backward**? *(`insertFront`, `deleteLast`)*
- [ ] Why can't 622's formula be copied into `insertFront`? *(`front++` walks toward rear and overwrites live elements)*
- [ ] Why must every backward modulo be `(x - 1 + capacity) % capacity`? *(Java `%` returns negative for negative operands)*
- [ ] Why does the `-1` sentinel work forward but not backward? *(`(-1+1)%cap = 0` but `(-1-1+cap)%cap = cap-2`)*
- [ ] At what `k` does the empty-`insertFront` bug first appear, and why not below it? *(`k >= 3`; `k=1` and `k=2` accidentally give 0)*
- [ ] In Approach 2, why is `front == rear` a safe test for "last element leaving"? *(a full deque has them `capacity-1` apart, so it can only hold at `count == 1`)*
- [ ] Approach 3: the two identities? *(`rear = (front + count - 1) % capacity`; next free slot `= (front + count) % capacity`)*
- [ ] Approach 3: why is `deleteLast()` just `count--`? *(rear is derived, so nothing to update)*
- [ ] Which bug becomes structurally impossible in Approach 3, and why? *(directional `deleteLast` bug — no `rear` field exists to move wrongly)*
- [ ] Can `rear < front`? *(yes, wrapped — array order ≠ deque order)*
- [ ] Complexity, stated properly? *(O(1) every op; O(k) storage required by the problem, O(1) auxiliary)*
- [ ] How do you resize a wrapped deque? *(copy in logical order via `(front + i) % capacity`, never `Arrays.copyOf`)*
- [ ] What does `ArrayDeque` do that this doesn't? *(power-of-two capacity → `& (capacity - 1)` instead of `%`)*
- [ ] Array vs doubly linked list for a fixed-capacity deque? *(array: contiguous, zero allocation, cache-friendly; DLL: ~8–10× memory per element, GC pressure, but unbounded)*

---

## 14. One-line takeaway

> A deque is a circular queue where **both** pointers move in **both** directions — so write all four movement formulas before writing any code, remember that Java's `%` goes negative, and prefer the representation with **one source of truth** (`front` + `count`, rear derived), because every redundant field is a desync bug waiting to happen.