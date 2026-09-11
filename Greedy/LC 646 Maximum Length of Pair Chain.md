# LC 646 — Maximum Length of Pair Chain

**Difficulty:** Medium
**Patterns:** Greedy · Interval Scheduling · LIS-style DP · Counting Sort
**Language:** Java
**Final result:** beats ~94%

---

## 1. Problem Restated

You are given `pairs[i] = [left_i, right_i]` with the guarantee `left_i < right_i`.

A pair `(c, d)` **can follow** a pair `(a, b)` if and only if:

```
b < c        (STRICTLY less than — this matters a lot)
```

A **chain** is a sequence of pairs where each one follows the previous. You may pick the pairs in **any order** (you are not restricted to the array order) and you do **not** have to use all of them.

Return the length of the longest possible chain.

### Constraints (these unlock the final optimization)

```
n == pairs.length
1 <= n <= 1000
-1000 <= left_i < right_i <= 1000
```

Two things to burn into memory:
- The value range is **tiny and fixed** (`[-1000, 1000]`, 2001 possible values).
- `n <= 1000`, so even O(n²) passes. The fast solutions are about elegance + constant factors, not about avoiding TLE.

### Mental Model

Forget "pairs". These are **intervals on a number line**, and you want the **largest set of intervals that don't touch each other**.

```
        -6 ────────────── 9
           1 ──── 6
                    8 ── 10
        -1 ── 4
-6 ── -2
-9 ───────── 8
  -5 ── 3
     0 ── 3
```

This is the **Activity Selection Problem** in disguise. Once you see that, the solution is a classic.

---

## 2. Approach 1 — Brute Force (Subsets)

### Idea
Every possible chain is some subset of the pairs, arranged in sorted order. So: generate all `2^n` subsets, sort each one, check whether it forms a valid chain, track the max size.

### Code sketch
```java
// CONCEPTUAL ONLY — will not pass, n = 1000 means 2^1000 subsets
int best = 0;
for (int mask = 0; mask < (1 << n); mask++) {
    List<int[]> subset = new ArrayList<>();
    for (int i = 0; i < n; i++)
        if ((mask & (1 << i)) != 0) subset.add(pairs[i]);

    subset.sort((a, b) -> Integer.compare(a[0], b[0]));

    boolean valid = true;
    for (int i = 1; i < subset.size(); i++)
        if (subset.get(i - 1)[1] >= subset.get(i)[0]) { valid = false; break; }

    if (valid) best = Math.max(best, subset.size());
}
return best;
```

### Complexity
| | |
|---|---|
| Time | **O(2ⁿ · n log n)** — `2ⁿ` subsets × sorting/validating each |
| Space | **O(n)** for the current subset |

### Why it dies
`2^1000` is more than the number of atoms in the universe. Dead on arrival. But it is worth writing out once, because it establishes the **search space** and tells you the problem is fundamentally "pick a subset with a constraint" — which is the signature of DP or Greedy.

---

## 3. Approach 2 — Better: O(n²) DP (Longest Increasing Subsequence variant)

### The key realization
If you **sort the pairs by their left endpoint**, then any valid chain must appear as a *subsequence* of the sorted array. Why? Because in a chain, `b < c` and we know `c < d`, so left endpoints are also strictly increasing along a chain. Sorting by left endpoint puts chain members in the correct relative order automatically.

Once that's true, the problem becomes **exactly LIS**, with a custom "can I extend?" predicate:

> Instead of `nums[j] < nums[i]`, the predicate is `pairs[j][1] < pairs[i][0]`.

### Recurrence
```
dp[i] = length of the longest chain that ENDS with pairs[i]

dp[i] = 1 + max( dp[j] )   for all j < i where pairs[j][1] < pairs[i][0]
      = 1                  if no such j exists

answer = max(dp[i]) over all i
```

