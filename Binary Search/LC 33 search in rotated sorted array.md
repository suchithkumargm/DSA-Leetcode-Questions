# LC 33 — Search in Rotated Sorted Array

**Difficulty:** Medium
**Pattern:** Modified Binary Search / "Half is always sorted"
**Language:** Java
**Related:** LC 153, LC 154, LC 81, LC 162, LC 852

---

## 1. Problem

Given a sorted array of **distinct** integers that has been **rotated** at some unknown pivot, find the index of `target`. Return `-1` if absent.

**Must run in O(log n).**

```
Original:  [0, 1, 2, 4, 5, 6, 7]
Rotated 4: [4, 5, 6, 7, 0, 1, 2]
                       ^ pivot (smallest element)
```

**Constraints:**
- `1 <= nums.length <= 5000`
- `-10^4 <= nums[i] <= 10^4`
- All values in `nums` are **unique**
- `nums` is guaranteed to be a rotated sorted array (rotation of 0 counts — array may be fully sorted)

---

## 2. The Core Observation (read this first)

A rotated sorted array is **two sorted runs glued together**, where every element in the left run is greater than every element in the right run.

```
[4, 5, 6, 7, | 0, 1, 2]
 └─ run A ─┘   └ run B ┘
 all of A  >  all of B
```

Now the key insight that makes binary search work:

> **If you cut a rotated sorted array at ANY midpoint, at least one of the two halves is a clean, fully-sorted range.**

Why? There is only **one** "break point" (the pivot). A single break can only land in one of the two halves. The other half is untouched and therefore sorted.

```
[4, 5, 6, 7, 0, 1, 2]      mid = index 3 (value 7)
 └── 4..7 ──┘  └0..2┘
   SORTED       SORTED     (here both happen to be, that's fine)

[5, 6, 7, 0, 1, 2, 4]      mid = index 3 (value 0)
 └─5,6,7─┘ └─0,1,2,4─┘
  SORTED     SORTED

[6, 7, 0, 1, 2, 4, 5]      mid = index 3 (value 1)
 └6,7,0,1┘ └─2,4,5─┘
  BROKEN     SORTED   ← only the right half is usable
```

Once you can identify the sorted half, you can ask a question you *can* answer with certainty:
**"Does target lie inside this sorted half's range?"**
- **Yes** → discard the other half.
- **No** → it must be in the other half, so discard this one.

Either way you throw away half the array → O(log n).

This is the whole problem. Everything below is bookkeeping.

---

## 3. Approach 1 — Brute Force (Linear Scan)

Ignore the structure entirely. Walk the array.

```java
class Solution {
    public int search(int[] nums, int target) {
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] == target) {
                return i;
            }
        }
        return -1;
    }
}
```

**Time:** O(n) — worst case touches every element.
**Space:** O(1).

**Why it's rejected:** The problem explicitly demands O(log n). This solution never uses the fact that the array is sorted-ish. It would work identically on random data — that's the tell that you're leaving information on the table.

> ⚠️ **Note on "beats 100%":** With `n <= 5000`, this passes instantly and LeetCode's runtime percentile is basically measuring server noise. **Never** use runtime percentile as evidence your complexity is good. State your Big-O out loud instead.

**What's blocking the optimisation:** Plain binary search needs a globally sorted array to know which direction to move. The rotation breaks that guarantee. So the question becomes: *can I restore the guarantee?*

---

## 4. Approach 2 — Find the Pivot, then Binary Search (the "better" step)

The most natural first optimisation: if the rotation is what's in the way, **find where the rotation happened**, then you have two perfectly sorted subarrays and can run textbook binary search on the right one.

### Step A: Find the pivot (index of the minimum element)

Binary search on the property: `nums[mid] > nums[end]` means the minimum lies strictly to the **right** of `mid`.

```
[4, 5, 6, 7, 0, 1, 2]
 mid=7, end=2  →  7 > 2  →  minimum is to the right, discard mid and left
```

If `nums[mid] <= nums[end]`, the right half (including `mid`) is sorted, so the minimum is `mid` or to its left — keep `mid` as a candidate (`end = mid`, not `mid - 1`).

### Step B: Pick the correct sorted half, run standard binary search

