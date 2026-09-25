# LC 283 — Move Zeroes

**Difficulty:** Easy · **Sheets:** NeetCode | Striver SDE | Love Babbar 450 | Apna College
**Pattern:** Boundary write-pointer (slow/fast array pointers)

---

## 1. Problem

Given an integer array `nums`, move all `0`s to the end while **maintaining the relative order** of
the non-zero elements. Do this **in place**, without making a copy.

```
[0,1,0,3,12]  →  [1,3,12,0,0]
```

**Follow-up:** minimise the number of operations.

---

## 2. The idea

Keep one index, `writeIndex`, marking the **boundary of the finished region**.

```
[ 1, 3, 12, 0, 0 ]
           ↑
      writeIndex
  ── done ──┤── not yet placed ──
```

### The definition that makes everything else follow

> **`writeIndex` = the next slot where a non-zero element belongs.**
>
> **Invariant:** `nums[0 .. writeIndex)` holds non-zero elements in their final relative order.

Scan with `i`. When `nums[i]` is non-zero it belongs at `writeIndex` — place it and advance the
boundary. When `nums[i]` is zero, **do nothing**: the boundary stays parked, holding that slot open
for the next non-zero to claim.

That parking behaviour is the entire mechanism, and it is what dictates where the increment goes
(§5).

---

## 3. Approaches

| # | Approach | Time | Space | Writes | Verdict |
|---|---|---|---|---|---|
| 1 | New array: copy non-zeros, pad with 0s | O(n) | O(n) | 2n | ❌ problem forbids a copy |
| 2 | Find each 0 and shift everything left | O(n²) | O(1) | many | ❌ |
| 3 | Two pointers, **overwrite then fill** | O(n) | O(1) | n | ✅ |
| 4 | Two pointers, **swap** | O(n) | O(1) | ≤ 3× non-zeros | ✅ |

3 and 4 are the same idea with different write strategies. §6 compares them.

---

## 4. Solution (swap, one pass)

```java
class Solution {
    public void moveZeroes(int[] nums) {
        // writeIndex = the next slot where a non-zero element belongs.
        // Invariant: nums[0 .. writeIndex) are non-zeros in final relative order.
        int writeIndex = 0;

        for (int i = 0; i < nums.length; i++) {
            if (nums[i] != 0) {
                int temp = nums[i];
                nums[i] = nums[writeIndex];
                nums[writeIndex] = temp;
                writeIndex++;
            }
        }
    }
}
```

**O(n) time · O(1) space · one pass.**

### Notes

- **The self-swap when `i == writeIndex` is harmless** — swapping a slot with itself is a no-op.
  Guarding it with `if (i != writeIndex)` saves three assignments per already-placed element, which
  matters on arrays with few zeros. Correctness is unaffected either way; know which trade you took.
- **No trailing fill loop is needed.** Every zero encountered gets pushed rightward into the vacated
  slot automatically. If you still have a `while` loop padding zeros at the end, it's dead code.
- **Relative order is preserved** because `i` only moves forward and non-zeros are placed in the
  order encountered.

---

## 5. Common pitfalls — all on one line

Every classic bug in this problem is a misplaced `writeIndex++`. Each becomes impossible once the
definition in §2 is written down first.

### Pitfall 1 — copy-then-clear destroys the element

```java
if (nums[i] != 0) {
    nums[writeIndex] = nums[i];
    nums[i] = 0;            // 🚩 when i == writeIndex, this wipes what you just wrote
    writeIndex++;
}
```

**Trace `[1,2,3]` — an array with no zeros at all:**

| i | writeIndex | `nums[w] = nums[i]` | `nums[i] = 0` | array |
|---|---|---|---|---|
| 0 | 0 | `nums[0] = 1` (no-op) | `nums[0] = 0` 💥 | `[0,2,3]` |
| 1 | 1 | `nums[1] = 2` (no-op) | `nums[1] = 0` 💥 | `[0,0,3]` |
| 2 | 2 | `nums[2] = 3` (no-op) | `nums[2] = 0` 💥 | `[0,0,0]` |

