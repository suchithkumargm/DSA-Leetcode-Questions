# LC 26 — Remove Duplicates from Sorted Array

**Difficulty:** Easy · **Sheets:** NeetCode | Striver SDE | Love Babbar 450 | Apna College
**Pattern:** Boundary write-pointer (slow/fast array pointers)

---

## 1. Problem

Given a **sorted** integer array `nums`, remove duplicates **in place** so each element appears
once, preserving order. Return `k`, the number of unique elements. The first `k` slots of `nums`
must hold them; whatever follows doesn't matter.

```
[1,1,2]        → k = 2, nums = [1,2,_]
[0,0,1,1,1,2,2,3,3,4] → k = 5, nums = [0,1,2,3,4,_,_,_,_,_]
```

The array is **sorted** — that's what makes duplicates contiguous and the whole problem O(n).

---

## 2. The definition that makes everything else follow

> **`k` = the next slot where a unique element belongs.**
>
> **Invariant:** `nums[0 .. k)` holds the unique elements seen so far, in order.

Scan with `i` from index 1. When `nums[i]` is a **new** value it belongs at `k` — write it and
advance. When it's a duplicate, **do nothing**: `k` stays parked, holding that slot for the next
new value.

`k` starts at **1**, not 0, because `nums[0]` is always unique by definition — nothing precedes it.

> ⚠️ **`k` never points at a duplicate.** Naming it `duplicatePointer` describes something it never
> holds, and a name that lies can't guide you when you're deciding where the increment goes.

---

## 3. Solution

```java
class Solution {
    public int removeDuplicates(int[] nums) {
        if (nums.length == 0) return 0;

        int k = 1;                                  // next slot for a unique element

        for (int i = 1; i < nums.length; i++) {
            if (nums[i] != nums[i - 1]) {           // outer: is this element kept?
                if (i != k) {                       // inner (optional): is the write needed?
                    nums[k] = nums[i];
                }
                k++;                                // always, when an element is kept
            }
        }
        return k;
    }
}
```

**O(n) time · O(1) space · one pass.**

The inner `i != k` guard is optional — `nums[k] = nums[i]` when `i == k` is a harmless self-write.
It saves an assignment per already-placed element on arrays with few duplicates.

---

## 4. Which element do you compare against?

Two plausible candidates, and **for this problem both work**:

```java
if (nums[i] != nums[i - 1])   // compare with the previous element in the scan
if (nums[i] != nums[k - 1])   // compare with the last element actually kept
```

They agree here because the array is sorted, so all copies of a value are contiguous: the previous
scanned element and the last kept element are always the same value.

> ⚠️ **They diverge in LC 80** (allow up to two copies), where you must compare against
> `nums[k - 2]` — the *kept* region, not the scan position. Prefer `nums[k - 1]` as the default
> habit, because it generalises: it asks *"is this different from what I've already accepted?"*,
> which is the question the invariant is actually about.

---

## 5. Common pitfalls

### Merging the write-guard into the keep-test

```java
if (nums[i] != nums[i-1] && i != k) {   // 🚩
    nums[k] = nums[i];
    k++;
}
```

When `i == k` the entire block is skipped — including the increment — so `k` falls behind and never
recovers.

Trace `[1,2,3]` (no duplicates at all):

| i | test | `i != k`? | action | k after |
|---|---|---|---|---|
| 1 | `2 != 1` ✅ | `1 != 1` ❌ | **skip everything** | 1 |
| 2 | `3 != 2` ✅ | `2 != 1` ✅ | `nums[1] = 3`, k++ | 2 |

Returns 2; correct answer is 3.

> **The `i != k` test is only about skipping a pointless self-write. It has nothing to do with
> advancing.** Nest the two conditions; never merge them.
>
> Note the sample input `[1,1,2]` hides this bug. **Trace with the input that stresses the branch**
> — here, "what if nothing is a duplicate?"

### Starting `k` at 0

Then the first unique element is written to slot 0 twice and the count is off by one. `nums[0]` is
unique by definition, so the boundary starts at 1.

### Starting the scan at `i = 0`