### Code
```java
class Solution {
    public int findLongestChain(int[][] pairs) {
        Arrays.sort(pairs, (a, b) -> Integer.compare(a[0], b[0]));
        int n = pairs.length;
        int[] dp = new int[n];
        Arrays.fill(dp, 1);

        int best = 1;
        for (int i = 1; i < n; i++) {
            for (int j = 0; j < i; j++) {
                if (pairs[j][1] < pairs[i][0]) {
                    dp[i] = Math.max(dp[i], dp[j] + 1);
                }
            }
            best = Math.max(best, dp[i]);
        }
        return best;
    }
}
```

### Complexity
| | |
|---|---|
| Time | **O(n²)** — dominates the O(n log n) sort |
| Space | **O(n)** for the `dp` array |

With `n <= 1000`, that's ~10⁶ operations. It **passes**, but slowly (typically bottom 20–30% on LeetCode).

### Why this is still worth knowing
1. It's the **general** tool. If the problem asked for max *weight* chain instead of max *count*, greedy would break and this DP would be the answer.
2. It teaches you the "sort to impose an order, then LIS" transformation, which shows up constantly (Russian Doll Envelopes LC 354, Box Stacking, Maximum Height by Stacking Cuboids LC 1691).

---

## 4. Approach 3 — Optimal: Greedy by Right Endpoint ⭐

### The insight (this is the whole problem)

> **Sort by RIGHT endpoint. Then greedily take every pair whose left endpoint clears the last taken right endpoint.**

Intuition: the pair that **finishes earliest** leaves the most room on the number line for everything after it. Greedily grabbing the earliest-finishing compatible interval can never hurt you.

This is the standard **Activity Selection / Interval Scheduling** greedy.

### Why sorting by LEFT and going greedy FAILS

```java
pairs = [[1, 100], [2, 3], [4, 5]]
```
Sorted by left: `[1,100], [2,3], [4,5]`.
Greedy takes `[1,100]` first, and now nothing fits → answer **1**.
True answer is **2** (`[2,3] → [4,5]`).

A huge interval with a small left endpoint poisons the whole line. Sorting by *right* endpoint structurally prevents this: `[1,100]` sorts to the very end, so it's considered last, after the small ones have already been banked.

### Proof of correctness (exchange argument — say this in an interview)

Let `G = g₁, g₂, g₃, ...` be the greedy chain and `O = o₁, o₂, o₃, ...` be any optimal chain, both written in increasing order.

**Claim:** for every `k`, `right(g_k) <= right(o_k)`.

*Induction on k:*
- **Base (k=1):** greedy picks the globally smallest right endpoint, so `right(g₁) <= right(o₁)`. ✓
- **Step:** assume `right(g_k) <= right(o_k)`. Since `O` is valid, `left(o_{k+1}) > right(o_k) >= right(g_k)`. So `o_{k+1}` was a *legal candidate* for greedy at step `k+1`. Greedy scans in increasing right-endpoint order and takes the first legal one, so whatever it took satisfies `right(g_{k+1}) <= right(o_{k+1})`. ✓

**Conclusion:** greedy never falls behind. If `O` had more elements than `G`, then `o_{|G|+1}` would have `left(o_{|G|+1}) > right(o_{|G|}) >= right(g_{|G|})`, meaning greedy still had a legal pair available and would have taken it. Contradiction. So `|G| = |O|`. ∎

### Final Code (the 94% submission)

```java
class Solution {
    public int findLongestChain(int[][] pairs) {
        Arrays.sort(pairs, (a, b) -> Integer.compare(a[1], b[1]));

        int prevEnd = pairs[0][1];
        int count = 1;

        for (int i = 1; i < pairs.length; i++) {
            if (pairs[i][0] > prevEnd) {   // STRICT >
                count++;
                prevEnd = pairs[i][1];
            }
        }

        return count;
    }
}
```

### Complexity
| | |
|---|---|
| Time | **O(n log n)** — sort dominates; the scan is O(n) |
| Space | **O(1)** auxiliary by convention. *Honest answer:* TimSort on an object array allocates an O(n) temp buffer, so it's O(n) if you count the sort's internal memory. Say "O(1) extra, ignoring sort space" in an interview — that shows you know the difference. |