```java
class Solution {
    public int search(int[] nums, int target) {
        int n = nums.length;
        int pivot = findPivot(nums);   // index of the smallest element

        if (pivot == 0) {                       // not rotated at all
            return binarySearch(nums, 0, n - 1, target);
        }
        if (target >= nums[0]) {                // target lives in the left run
            return binarySearch(nums, 0, pivot - 1, target);
        }
        return binarySearch(nums, pivot, n - 1, target);   // right run
    }

    private int findPivot(int[] nums) {
        int start = 0, end = nums.length - 1;
        while (start < end) {                   // note: <  not <=
            int mid = start + (end - start) / 2;
            if (nums[mid] > nums[end]) {
                start = mid + 1;                // min strictly right of mid
            } else {
                end = mid;                      // mid could BE the min — keep it
            }
        }
        return start;                           // start == end == pivot index
    }

    private int binarySearch(int[] nums, int lo, int hi, int target) {
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) return mid;
            else if (nums[mid] < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return -1;
    }
}
```

**Time:** O(log n) + O(log n) = **O(log n)** — two passes, but constants don't change the class.
**Space:** O(1).

**Verdict:** This is a completely valid, accept-worthy answer. It's also **more code, two loops, and two sets of boundary conditions to get right**. In an interview it's more surface area to bug. It's genuinely useful to know though, because `findPivot` *is* the answer to LC 153, and LC 33 with duplicates (LC 81) is easier to reason about pivot-first.

**What's still wasteful:** We traverse the array twice conceptually. We find the pivot, then we throw that work away and search again. Can we make the *single* search smart enough to handle the rotation inline?

---

## 5. Approach 3 — One-Pass Modified Binary Search (OPTIMAL)

Instead of locating the pivot up front, decide at **each step** which half is sorted, and use that half's range as the decision oracle.

```java
class Solution {
    public int search(int[] nums, int target) {
        int start = 0, end = nums.length - 1;

        while (start <= end) {
            int mid = start + (end - start) / 2;

            if (nums[mid] == target) {
                return mid;
            }
            else if (nums[start] <= nums[mid]) {        // LEFT half is sorted
                if (target >= nums[start] && target < nums[mid]) {
                    end = mid - 1;                      // target inside left range
                } else {
                    start = mid + 1;                    // must be on the right
                }
            }
            else {                                      // RIGHT half is sorted
                if (target > nums[mid] && target <= nums[end]) {
                    start = mid + 1;                    // target inside right range
                } else {
                    end = mid - 1;                      // must be on the left
                }
            }
        }

        return -1;
    }
}
```

**Time:** **O(log n)** — the search window halves every iteration. `n → n/2 → n/4 → ... → 1` takes log₂(n) steps. For n = 5000 that's ~13 iterations.
**Space:** **O(1)** — a fixed number of int variables, iterative so no call stack growth.

### The decision table (memorise this shape, not the code)

| Which half is sorted | Test | Then |
|---|---|---|
| Left (`nums[start] <= nums[mid]`) | `nums[start] <= target < nums[mid]` | go left (`end = mid - 1`) |
| Left | otherwise | go right (`start = mid + 1`) |
| Right (else) | `nums[mid] < target <= nums[end]` | go right (`start = mid + 1`) |
| Right | otherwise | go left (`end = mid - 1`) |

Notice the symmetry: **you always test against the sorted half's two endpoints.** That's the only range you can make a confident claim about.

---

## 6. Dry Runs

### Dry Run A — target found, exercises the left-sorted branch

`nums = [4, 5, 6, 7, 0, 1, 2]`, `target = 0`

| # | start | end | mid | nums[mid] | Sorted half | Reasoning | Action |
|---|---|---|---|---|---|---|---|
| 1 | 0 | 6 | 3 | 7 | Left, `nums[0]=4 <= 7` | Is `0` in `[4, 7)`? **No** | `start = 4` |
| 2 | 4 | 6 | 5 | 1 | Left, `nums[4]=0 <= 1` | Is `0` in `[0, 1)`? **Yes** | `end = 4` |
| 3 | 4 | 4 | 4 | 0 | — | `nums[4] == target` | **return 4** ✅ |

3 iterations for n = 7. `log₂(7) ≈ 2.8`. Checks out.

---

### Dry Run B — target absent, loop must terminate

