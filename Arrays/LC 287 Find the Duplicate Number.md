# LC 287 — Find the Duplicate Number

**Difficulty:** Medium · **Sheets:** NeetCode 150 | Striver SDE
**Pattern:** Floyd's cycle detection on an array treated as a function graph

---

## 1. Problem

Given an array `nums` of `n + 1` integers where each value is in the range `[1, n]`, there is
exactly **one repeated number**. Return it.

**Constraints that define the problem:**
- You **must not modify** the array.
- You must use only **O(1) extra space**.

```
[1,3,4,2,2]   → 2
[3,1,3,4,2]   → 3
```

Note the repeated number may appear **more than twice**.

---

## 2. Why the constraints are the problem

Without them this is trivial. The two restrictions kill the obvious solutions:

| Approach | Time | Space | Killed by |
|---|---|---|---|
| Sort, then scan for adjacent equals | O(n log n) | O(log n) | **modifies the array** |
| HashSet of seen values | O(n) | **O(n)** | **extra space** |
| Negate `nums[abs(v)]` as a visited marker | O(n) | O(1) | **modifies the array** (restorable, but still) |
| Binary search on the value range | O(n log n) | O(1) | ✅ legal, see §7 |
| **Floyd's cycle detection** | **O(n)** | **O(1)** | ✅ the intended answer |

> Both "obvious" solutions are worth stating in an interview *with the reason they're disqualified*
> — it shows you read the constraints rather than missed them.

**Set version, done right:** don't remove elements as you go. Just check membership before adding —
the first value already present is the answer.

---

## 3. The setup — why an array is a linked list here

Look at the constraints again: **`n + 1` integers, each in `[1, n]`.** So **every value is a valid
index**. That is not a coincidence; it is the entire construction.

> ### 🔑 The fact
> **`i → nums[i]` is a `next` pointer.** The array *is* a linked list: the index is the node, and
> the value stored there is the pointer to the next node.

### Why a cycle must exist

There are only `n + 1` indices, and every hop lands on one. After `n + 1` hops you've used them all,
so the next hop must revisit — **pigeonhole principle**. No visited set needed, and none allowed.

### Why the cycle entrance is the answer

> **Two different indices `i ≠ j` with `nums[i] == nums[j]` — that is exactly what a duplicate
> value *is*.** And those two indices both point at the same next index, which is precisely a node
> with two incoming edges. That node is the cycle entrance.

So the duplicated **value** and the entrance **index** are the same number. Finding the entrance
finds the duplicate.

### Why index 0 is a safe starting point

Values are in `[1, n]`, so **no value is ever 0** — nothing points at index 0. It can never be
inside the cycle, which makes it a valid "head": the tail leading into the cycle.

### Worked example — `nums = [1,3,4,2,2]`

```
index:  0  1  2  3  4
value:  1  3  4  2  2

start at 0
nums[0] = 1  → index 1
nums[1] = 3  → index 3
nums[3] = 2  → index 2
nums[2] = 4  → index 4
nums[4] = 2  → index 2    ← revisited
```

Chain of indices: `0, 1, 3, 2, 4, 2, 4, 2, …`
Tail = `0 → 1 → 3`, cycle = `2 → 4 → 2`, **entrance = index 2 = the answer** ✅

Indices 3 and 4 both hold the value 2 — two edges into node 2. That's the duplicate.

---

## 4. Solution — Floyd's, two phases

```java
class Solution {
    public int findDuplicate(int[] nums) {
        int slow = 0;
        int fast = 0;

        // PHASE 1 — find a meeting point inside the cycle
        do {
            slow = nums[slow];             // one hop
            fast = nums[nums[fast]];       // two hops
        } while (slow != fast);

        // PHASE 2 — find the cycle entrance
        slow = 0;
        while (slow != fast) {
            slow = nums[slow];             // one hop
            fast = nums[fast];             // one hop — NOT two anymore
        }

        return slow;                       // the entrance INDEX is the duplicated VALUE
    }
}
```

**O(n) time · O(1) space.**

### Details that matter

**`do-while`, not `while`.** Both pointers start at 0, so `while (slow != fast)` never enters. Either
use a `do-while` or advance both once before the loop.

**The loop has no other exit condition.** Unlike LC 141, there's no null to guard against — every
index is valid and every hop lands somewhere. The *only* termination is the meeting. Any additional
condition is invented.

**Phase 1 returns nothing.** It locates a meeting point somewhere *inside* the cycle, which is not
the entrance. Phase 2 is mandatory.