### Two traps

1. **`>` vs `>=`.** The problem says `b < c`, strictly. `[[1,2],[2,3]]` → answer is **1**, not 2, because `2 < 2` is false. Using `>=` silently gives the wrong answer on touching intervals.
2. **`pairs[0]`.** Safe here because `n >= 1`. If the constraint allowed `n == 0`, this NPEs/AIOOBEs. A defensive version initializes `prevEnd = Integer.MIN_VALUE, count = 0` and loops from `i = 0`.

---

## 5. The Optimization Journey (why 42% → 94%)

The algorithm never changed. Only the **comparator** changed. Both versions are O(n log n); the difference is entirely constant factor.

### ❌ Version 1 — 42%
```java
Arrays.sort(pairs, Comparator.comparingDouble(pair -> pair[1]));
```

What this actually costs per comparison:
1. **int → double widening** on both operands, every single comparison. Floating-point conversion where integer comparison would do.
2. **An extra layer of indirection.** `comparingDouble` wraps your lambda in a `ToDoubleFunction`, then wraps *that* in a generated `Comparator`. Each comparison is a call through the `Comparator` interface, which then calls through the `ToDoubleFunction` interface. Two megamorphic-prone virtual dispatches instead of one.
3. **Worse JIT inlining.** The nested functional-interface layers make it harder for HotSpot to inline the comparison body into TimSort's hot merge loop.

None of these is catastrophic alone. But with `n = 1000`, the sort makes ~10,000 comparisons per test case, over many test cases. The overhead compounds.

### ✅ Version 2 — 94%
```java
Arrays.sort(pairs, (a, b) -> Integer.compare(a[1], b[1]));
```
Pure integer arithmetic, one lambda, one interface call, easily inlined.

### ⚠️ Version 3 — the "faster" one you shouldn't default to
```java
Arrays.sort(pairs, (a, b) -> a[1] - b[1]);
```
Marginally faster still (subtraction vs. a branch). **But it overflows** when `a[1] - b[1]` exceeds `Integer.MAX_VALUE` — e.g. `a[1] = 2_000_000_000, b[1] = -2_000_000_000` gives a negative result, inverting the comparison and corrupting the sort.

Here the constraints cap values at ±1000, so it's provably safe. **Still don't build the habit.** The subtraction comparator is a classic interview probe — interviewers deliberately ask "what's wrong with this comparator?" `Integer.compare` costs essentially nothing and is always correct.

### Bonus context: why the sort is slow at all

`int[][]` is an **array of objects** (each row is an `int[]` object reference). Java dispatches object-array sorts to **TimSort**, which:
- allocates an O(n) temporary merge buffer,
- performs comparisons through an interface call,
- has significant setup overhead for small inputs.

Primitive arrays (`int[]`) get **dual-pivot quicksort** instead — no allocation, no virtual dispatch, dramatically faster. You cannot access that path while sorting `int[][]` directly. This observation is what leads to the next section.

### Reality check on LeetCode percentiles
Runtime percentile on LeetCode is **noisy** — the same byte-identical submission routinely swings 20–30 percentage points between runs depending on server load and JIT warmup. Chase algorithmic complexity and clean code; treat the percentile as a rough signal, not a score.

---

## 6. Approach 4 — Beyond Sorting: O(n + range) Counting Sort

Since `-1000 <= right_i <= 1000`, there are only **2001 possible right-endpoint values**. You don't need a comparison sort at all — you can bucket by right endpoint and sweep the number line in order.

