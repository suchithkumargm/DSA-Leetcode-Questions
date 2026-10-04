# LC 1200 — Minimum Absolute Difference

**Difficulty:** Easy
**Tags:** Sorting, Array, Greedy-ish (adjacency argument)
**Language:** Java
**Date solved:** Sep 2026

---

## 1. Problem

Given an array of **distinct** integers `arr`, find all pairs `[a, b]` such that:

- `a < b`
- `b - a` equals the **minimum absolute difference** between any two elements in the array

Return the pairs in **ascending order**.

### Constraints (these matter a lot here)

| Constraint | Value |
|---|---|
| `arr.length` | `2 .. 10^5` |
| `arr[i]` | `-10^6 .. 10^6` |
| Elements | **all distinct** |

> The distinctness + bounded range combo is what unlocks the counting/bitset approach in §5. Always read constraints before deciding the approach.

### Examples

```
arr = [4,2,1,3]                     -> [[1,2],[2,3],[3,4]]     (minDiff = 1)
arr = [1,3,6,10,15]                 -> [[1,3]]                 (minDiff = 2)
arr = [3,8,-10,23,19,-4,-14,27]     -> [[-14,-10],[19,23],[23,27]]  (minDiff = 4)
```

---

## 2. The Core Insight (the only real "aha" in this problem)

> **After sorting, the minimum difference must occur between two *adjacent* elements.**

**Why?** Take any pair `(arr[i], arr[j])` with `i < j` in the sorted array. Then:

```
arr[j] - arr[i] = (arr[i+1] - arr[i]) + (arr[i+2] - arr[i+1]) + ... + (arr[j] - arr[j-1])
```

The gap between a non-adjacent pair is the **sum of the adjacent gaps in between**. Since the array is sorted, every adjacent gap is `>= 0`, so a non-adjacent pair can never be *smaller* than the smallest adjacent gap it is made up of.

**Consequence:** we only need to examine `n - 1` adjacent pairs, not all `n(n-1)/2` pairs.
This is the single step that takes us from `O(n²)` to `O(n log n)`.

Two more small facts that follow from sorting:

1. `arr[i] - arr[i-1]` is already `>= 0`, so **`Math.abs` is unnecessary**.
2. Scanning left→right produces pairs already in ascending order → **no final sort of the answer needed**.

---

## 3. Approach 1 — Brute Force

Check every pair, track the minimum, then collect.

```java
// Conceptual — not submitted, TLEs at n = 10^5
public List<List<Integer>> bruteForce(int[] arr) {
    int n = arr.length;
    int minDiff = Integer.MAX_VALUE;

    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            minDiff = Math.min(minDiff, Math.abs(arr[i] - arr[j]));

    List<List<Integer>> ans = new ArrayList<>();
    for (int i = 0; i < n; i++)
        for (int j = i + 1; j < n; j++)
            if (Math.abs(arr[i] - arr[j]) == minDiff)
                ans.add(List.of(Math.min(arr[i], arr[j]), Math.max(arr[i], arr[j])));

    Collections.sort(ans, (a, b) -> a.get(0) - b.get(0));  // needed! pairs come out unordered
    return ans;
}
```

| Metric | Value | Intuition |
|---|---|---|
| Time | `O(n²)` | ~5×10⁹ operations at n = 10⁵ → **TLE** |
| Space | `O(1)` aux | plus output |

**Why it's bad:** it re-derives, for every pair, information that sorting gives us for free. It also forces an extra sort of the *output* because pairs are discovered in arbitrary order.

---

## 4. Approach 2 — Sort + Two Passes (my first accepted solution)

Sort, scan once to find `minDiff`, scan again to collect matching adjacent pairs.

```java
class Solution {
    public List<List<Integer>> minimumAbsDifference(int[] arr) {
        Arrays.sort(arr);

        int minDiff = Integer.MAX_VALUE;
        for (int i = 1; i < arr.length; i++) {
            minDiff = Math.min(minDiff, arr[i] - arr[i-1]);   // no Math.abs needed
        }

        List<List<Integer>> ans = new ArrayList<>();
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] - arr[i-1] == minDiff) {
                ans.add(List.of(arr[i-1], arr[i]));           // immutable, no copy
            }
        }
        return ans;
    }
}
```

| Metric | Value | Intuition |
|---|---|---|
| Time | `O(n log n)` | dominated by `Arrays.sort`; the two scans are `2n` = negligible |
| Space | `O(1)` aux | ignoring output list; `Arrays.sort` on primitives is in-place |

**Properties:**
- Never allocates a pair that gets thrown away — `minDiff` is known before collection starts.
- Two passes, but both are linear and cache-friendly.

---

## 5. Approach 3 — Sort + Single Pass (cleanest reasoning)

Don't precompute `minDiff`. Maintain it as a running best, and **invalidate the collected answer whenever a strictly smaller gap appears**.