**Return `slow`, not `nums[slow]`.** The entrance index **is** the duplicated value — that's the
whole construction (§3). Returning `nums[slow]` returns the *next* node instead.

**It doesn't matter which pointer you reset.** They're at the same node; convention resets `slow`.

**Both move at speed 1 in phase 2.** Keeping `fast` at speed 2 breaks it entirely.

### Why `fast` must catch `slow` (phase 1)

> Once both are inside the cycle, `fast` gains **exactly one position** per iteration (2 − 1 = 1).
> **A gap that changes by exactly 1 cannot skip over zero** — it must land on it.

No arithmetic needed. Same argument as LC 141.

### Why phase 2 lands on the entrance

Let `L` = distance from start to the entrance, `x` = distance from entrance to the meeting point,
`C` = cycle length.

- `slow` travelled `L + x`; `fast` travelled `2(L + x)`.
- The difference, `L + x`, must be a whole number of laps: **`L + x ≡ 0 (mod C)`**.
- So from the meeting point, walking `L` more steps completes the lap and lands on the entrance.
- And walking `L` from the start lands on the entrance by definition.

Both pointers need exactly `L` steps to reach the same node, so moving them in lockstep makes them
meet there.

### Complexity, justified rather than asserted

- **Phase 1:** at most `tail + cycle` hops.
- **Phase 2:** exactly `L` hops.
- Both bounded by `n`. → **O(n)**. No formula required.

---

## 5. Common pitfalls

**Asking "do I need a visited array?"** No — that's the O(n)-space solution Floyd's exists to avoid.
The whole point is detecting a cycle with two pointers and no bookkeeping.

**Inventing a loop condition.** e.g. `while (fast != nums[nums.length - 1])`. Unmotivated and wrong.
In an array-as-graph there is nothing to guard; the meeting is the only exit.

**Returning from phase 1.** The meeting point is inside the cycle, not at its entrance.

**Returning `nums[slow]` instead of `slow`.**

**`while` instead of `do-while`** with both pointers at 0 → the loop body never executes.

**Keeping `fast` at speed 2 in phase 2.**

**Starting the two pointers at *different* positions.**

```java
int slow = nums[0];
int fast = nums[nums[0]];     // 🚩 one hop ahead — infinite loop in phase 2
```

The §4 derivation requires `fast` to have travelled **exactly twice** `slow`'s distance. An offset
start breaks that, so `L + x ≡ 0 (mod C)` no longer holds and phase 2's pointers chase each other
forever. On `[1,3,4,2,2]` they land on 2 and 4 alternately and never coincide.

> **Starting both at `nums[0]` is fine** — they're co-located, just one hop in from index 0, and
> phase 2 then resets to `nums[0]` to match. The derivation needs the two pointers **together**,
> not specifically at index 0. The conventional form starts both at `0` (safe because no value is
> 0, so nothing points there) and needs a `do-while` to enter the loop; the `nums[0]` variant is
> equally correct. **Pick one and make phase 2's reset match it.**

**Nesting phase 2 inside the phase-1 loop.**

```java
do {
    slow = nums[slow];
    fast = nums[nums[fast]];
    if (slow == fast) { /* phase 2 */ return slow; }   // 🚩
} while (slow != fast);
return nums[0];                                        // 🚩 unreachable
```

Correct, but the inner `if` tests exactly what the loop condition tests one line later, which leaves
a trailing `return` that can never execute. An unreachable statement makes a reader stop and verify.
Two sequential loops say the same thing with nothing dead.

---

## 6. Dry run — `nums = [1,3,4,2,2]`

### Phase 1

| iter | slow = `nums[slow]` | fast = `nums[nums[fast]]` | meet? |
|---|---|---|---|
| start | 0 | 0 | — (do-while skips the test) |
| 1 | `nums[0]` = 1 | `nums[nums[0]]` = `nums[1]` = 3 | 1 ≠ 3 |
| 2 | `nums[1]` = 3 | `nums[nums[3]]` = `nums[2]` = 4 | 3 ≠ 4 |
| 3 | `nums[3]` = 2 | `nums[nums[4]]` = `nums[2]` = 4 | 2 ≠ 4 |
| 4 | `nums[2]` = 4 | `nums[nums[4]]` = `nums[2]` = 4 | **4 == 4** ✅ |

Meeting point: index 4.

### Phase 2

`slow = 0`, `fast = 4`.

| iter | slow | fast | meet? |
|---|---|---|---|
| 1 | `nums[0]` = 1 | `nums[4]` = 2 | 1 ≠ 2 |
| 2 | `nums[1]` = 3 | `nums[2]` = 4 | 3 ≠ 4 |
| 3 | `nums[3]` = 2 | `nums[4]` = 2 | **2 == 2** ✅ |