### The subtlety
Multiple pairs can share the same right endpoint. Among those, which do you keep? **The one with the largest left endpoint** — it has the best chance of clearing `prevEnd`, and whichever you pick, the resulting `prevEnd` is identical (it's the shared right value). So there's never a downside to keeping the max-left one.

### Code
```java
class Solution {
    private static final int OFFSET = 1000;
    private static final int RANGE  = 2001;   // values -1000 .. 1000

    public int findLongestChain(int[][] pairs) {
        int[] maxLeft = new int[RANGE];
        Arrays.fill(maxLeft, Integer.MIN_VALUE);   // MIN_VALUE = bucket empty

        for (int[] p : pairs) {
            int idx = p[1] + OFFSET;
            if (p[0] > maxLeft[idx]) maxLeft[idx] = p[0];
        }

        int count = 0;
        int prevEnd = Integer.MIN_VALUE;

        for (int r = 0; r < RANGE; r++) {
            if (maxLeft[r] == Integer.MIN_VALUE) continue;   // no pair ends here
            if (maxLeft[r] > prevEnd) {
                count++;
                prevEnd = r - OFFSET;
            }
        }

        return count;
    }
}
```

### Complexity
| | |
|---|---|
| Time | **O(n + R)** where R = 2001. Effectively **O(n)** — no `log n` factor, no comparator calls at all. |
| Space | **O(R)** = one fixed 2001-int array ≈ 8 KB |

### When to actually use this
This is a **constraint-exploiting** solution. It's strictly better here, but it's *worse engineering* in general: it hard-codes `±1000`, so it breaks the moment the constraints change. Mention it in an interview as "if the value range is bounded, we can counting-sort and drop to linear time" — that's the insight they want. Then write the greedy, because it's the robust answer.

---

## 7. Dry Runs

### Dry Run A — Main example (greedy, sorted by right)

**Input:** `[[-6,9], [1,6], [8,10], [-1,4], [-6,-2], [-9,8], [-5,3], [0,3]]`

**Step 1 — sort by right endpoint:**

| # | pair | right |
|---|------|-------|
| 0 | `[-6, -2]` | -2 |
| 1 | `[-5, 3]`  | 3 |
| 2 | `[0, 3]`   | 3 |
| 3 | `[-1, 4]`  | 4 |
| 4 | `[1, 6]`   | 6 |
| 5 | `[-9, 8]`  | 8 |
| 6 | `[-6, 9]`  | 9 |
| 7 | `[8, 10]`  | 10 |

**Step 2 — initialize:** `prevEnd = -2`, `count = 1` (pair `[-6,-2]` is taken)

**Step 3 — scan:**

| i | pair | check `left > prevEnd` | action | count | prevEnd |
|---|------|------------------------|--------|-------|---------|
| — | `[-6,-2]` | (seed) | TAKE | 1 | -2 |
| 1 | `[-5, 3]` | `-5 > -2` → **false** | skip | 1 | -2 |
| 2 | `[0, 3]`  | `0 > -2` → **true**   | **TAKE** | 2 | 3 |
| 3 | `[-1, 4]` | `-1 > 3` → **false**  | skip | 2 | 3 |
| 4 | `[1, 6]`  | `1 > 3` → **false**   | skip | 2 | 3 |
| 5 | `[-9, 8]` | `-9 > 3` → **false**  | skip | 2 | 3 |
| 6 | `[-6, 9]` | `-6 > 3` → **false**  | skip | 2 | 3 |
| 7 | `[8, 10]` | `8 > 3` → **true**    | **TAKE** | 3 | 10 |

**Output: 3** — chain `[-6,-2] → [0,3] → [8,10]`. ✓

Note how at `i = 1` and `i = 2` the two pairs tie on right endpoint (both 3). Greedy skips `[-5,3]` and takes `[0,3]`. Either would have set `prevEnd = 3`; only `[0,3]` clears the gate. This is exactly the "keep max left among ties" logic the counting-sort version makes explicit.

---

### Dry Run B — The strict-inequality trap

**Input:** `[[1,2], [2,3], [3,4]]`

Already sorted by right. `prevEnd = 2`, `count = 1`.

| i | pair | check | action | count | prevEnd |
|---|------|-------|--------|-------|---------|
| — | `[1,2]` | (seed) | TAKE | 1 | 2 |
| 1 | `[2,3]` | `2 > 2` → **false** | skip | 1 | 2 |
| 2 | `[3,4]` | `3 > 2` → **true**  | **TAKE** | 2 | 4 |

**Output: 2** — chain `[1,2] → [3,4]`.

If you had written `>=`, you'd take all three and return **3**, which is wrong. `[1,2]` cannot be followed by `[2,3]` because `2 < 2` is false.

---

### Dry Run C — DP approach on the same input as B

**Input sorted by LEFT:** `[1,2], [2,3], [3,4]` → `dp = [1, 1, 1]`

| i | j | `pairs[j][1] < pairs[i][0]`? | dp update |
|---|---|------------------------------|-----------|
| 1 | 0 | `2 < 2` → false | dp[1] stays 1 |
| 2 | 0 | `2 < 3` → **true**  | dp[2] = max(1, dp[0]+1) = **2** |
| 2 | 1 | `3 < 3` → false | dp[2] stays 2 |

`dp = [1, 1, 2]` → **answer 2**. ✓ Matches the greedy.

---

### Dry Run D — Counting sort on Example A

**Bucket fill** (`maxLeft[right + 1000]`, only non-empty shown):

| right | pairs ending here | maxLeft stored |
|-------|-------------------|----------------|
| -2 | `[-6,-2]` | -6 |
| 3  | `[-5,3]`, `[0,3]` | **0** ← max of -5 and 0 |
| 4  | `[-1,4]` | -1 |
| 6  | `[1,6]`  | 1 |
| 8  | `[-9,8]` | -9 |
| 9  | `[-6,9]` | -6 |
| 10 | `[8,10]` | 8 |

**Sweep** `r` from -1000 upward, skipping empty buckets. `prevEnd = MIN_VALUE`, `count = 0`.

| r | maxLeft | `maxLeft > prevEnd`? | count | prevEnd |
|---|---------|----------------------|-------|---------|
| -2 | -6 | `-6 > MIN` → **true**  | 1 | -2 |
| 3  | 0  | `0 > -2` → **true**    | 2 | 3 |
| 4  | -1 | `-1 > 3` → false       | 2 | 3 |
| 6  | 1  | `1 > 3` → false        | 2 | 3 |
| 8  | -9 | false                  | 2 | 3 |
| 9  | -6 | false                  | 2 | 3 |
| 10 | 8  | `8 > 3` → **true**     | 3 | 10 |

**Output: 3.** ✓ Matches.

---

## 8. Complexity Summary

| Approach | Time | Space | Verdict |
|----------|------|-------|---------|
| Brute force (all subsets) | O(2ⁿ · n log n) | O(n) | TLE. Conceptual only. |
| DP / LIS-style, sort by left | O(n²) | O(n) | Passes. Slow. Generalizes to weighted variants. |
| **Greedy, sort by right** | **O(n log n)** | **O(1)** aux | ⭐ The answer. Write this in an interview. |
| Greedy + counting sort | O(n + R), R = 2001 | O(R) | Fastest here. Constraint-dependent. Mention as a follow-up. |

### Complexity intuition to internalize

- **Why O(n log n) is the natural floor for the general greedy:** you must know the intervals in right-endpoint order, and ordering arbitrary values requires a comparison sort. The scan afterward is free (O(n), one pass, O(1) state).
- **Why counting sort escapes that floor:** the `log n` in comparison sorting comes from the information-theoretic bound on distinguishing `n!` orderings *when values are unbounded*. With a fixed range R, you can index directly instead of comparing, and the bound doesn't apply. This is the same reason radix/counting sort beats quicksort on bounded integers.
- **Why greedy beats DP here:** the DP explores "what's the best chain ending at `i`" for every `i`. The exchange argument proves you never need that — the earliest-finishing valid choice is always part of *some* optimal solution, so there's nothing to explore. Greedy replaces an O(n²) search with an O(1) decision per element.

---

## 9. Pattern Recognition — Where else this shows up

The "sort by right endpoint, greedily take non-overlapping" template solves an entire family:

| Problem | Variation |
|---------|-----------|
| **LC 435** Non-overlapping Intervals | Same greedy. Answer = `n - maxChain`. Uses `>=` since touching is allowed. |
| **LC 452** Min Arrows to Burst Balloons | Same greedy on "end" coordinate; count groups instead of length. Touching counts as overlapping. |
| **LC 1288** Remove Covered Intervals | Sort by left asc, right desc; sweep tracking max right. |
| **LC 56** Merge Intervals | Sort by **left**, merge forward. (You've done this one.) |
| **LC 253** Meeting Rooms II | Sort + min-heap of end times. Different goal: max concurrency. |
| Classic Activity Selection | The textbook ancestor of all of these. |
| **LC 354** Russian Doll Envelopes | LIS-style, *not* greedy — 2D constraint breaks the exchange argument. |

### Decision rule to memorize

> **Sort by RIGHT endpoint** when you want to *maximize the count* of non-overlapping intervals.
> **Sort by LEFT endpoint** when you want to *merge* or *sweep* intervals.
> **Fall back to LIS DP** when there are weights, or when the ordering constraint is multi-dimensional.

### The `>` vs `>=` rule

| Problem says | Use |
|---|---|
| "b **<** c" — strictly before, touching forbidden (LC 646) | `if (left > prevEnd)` |
| Touching is allowed, only true overlap forbidden (LC 435) | `if (left >= prevEnd)` |

Read the problem statement's inequality symbol literally. Getting this backwards is the single most common bug in this family.

---

## 10. Interview Talk Track (60-second version)

> "These are intervals, and I want the largest set that don't overlap — that's classic activity selection.
>
> The brute force is checking all `2ⁿ` subsets, clearly infeasible. A DP approach would sort by left endpoint and run LIS with a custom predicate — `dp[i] = 1 + max(dp[j])` where `pairs[j][1] < pairs[i][0]` — that's O(n²) and it works, but it's doing unnecessary work.
>
> The greedy insight is: sort by *right* endpoint and always take the earliest-finishing compatible interval. That's optimal by an exchange argument — greedy's k-th pick always ends no later than any optimal solution's k-th pick, so greedy is never behind and can never run out first.
>
> That's O(n log n) for the sort, O(1) extra space, one linear pass. Note the comparison is strict — the problem requires `b < c`, so touching intervals don't chain.
>
> One follow-up: the constraints bound values to ±1000, so I could bucket by right endpoint and sweep, keeping the max left endpoint per bucket. That drops it to O(n + R) linear time at the cost of a fixed 2001-element array — though I'd only ship that if the bounded range is a guarantee I can rely on."

---

## 11. Self-Test Checklist

Before you consider this problem "known", make sure you can answer these from memory:

- [ ] Why does sorting by **left** endpoint break the greedy? (Give the counterexample.)
- [ ] State the exchange argument proving the right-endpoint greedy is optimal.
- [ ] Why is the comparison `>` and not `>=`? What input distinguishes them?
- [ ] Write the O(n²) DP recurrence without looking.
- [ ] Why does `Comparator.comparingDouble` cost more than `Integer.compare`?
- [ ] Why is `(a, b) -> a[1] - b[1]` dangerous in general but safe here?
- [ ] Why does Java use TimSort for `int[][]` but quicksort for `int[]`?
- [ ] How do the counting-sort buckets handle two pairs with the same right endpoint?
- [ ] How would LC 435's answer be derived from this exact greedy?
- [ ] If each pair had a **weight** and you wanted max total weight, which approach survives?

---

## 12. Final Submission

```java
class Solution {
    public int findLongestChain(int[][] pairs) {
        Arrays.sort(pairs, (a, b) -> Integer.compare(a[1], b[1]));

        int prevEnd = pairs[0][1];
        int count = 1;

        for (int i = 1; i < pairs.length; i++) {
            if (pairs[i][0] > prevEnd) {
                count++;
                prevEnd = pairs[i][1];
            }
        }

        return count;
    }
}
```

**Time:** O(n log n) · **Space:** O(1) auxiliary · **Beats ~94%**