```java
class Solution {
    public List<List<Integer>> minimumAbsDifference(int[] arr) {
        Arrays.sort(arr);

        List<List<Integer>> ans = new ArrayList<>();
        int minDiff = Integer.MAX_VALUE;

        for (int i = 1; i < arr.length; i++) {
            int diff = arr[i] - arr[i-1];

            if (diff < minDiff) {    // found a better minimum
                minDiff = diff;
                ans.clear();         // everything collected so far is now invalid
            }
            if (diff == minDiff) {   // NOT else-if — the line above may have just made this true
                ans.add(List.of(arr[i-1], arr[i]));
            }
        }
        return ans;
    }
}
```

### Two things that are easy to get wrong here

1. **Order of the ifs.** The `diff < minDiff` block must come first, and the second must be a plain `if`, not `else if`. When a new minimum is found we still need to add *that* pair.
2. **`ans.clear()` must happen before the add**, otherwise the new pair gets wiped too.

| Metric | Value | Intuition |
|---|---|---|
| Time | `O(n log n)` | same sort, one scan instead of two |
| Space | `O(1)` aux | same |

**Honest note:** this is *not faster* than the two-pass version. It can be marginally **slower**, because when gaps arrive in decreasing order it allocates pairs that `clear()` immediately discards. Total allocations are still bounded by `n-1`, so no asymptotic change — but there's no runtime win. **Prefer it for the clearer reasoning, not for speed.** In an interview it demonstrates the invariant "a new minimum invalidates prior results," which the two-pass version hides behind precomputation.

---

## 6. Approach 4 — Counting / Bitset (constraint exploitation, `O(n + range)`)

Because values are bounded by `±10⁶` **and distinct**, we can skip comparison sorting entirely. Distinctness means one *bit* per value suffices — no counts needed.

```java
class Solution {
    public List<List<Integer>> minimumAbsDifference(int[] arr) {
        final int OFFSET = 1_000_000;
        final int SIZE   = 2_000_001;

        boolean[] present = new boolean[SIZE];
        int lo = Integer.MAX_VALUE, hi = Integer.MIN_VALUE;

        for (int v : arr) {
            present[v + OFFSET] = true;
            lo = Math.min(lo, v + OFFSET);
            hi = Math.max(hi, v + OFFSET);
        }

        // Pass 1: find the minimum gap between consecutive present values
        int minDiff = Integer.MAX_VALUE, prev = -1;
        for (int i = lo; i <= hi; i++) {
            if (!present[i]) continue;
            if (prev != -1) minDiff = Math.min(minDiff, i - prev);
            prev = i;
        }

        // Pass 2: collect pairs with that gap
        List<List<Integer>> ans = new ArrayList<>();
        prev = -1;
        for (int i = lo; i <= hi; i++) {
            if (!present[i]) continue;
            if (prev != -1 && i - prev == minDiff) {
                ans.add(List.of(prev - OFFSET, i - OFFSET));
            }
            prev = i;
        }
        return ans;
    }
}
```

| Metric | Value | Intuition |
|---|---|---|
| Time | `O(n + range)` | `range` = 2×10⁶ fixed |
| Space | `O(range)` | 2 MB `boolean[]` (or 250 KB with a real bitset / `long[]`) |

**Is it actually better?** At `n = 10⁵`, `n log n ≈ 1.7×10⁶` vs `range = 2×10⁶`. It's a **wash** — and we traded `O(1)` space for 2 MB. Worth knowing the technique exists (it's the same idea as counting sort), but say out loud in an interview that it only pays off when `n` is large relative to the value range.

---

## 7. Dry Run

### 7a. Two-pass, on LC Example 3

```
arr = [3, 8, -10, 23, 19, -4, -14, 27]
sorted = [-14, -10, -4, 3, 8, 19, 23, 27]
```

**Pass 1 — find minDiff:**

| i | arr[i-1] | arr[i] | diff | minDiff after |
|---|---|---|---|---|
| 1 | -14 | -10 | 4  | 4 |
| 2 | -10 | -4  | 6  | 4 |
| 3 | -4  | 3   | 7  | 4 |
| 4 | 3   | 8   | 5  | 4 |
| 5 | 8   | 19  | 11 | 4 |
| 6 | 19  | 23  | 4  | 4 |
| 7 | 23  | 27  | 4  | 4 |

`minDiff = 4`

**Pass 2 — collect:**

| i | diff | == 4? | ans |
|---|---|---|---|
| 1 | 4  | yes | `[[-14,-10]]` |
| 2 | 6  | no  | `[[-14,-10]]` |
| 3 | 7  | no  | `[[-14,-10]]` |
| 4 | 5  | no  | `[[-14,-10]]` |
| 5 | 11 | no  | `[[-14,-10]]` |
| 6 | 4  | yes | `[[-14,-10],[19,23]]` |
| 7 | 4  | yes | `[[-14,-10],[19,23],[23,27]]` |