`nums = [4, 5, 6, 7, 0, 1, 2]`, `target = 3`

| # | start | end | mid | nums[mid] | Sorted half | Reasoning | Action |
|---|---|---|---|---|---|---|---|
| 1 | 0 | 6 | 3 | 7 | Left `[4, 7]` | `3` in `[4, 7)`? No | `start = 4` |
| 2 | 4 | 6 | 5 | 1 | Left `[0, 1]` | `3` in `[0, 1)`? No | `start = 6` |
| 3 | 6 | 6 | 6 | 2 | Left `[2, 2]` | `3` in `[2, 2)`? No | `start = 7` |
| 4 | 7 | 6 | — | — | — | `start > end`, loop exits | **return −1** ✅ |

Note iteration 3: when `start == end == mid`, the "left half" is a single element and `nums[start] <= nums[mid]` is `2 <= 2` → true. **This is exactly why the `<=` matters** (see §7).

---

### Dry Run C — exercises the right-sorted branch

`nums = [5, 6, 7, 0, 1, 2, 4]`, `target = 6`

| # | start | end | mid | nums[mid] | Sorted half | Reasoning | Action |
|---|---|---|---|---|---|---|---|
| 1 | 0 | 6 | 3 | 0 | Right, `nums[0]=5 <= 0`? **No** → right sorted `[0, 4]` | `6` in `(0, 4]`? No | `end = 2` |
| 2 | 0 | 2 | 1 | 6 | — | `nums[1] == target` | **return 1** ✅ |

---

### Dry Run D — no rotation at all (edge case)

`nums = [1, 2, 3, 4, 5]`, `target = 5`

| # | start | end | mid | nums[mid] | Sorted half | Reasoning | Action |
|---|---|---|---|---|---|---|---|
| 1 | 0 | 4 | 2 | 3 | Left `[1, 3]` | `5` in `[1, 3)`? No | `start = 3` |
| 2 | 3 | 4 | 3 | 4 | Left `[4, 4]` | `5` in `[4, 4)`? No | `start = 4` |
| 3 | 4 | 4 | 4 | 5 | — | match | **return 4** ✅ |

A non-rotated array is just "rotated by 0" — the left half is always sorted, and the algorithm degenerates gracefully into plain binary search. No special-casing needed.

---

## 7. ⚠️ The `<=` is Load-Bearing

`else if (nums[start] <= nums[mid])` — **the `=` cannot be dropped.**

When the window narrows to **two elements**, `mid = start + (end - start) / 2` rounds down, so `mid == start`, which means `nums[start] == nums[mid]`.

**Counter-example with strict `<`:** `nums = [3, 1]`, `target = 1`

| With `<=` (correct) | With `<` (BUG) |
|---|---|
| `start=0, end=1, mid=0`, `nums[0]=3` | same |
| `3 <= 3` → true → left half `[3,3]` sorted | `3 < 3` → **false** → assumes right half sorted |
| `1` in `[3, 3)`? No → `start = 1` | Is `1` in `(3, 1]`? `1 > 3` false → `end = -1` |
| `start=1, end=1, mid=1`, `nums[1]=1` → **return 1** ✅ | loop exits → **return −1** ❌ |

The single-element "half" `[nums[mid], nums[mid]]` is trivially sorted, and treating it as such is what keeps the invariant true. This is the #1 bug people ship on this problem.

---

## 8. Other Bugs & Gotchas

| Bug | Symptom | Fix |
|---|---|---|
| `int mid = (start + end) / 2` | Integer overflow when `start + end > 2^31 - 1` | `start + (end - start) / 2`. Doesn't bite at n=5000, but it's the correct habit and interviewers notice. |
| `while (start < end)` instead of `<=` | Misses the target when the window shrinks to one element | Use `<=` for "find exact value" searches. Use `<` for "converge to a boundary" searches (like `findPivot`). |
| Comparing against `nums[mid]` when the half isn't sorted | Wrong branch taken, target lost | Only ever range-test against the **sorted** half's endpoints. |
| Forgetting the `nums[mid] == target` check first | Infinite loop or off-by-one | Check equality before the branching logic. |
| `end = mid` in the main loop | Infinite loop (window stops shrinking when `mid == start`) | Always `mid - 1` / `mid + 1` here, since `mid` is already ruled out by the equality check. |