Return **2** ✅ — which matches the entrance found by hand in §3.

---

## 7. The other legal solution — binary search on the value range

Worth knowing, and the one people reach for without the graph framing.

> For a candidate value `v`, count how many array elements are **≤ v**. If the array had no
> duplicates, exactly `v` elements would be ≤ `v`. **A count greater than `v` means the duplicate
> lies in `[1, v]`.**

```java
public int findDuplicate(int[] nums) {
    int low = 1, high = nums.length - 1;
    while (low < high) {
        int mid = low + (high - low) / 2;
        int count = 0;
        for (int num : nums) if (num <= mid) count++;

        if (count > mid) high = mid;    // duplicate is in [low, mid]
        else             low = mid + 1; // duplicate is in [mid+1, high]
    }
    return low;
}
```

**O(n log n) time · O(1) space.** Worse than Floyd's, but legal, easier to explain, and it's the
same **binary-search-on-answer** shape as LC 1552 — the predicate *"count ≤ v exceeds v"* is
monotone in `v`.

---

## 8. Edge cases

| case | result | note |
|---|---|---|
| `[1,1]` | 1 | smallest valid input, n = 1 |
| duplicate appears many times, `[2,2,2,2,2]` | 2 | cycle is a self-loop at index 2 |
| duplicate is 1, `[1,1,2]` | 1 | |
| duplicate is n | works | |
| array already "sorted-ish" | works | the chain doesn't care about order |

⚠️ Floyd's requires the guarantee that values are in `[1, n]` and the array has `n + 1` entries. If
a value could be 0 or out of range, the array is no longer a valid function graph and the whole
construction collapses.

---

## 9. Interview Q&A

**Q: How is an array a linked list here?**
Every value is in `[1, n]`, so every value is a valid index. `i → nums[i]` is the next pointer.

**Q: Why must there be a cycle?**
`n + 1` indices, every hop lands on one — pigeonhole forces a revisit.

**Q: Why is the cycle entrance the duplicate?**
The entrance is the only node with two incoming edges, which means two indices hold the same value.
That value is the duplicate, and it equals the entrance's index.

**Q: Why does phase 2 work?**
`L + x ≡ 0 (mod C)`, so both a pointer at the start and a pointer at the meeting point need exactly
`L` steps to reach the entrance. §4.

**Q: Why start at index 0?**
No value is 0, so nothing points at index 0 — it can't be inside the cycle, which makes it a valid
tail.

**Q: Can you do it without Floyd's, still in O(1) space?**
Yes — binary search on the value range, O(n log n). §7.

**Q: What if the array could be modified?**
Then index-as-marker: for each value `v`, negate `nums[abs(v)]`; the first already-negative slot you
hit is the duplicate. O(n)/O(1), but it mutates the input.

**Q: What if there were multiple duplicates?**
Floyd's finds one cycle, so it returns one duplicate — not all. The problem's "exactly one repeated
number" guarantee is load-bearing.

---

## 10. Pattern

> **`i → nums[i]` turns an array into a linked list, and the cycle entrance is the duplicate —
> because two indices pointing at the same place is exactly what a duplicate is.**

The transferable half: **when values happen to be valid indices, the array is a graph.** That
reframing also drives LC 41, LC 442, LC 448 and LC 645.

---

## 11. Takeaways

1. **Values in `[1, n]` with `n+1` slots is a signal, not a detail.** Every value being a valid index
   is what makes the array a function graph.
2. **Pigeonhole guarantees the cycle.** No visited tracking needed — or allowed.
3. **The entrance is the node with two incoming edges**, which is the duplicate.
4. **Floyd's exists to avoid tracking visits.** If you're reaching for a visited set, you've left the
   technique.
5. **In an array-as-graph there is nothing to null-guard.** The only loop exit is the meeting.
6. **Return the index, not the value at it.**
7. **Both pointers must start together** — at `0` with a `do-while`, or both at `nums[0]`. What
   breaks is an *offset* start, and it breaks phase 2, not phase 1.
8. **Keep the two phases as sequential loops.** Nesting phase 2 inside phase 1 leaves an unreachable
   return.
9. **Phase 2 moves both at speed 1.**
10. **Binary search on the value range is the backup O(1)-space answer** — monotone predicate, same
    shape as LC 1552.
11. **Read the constraints first.** Both "obvious" solutions here are disqualified *by the
    constraints*, which is the whole reason the problem is medium.