Output: `[[-14,-10],[19,23],[23,27]]` ✅
Note `23` appears in two pairs — that's allowed, pairs may share an element.

---

### 7b. Single-pass, on an input that actually triggers `clear()`

```
arr = [40, 11, 26, 27, -20]
sorted = [-20, 11, 26, 27, 40]
gaps:        31   15   1   13     <- strictly decreasing at first: worst case for clear()
```

Initial: `minDiff = MAX_VALUE`, `ans = []`

| i | pair | diff | `diff < minDiff`? | action | minDiff | ans |
|---|---|---|---|---|---|---|
| 1 | (-20, 11) | 31 | yes (31 < MAX) | `clear()` (no-op), then add | 31 | `[[-20,11]]` |
| 2 | (11, 26)  | 15 | yes (15 < 31)  | `clear()` **discards `[-20,11]`**, then add | 15 | `[[11,26]]` |
| 3 | (26, 27)  | 1  | yes (1 < 15)   | `clear()` **discards `[11,26]`**, then add | 1  | `[[26,27]]` |
| 4 | (27, 40)  | 13 | no             | `13 == 1`? no → skip | 1 | `[[26,27]]` |

Output: `[[26,27]]` ✅

This trace is exactly the wasted-allocation case mentioned in §5: two pairs were built and immediately thrown away.

---

## 8. Java-Specific Notes Learned Here

### `List.of(a, b)` vs `new ArrayList<>(List.of(a, b))`
My first version wrapped in `new ArrayList<>()`, allocating **two** objects per pair and copying elements between them. Since the inner lists are never mutated, `List.of(...)` alone is correct and cheaper. Use the `ArrayList` wrapper **only** if the inner list must later be modified — `List.of` returns an immutable list and throws `UnsupportedOperationException` on `add`/`set`.

`Arrays.asList(a, b)` is the older idiom: fixed-size, but `set` works. `List.of` also rejects `null` elements; `Arrays.asList` allows them.

### `Math.abs` after sorting
Redundant. On an `int` the JIT compiles `Math.abs` to a branch-free sign trick, so removing it is **not** a measurable speedup — but removing it *documents* that the sorted invariant is being used.

### `Arrays.sort(int[])`
Dual-pivot Quicksort — `O(n log n)` average, in-place, no boxing.
`Arrays.sort(Integer[])` would be TimSort + boxing overhead. Always sort the primitive array when possible.

---

## 9. On LeetCode Runtime Percentiles (important lesson from this problem)

Submission history on identical-complexity solutions:

| Version | Beats |
|---|---|
| Two-pass, `Math.abs`, `new ArrayList<>(List.of(...))` | 33% |
| Two-pass, no `abs`, plain `List.of(...)` | 98% |
| Single-pass with `clear()` | 33% |

**The only real improvement was dropping the redundant `ArrayList` allocation.** The rest is noise.

Why: total runtime here is a few milliseconds, dominated by `Arrays.sort` plus JIT warmup and harness overhead. Thousands of submissions land in that same few-millisecond band, so the percentile curve is nearly vertical — **1 ms of jitter swings you 60 percentile points**. Resubmitting identical code produces different numbers.

> **Takeaway:** on problems whose runtime is dominated by a library sort, optimize for correctness, clarity, and allocation count. Do **not** chase the percentile number — it isn't a measurement.

---

## 10. Pattern Summary — Transferable Takeaways

1. **"Minimum difference between any two elements" → sort first.** After sorting, only adjacent pairs can achieve the minimum. This generalizes: LC 539 (Minimum Time Difference), LC 530 / 783 (min abs diff in a BST → in-order traversal *is* the sorted order).
2. **Running-minimum + invalidate pattern.** When collecting all items achieving an extremum in one pass: `if (better) { best = x; results.clear(); } if (equal) results.add(...)`. Plain `if`, not `else if`. Reusable for "all pairs with max sum," "all shortest paths of equal length," etc.
3. **Read the constraints for value bounds.** Small bounded range + distinct elements → counting sort / bitset becomes available. Check whether it actually beats `n log n` before committing.
4. **Sorted scan → output already sorted.** Saves a final `Collections.sort` that the brute force needs.
5. **Count your allocations in hot loops.** Redundant wrapper objects are one of the few genuine constant-factor wins available in Java.

---

---

## Final Answer (the version to remember)

```java
class Solution {
    public List<List<Integer>> minimumAbsDifference(int[] arr) {
        Arrays.sort(arr);

        List<List<Integer>> ans = new ArrayList<>();
        int minDiff = Integer.MAX_VALUE;

        for (int i = 1; i < arr.length; i++) {
            int diff = arr[i] - arr[i-1];
            if (diff < minDiff) {
                minDiff = diff;
                ans.clear();
            }
            if (diff == minDiff) {
                ans.add(List.of(arr[i-1], arr[i]));
            }
        }
        return ans;
    }
}
```

**Time `O(n log n)` · Space `O(1)` auxiliary**