Result: `[0,0,0]`. When `i == writeIndex` the copy is a no-op and the clear is pure destruction.

> ⚠️ **The sample input `[0,1,0,3,12]` hides this bug completely.** Choose a trace input that
> stresses the branch you're unsure about — here, "what if there are no zeros?"

### Pitfall 2 — the guard swallows the increment

```java
if (nums[i] != 0 && i != writeIndex) {   // 🚩 one combined condition
    swap(nums, i, writeIndex);
    writeIndex++;
}
```

Trace `[1,2,0,3]`: at `i = 0` the value is non-zero but `i == writeIndex`, so the whole block is
skipped — **and `writeIndex` never advances past the leading non-zeros.** By the time a real zero
appears, the boundary points at the wrong slot.

The `i != writeIndex` test is only about avoiding a pointless self-swap. It has nothing to do with
advancing the boundary.

### Pitfall 3 — hoisting the increment out of the `if`

```java
for (int i = 0; i < nums.length; i++) {
    if (nums[i] != 0) { swap(nums, i, writeIndex); }
    writeIndex++;        // 🚩 now advances on zeros too
}
```

`writeIndex` now advances every iteration, making it a second copy of `i`. Every swap becomes a
self-swap and nothing moves.

### The correct nesting

```java
if (nums[i] != 0) {            // outer: does this element get placed?
    if (i != writeIndex) {     // inner (optional): is the swap worth executing?
        swap(nums, i, writeIndex);
    }
    writeIndex++;              // inside the OUTER if, outside the INNER if
}
```

**Two nested checks, not one combined check.** The outer decides whether an element is placed; the
inner decides only whether the swap is worth executing.

### Pitfall 4 — `nums[i] > 0` instead of `nums[i] != 0`

Negative numbers are non-zero and must move left.

---

## 6. Swap vs overwrite-then-fill

The follow-up asks you to minimise operations. Two valid strategies:

### Overwrite-then-fill (two passes, one write each)

```java
int w = 0;
for (int i = 0; i < nums.length; i++) {
    if (nums[i] != 0) nums[w++] = nums[i];      // pass 1: compact
}
while (w < nums.length) nums[w++] = 0;          // pass 2: pad
```

### Comparison

| | writes | passes |
|---|---|---|
| overwrite-then-fill | (non-zeros) + (zeros) = **n** | 2 |
| swap, with self-swap guard | 3 × (non-zeros **after** the first zero) | 1 |
| swap, no guard | 3 × (non-zeros) | 1 |

**Neither strictly dominates.** Swap wins when zeros are rare *and* the self-swap is guarded.
Overwrite wins when zeros are common — a swap costs 3 assignments, a copy costs 1.

> **Interview answer:** *"Swap is a single pass and, with the `i != writeIndex` guard, never writes
> to an index unnecessarily. Overwrite-then-fill is two passes but each write is one assignment
> instead of three. Both are O(n); which is faster depends on the density of zeros."*

That's a stronger answer than declaring one optimal.

---

## 7. Dry run — `[0,1,0,3,12]`

| i | `nums[i]` | non-zero? | action | `writeIndex` after | array after |
|---|---|---|---|---|---|
| start | — | — | — | 0 | `[0,1,0,3,12]` |
| 0 | 0 | no | **nothing — boundary parks** | **0** | `[0,1,0,3,12]` |
| 1 | 1 | yes | swap(1, 0) | 1 | `[1,0,0,3,12]` |
| 2 | 0 | no | **nothing — boundary parks** | **1** | `[1,0,0,3,12]` |
| 3 | 3 | yes | swap(3, 1) | 2 | `[1,3,0,0,12]` |
| 4 | 12 | yes | swap(4, 2) | 3 | `[1,3,12,0,0]` |

✅ `[1,3,12,0,0]`