**On `<` vs `<=` in the inner range tests:** By the time you reach the branches, you already know `nums[mid] != target`. So `target < nums[mid]` and `target <= nums[mid]` behave identically. Your original code used `<=` and was still correct — but `<` documents the fact that `mid` is already eliminated, and it's the version to write.

---

## 9. Complexity Summary

| Approach | Time | Space | Notes |
|---|---|---|---|
| Brute force linear scan | O(n) | O(1) | Ignores all structure. Fails the stated requirement. |
| Find pivot + binary search | O(log n) | O(1) | Two passes. More code, more boundary cases. Reuses LC 153. |
| **One-pass modified binary search** | **O(log n)** | **O(1)** | **Optimal.** Single loop, single set of invariants. |

**Why O(log n):** each iteration eliminates exactly half the remaining window. Number of halvings to reach size 1 is log₂(n).

**Why O(1) space:** iterative, so no recursion stack. Only `start`, `end`, `mid` regardless of input size.

> A recursive version would be O(log n) space due to the call stack. Java doesn't guarantee tail-call optimisation, so **prefer iterative binary search** in interviews.

---

## 10. Reusable Mental Template

```
while (start <= end):
    mid = start + (end - start) / 2
    if nums[mid] == target: return mid

    if LEFT half is sorted:            # nums[start] <= nums[mid]
        if target in [nums[start], nums[mid]):  go left
        else:                                    go right
    else:                              # RIGHT half must be sorted
        if target in (nums[mid], nums[end]]:     go right
        else:                                    go left
return -1
```

**The generalisable idea:** binary search doesn't require a *sorted array*. It requires a **predicate you can evaluate at `mid` that reliably tells you which half to discard.** Here the predicate is built by first identifying which half is trustworthy. Carry that framing forward — it unlocks LC 162, LC 875, LC 410, and most "binary search on answer" problems.

---

## 11. Follow-Ups (in the order worth doing them)

| Problem | Why | Connection |
|---|---|---|
| **LC 153** — Find Min in Rotated Sorted Array | Cleanest distillation of the pattern | It's `findPivot` from §4. Compare against `nums[end]`, not `nums[start]`. |
| **LC 81** — Search in Rotated Array II | Adds duplicates | `nums[start] == nums[mid] == nums[end]` makes "which half is sorted" undecidable. Must shrink by one (`start++`) → worst case **O(n)**. Knowing *why* it degrades is a strong interview answer. |
| **LC 154** — Find Min in Rotated Array II | LC 153 + duplicates | Same degradation reasoning. |
| **LC 162** — Find Peak Element | Binary search with **no sorted array at all** | Predicate is `nums[mid] < nums[mid+1]` → a peak exists to the right. Breaks the "sorted" mental model usefully. |
| **LC 852** — Peak Index in a Mountain Array | Easier version of 162 | Good warm-up. |

---

## 12. Interview Talking Points

Say these out loud — they're what separates "solved it" from "understands it":

1. **"The array has exactly one break point, so cutting at any midpoint leaves at least one half fully sorted."** ← lead with this, it's the whole insight.
2. **"I test the target against the sorted half's endpoints, because that's the only range where membership is decidable."**
3. **"The `<=` handles the two-element window where `mid` collapses onto `start`."** — proactively mentioning this signals you've actually debugged binary search before.
4. **"`start + (end - start) / 2` avoids overflow."**
5. **"With duplicates this degrades to O(n) worst case, because you can no longer tell which half is sorted."** — great unprompted extension.
6. **"There's also a two-pass version: find the pivot with binary search, then binary search the correct run. Same complexity, but more boundary conditions to get right, so I prefer the one-pass."**

---

## 13. Recall Checklist

Before your next revision, can you answer these without looking?

- [ ] Why is at least one half always sorted?
- [ ] Which endpoints do you compare against, and why those?
- [ ] What breaks if `nums[start] <= nums[mid]` becomes `<`? Give the failing input.
- [ ] Why `while (start <= end)` here but `while (start < end)` in `findPivot`?
- [ ] Why is space O(1) and not O(log n)?
- [ ] What's the worst case with duplicates, and why?
- [ ] Write it from scratch in under 5 minutes.