`nums[i - 1]` underflows. Scan from 1.

### Advancing `k` on a duplicate

Then `k` is just a copy of `i`, every write is a self-write, and nothing is compacted.

### Trying to actually delete elements

The problem doesn't ask for that. Anything past index `k` is explicitly ignored — don't shift,
don't resize, don't null out.

---

## 6. Dry run — `[0,0,1,1,1,2,2,3,3,4]`

| i | `nums[i]` | `nums[i-1]` | new? | action | k after | `nums[0..k)` |
|---|---|---|---|---|---|---|
| start | — | — | — | — | 1 | `[0]` |
| 1 | 0 | 0 | no | **park** | 1 | `[0]` |
| 2 | 1 | 0 | yes | `nums[1] = 1` | 2 | `[0,1]` |
| 3 | 1 | 1 | no | **park** | 2 | `[0,1]` |
| 4 | 1 | 1 | no | **park** | 2 | `[0,1]` |
| 5 | 2 | 1 | yes | `nums[2] = 2` | 3 | `[0,1,2]` |
| 6 | 2 | 2 | no | **park** | 3 | `[0,1,2]` |
| 7 | 3 | 2 | yes | `nums[3] = 3` | 4 | `[0,1,2,3]` |
| 8 | 3 | 3 | no | **park** | 4 | `[0,1,2,3]` |
| 9 | 4 | 3 | yes | `nums[4] = 4` | 5 | `[0,1,2,3,4]` |

Return **5** ✅

The "park" rows are the whole problem: a duplicate leaves `k` where it is, holding its slot open.
That's exactly why `k++` lives inside the keep-test.

---

## 7. Edge cases

| case | expected | ✓ |
|---|---|---|
| `[]` | 0 | ✅ with the length guard |
| `[1]` | 1 | ✅ loop never runs, returns 1 |
| `[1,1,1,1]` — all duplicates | 1 | ✅ `k` stays at 1 |
| `[1,2,3,4]` — no duplicates | 4 | ✅ (the case that exposes the merged-condition bug) |
| duplicates at the very end | correct count | ✅ they simply park |
| negative numbers | works | ✅ comparison is `!=`, not sign-based |

---

## 8. Interview Q&A

**Q: Why does this need the array to be sorted?**
So all copies of a value are contiguous. Comparing against the immediately preceding element is only
a valid duplicate test when equal values are adjacent. Unsorted, you'd need a set — O(n) space.

**Q: Why not delete the duplicates?**
Deletion in an array is O(n) per removal → O(n²). Overwriting into a compacted prefix is O(n), and
the problem explicitly doesn't care what's past index `k`.

**Q: What's in the array after index `k`?**
Leftover stale values. The problem says they're ignored. Don't waste time clearing them unless asked.

**Q: Generalise it to "allow at most two copies of each element."**
LC 80. Compare `nums[i]` against `nums[k - 2]` instead, and start `k` at 2. The structure is
unchanged — only the predicate moves.

**Q: What if you had to preserve the original array?**
Then you need O(n) output space — the in-place requirement is what makes the write-pointer necessary.

---

## 9. Pattern

> **One pointer marks the boundary of the finished region; it advances only when something is
> placed there.**

Note it says nothing about duplicates — that's just today's filter predicate. The same code shape
solves Remove Element, Move Zeroes, partitioning, and Dutch National Flag.

---

## 10. Takeaways

1. **Write the pointer's meaning in one sentence before the loop.** *"`k` = the next slot where a
   unique element belongs."* The increment's position then follows by force.
2. **Name the pointer after what it holds.** A name that lies can't guide a decision.
3. **Nest the conditions, don't merge them.** Outer: "is this element kept?" Inner: "is the write
   needed?"
4. **The boundary must not advance on a skipped element.** That parking is the mechanism.
5. **Prefer comparing against the *kept* region** (`nums[k-1]`) over the scan position
   (`nums[i-1]`) — it generalises to LC 80.
6. **Trace with the input that stresses the uncertain branch**, not the sample.
7. **`k` starts at 1** because `nums[0]` is unique by definition.