**Rows i=0 and i=2 are the whole problem in one line:** a zero leaves `writeIndex` parked, holding
its slot open. That is exactly why `writeIndex++` lives inside `if (nums[i] != 0)`.

> ⚠️ **Trace discipline:** track `writeIndex` explicitly in its own column. It's tempting to read
> each swap off the array visually and let the pointer column drift — but then the trace is
> confirming your intuition rather than checking it, and a wrong pointer value slips through
> unnoticed.

---

## 8. Edge cases

| case | expected | ✓ |
|---|---|---|
| `[1,2,3]` — no zeros | unchanged | ✅ (the case that exposes pitfall 1) |
| `[0,0,0]` — all zeros | unchanged | ✅ `writeIndex` stays 0, no swaps |
| `[0]` / `[5]` — single element | unchanged | ✅ |
| `[0,0,1]` — leading zeros | `[1,0,0]` | ✅ |
| `[1,0,0]` — trailing zeros | unchanged | ✅ |
| empty array | unchanged | ✅ loop never runs |
| negative numbers | treated as non-zero | ✅ the test is `!= 0`, not `> 0` |

---

## 9. Interview Q&A

**Q: Why does this preserve relative order?**
`i` only moves forward, and non-zeros are written to `writeIndex` in the order encountered. Nothing
ever moves a non-zero backwards past another.

**Q: Why is the self-swap safe?**
Swapping `nums[i]` with itself is a no-op. The guard is a performance optimisation, not a
correctness requirement.

**Q: Can you do it in one pass with fewer writes?**
Yes, with the `i != writeIndex` guard — then you only write when an element actually needs to move.
On an array with no zeros, that's zero writes.

**Q: What if you wanted all zeros at the *front* instead?**
Walk from the right with the boundary starting at `n-1`, or negate the predicate and reverse the
traversal. The structure is unchanged.

**Q: What if the predicate were something else — "move all negatives to the end"?**
Only the `if` condition changes. That's the sign that the pattern, not the zeros, is the content.

---

## 10. Pattern

> **One pointer marks the boundary of the finished region; it advances only when something is
> placed there.**

Note this says nothing about **zeros** — the zero test is just today's filter predicate. That's why
the same code shape solves Remove Element, Remove Duplicates, partitioning, and Dutch National
Flag.

This is the **boundary write-pointer** family, one of the highest-frequency array patterns in
interviews. Worth making automatic.

---

## 11. Takeaways

1. **Write the pointer's meaning in one sentence before writing the loop.** *"`writeIndex` = the
   next slot where a non-zero belongs."* Every placement decision then follows by force rather than
   by trial and error.
2. **Nest the conditions, don't merge them.** Outer: "does this element get placed?" Inner: "is the
   swap worth executing?"
3. **The boundary pointer must not advance on a skipped element.** That parking is the mechanism.
4. **Trace with the input that stresses the uncertain branch**, not the sample input.
5. **Track every variable explicitly in a trace.** Reading results off the array lets a wrong
   pointer value pass unnoticed.
6. **A self-swap is harmless.** Guarding it is a performance choice.
7. **Swap vs overwrite-then-fill is a genuine trade** (1 pass / 3 writes vs 2 passes / 1 write).
   Name both.
8. **`!= 0`, never `> 0`.** Negatives are non-zero.

---

## 12. Related problems

| # | Problem | Connection |
|---|---|---|
| 27 | Remove Element | Identical boundary pointer, different predicate — the bare-bones version |
| 26 | Remove Duplicates from Sorted Array | Same write-pointer; predicate is "differs from previous" |
| 80 | Remove Duplicates from Sorted Array II | Same, allowing two copies |
| 75 | Sort Colors (Dutch National Flag) | **Two** boundary pointers instead of one |
| 88 | Merge Sorted Array | Write pointer filled from the back |
| 905 | Sort Array By Parity | Same pattern, parity predicate |
| 2149 | Rearrange Array Elements by Sign | Two boundary pointers, alternating |