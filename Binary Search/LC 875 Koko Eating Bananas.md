# LC 875 — Koko Eating Bananas

**Difficulty:** Medium
**Link:** https://leetcode.com/problems/koko-eating-bananas/
**Pattern:** Binary Search on Answer (Monotonic Predicate)
**Sheets:** NeetCode 150 (Binary Search), Striver A2Z (Binary Search on Answers), Love Babbar 450
**Language:** Java
**Date solved:** Sep 2026

---

## 1. Problem

Koko has `n` piles of bananas. `piles[i]` is the number of bananas in pile `i`.
The guards leave for `h` hours.

Each hour, Koko picks **one pile** and eats up to `k` bananas from it.

- If the pile has `>= k` bananas, she eats exactly `k`.
- If the pile has `< k` bananas, she eats the whole pile and **stops for that hour** — she does not move on to another pile in the same hour.

Return the **minimum integer `k`** such that she can finish all bananas within `h` hours.

**Constraints:**
```
1 <= piles.length <= 10^4
piles.length <= h <= 10^9
1 <= piles[i] <= 10^9
```

### Two constraints that matter a lot

1. **`h >= piles.length`** — guaranteed. So `k = max(piles)` **always works** (one pile per hour, one hour per pile). This gives a guaranteed-valid upper bound and justifies initialising `ans = max`.
2. **`piles[i]` up to 10^9** — the answer range is up to 10^9, so any linear scan over candidate speeds is dead on arrival. Also flags overflow as something to actively reason about.

---

## 2. The Core Insight (the whole problem in one line)

> **Piles are independent. Pile `p` at speed `k` costs exactly `ceil(p / k)` hours.**

Because she can't spill leftover capacity into another pile, there is no interaction between piles. Total hours:

```
hours(k) = Σ ceil(piles[i] / k)
```

Everything else in this problem is just: how do I search for the smallest `k` where `hours(k) <= h`?

### Why the round-robin simulation gives the same number

Useful to know because my first instinct was to simulate:

Each hour touches exactly one **non-empty** pile and removes `min(k, remaining)` from it. Pile `p` therefore needs exactly `ceil(p/k)` touches to empty, no matter when those touches happen. Total hours = total touches = `Σ ceil(p/k)`.

**The order of eating is irrelevant.** Simulation is correct but pure waste — it computes a closed-form value by brute force.

### The monotonic property (this is the interview justification)

Define `canFinish(k) = (hours(k) <= h)`.

If speed `k` works, then **every speed `> k` also works** — eating faster can never take more hours (`ceil(p/k)` is non-increasing in `k`).

So over `k = 1 .. max`, the predicate looks like:

```
k:          1  2  3  4  5  6  ... max
canFinish: F  F  F  T  T  T  ...  T
                    ^
                    first T = answer
```

**Search space sorted by a boolean predicate ⇒ binary search applies.** Say this out loud in an interview before writing code. It's the actual reason the algorithm is valid.

---

## 3. Approach 1 — Round-Robin Simulation + Linear Scan Down (my first attempt)

### Idea
Start at `k = max` (known to work), simulate eating, walk `k` downward until a speed fails, return the last one that worked.

### Code (buggy version I actually wrote first)

```java
class Solution {
    public int minEatingSpeed(int[] piles, int h) {
        int max = Integer.MIN_VALUE;
        for (int i = 0; i < piles.length; i++) max = Math.max(max, piles[i]);

        while (max > 0) {
            int tempHours = h;
            int[] tempArr = Arrays.copyOf(piles, piles.length);

            while (tempHours > 0) {
                for (int i = 0; i < tempArr.length; i++) {
                    tempArr[i] = tempArr[i] - max < 0 ? 0 : tempArr[i] - max;
                    tempHours--;
                }
            }

            for (int i = 0; i < tempArr.length; i++) {
                if (tempArr[i] != 0) return max;   // infeasible
            }
            max--;
        }
        return max;
    }
}
```

### Bug 1 — the inner `for` overshoots the hour budget

**Trace:** `piles = [30,11,23,4,20]`, `h = 6`, `max = 23`

Pass 1:
| i | before | after | tempHours |
|---|--------|-------|-----------|
| 0 | 30 | 7 | 5 |
| 1 | 11 | 0 | 4 |
| 2 | 23 | 0 | 3 |
| 3 | 4  | 0 | 2 |
| 4 | 20 | 0 | 1 |

`tempArr = [7,0,0,0,0]`, `tempHours = 1`. Outer `while(1 > 0)` is true, so we enter another full pass.

Pass 2:
| i | before | after | tempHours |
|---|--------|-------|-----------|
| 0 | 7 | 0 | **0** |
| 1 | 0 | 0 | -1 |
| 2 | 0 | 0 | -2 |
| 3 | 0 | 0 | -3 |
| 4 | 0 | 0 | -4 |

The `for` loop doesn't check `tempHours` — it burns 4 phantom hours.

**Fix:** `if (tempHours <= 0) break;` at the top of the `for` body.

### Bug 2 — charging an hour for an already-empty pile

**Trace:** `piles = [3,6,7,11]`, `h = 8`, `max = 4` (the correct answer)

Pass 1: `[0,2,3,7]`, `tempHours = 4`

Pass 2:
| i | before | after | tempHours |
|---|--------|-------|-----------|
| 0 | **0** | 0 | 3 ← wasted hour |
| 1 | 2 | 0 | 2 |
| 2 | 3 | 0 | 1 |
| 3 | 7 | 3 | 0 |

Ends `[0,0,0,3]` with hours exhausted ⇒ reports `4` as **infeasible**, returns `5`. Wrong.

Sanity check by formula: `ceil(3/4)+ceil(6/4)+ceil(7/4)+ceil(11/4) = 1+2+2+3 = 8 = h` ✓ feasible.

Koko would never sit at an emptied pile. **Fix:** `if (tempArr[i] == 0) continue;`

### Bug 3 — the `continue` creates an infinite loop

**Trace:** `piles = [1]`, `h = 2`, `max = 1`

- Pass 1: pile → 0, `tempHours = 1`
- Pass 2: `tempArr[0] == 0` → `continue`. Loop ends. **`tempHours` unchanged at 1.**
- `while (tempHours > 0)` is still true → spins forever.

Once every pile is empty and hours remain, no progress is possible. Need a progress detector:

```java
while (tempHours > 0) {
    boolean ate = false;
    for (int i = 0; i < tempArr.length; i++) {
        if (tempHours <= 0) break;
        if (tempArr[i] == 0) continue;
        tempArr[i] = Math.max(0, tempArr[i] - max);
        tempHours--;
        ate = true;
    }
    if (!ate) break;   // nothing left to eat; speed is feasible
}
```

> **General lesson:** any `while` loop containing a `continue` needs a guarantee that *some* iteration makes progress. A `continue` that skips the decrement is the classic infinite-loop shape.

### Bug 4 — wrong final return

Falling out of the outer loop means speed `1` was feasible and `max` got decremented to `0`. Returning `max` returns `0`, which isn't a valid speed. **Fix:** `return 1;`

### Bug 5 — `System.out.println(Arrays.toString(tempArr))` inside the loop

Printing an array per candidate speed is itself a TLE, independent of the algorithm. **Always strip debug prints before submitting.**

### Complexity
- Outer scan over candidate speeds: `O(m)` where `m = max(piles)`
- Simulation per candidate: `O(h · n)` in the worst case (up to `h` hour-steps, each a partial array walk)
- **Time: `O(m · h · n)`** — catastrophic. With `m = h = 10^9` this doesn't finish this century.
- **Space: `O(n)`** for the `Arrays.copyOf`.

**Verdict:** correct after 5 fixes, unusable. But worth having written once — the bugs taught more than the solution.

---

## 4. Approach 2 — Closed-Form `ceil` + Linear Scan Up (brute force, clean)

### Idea
Drop the simulation. Use `Σ ceil(p/k)` directly. Scan `k` upward from 1 and return the first feasible one.

Scanning **up** is better than down: the first feasible `k` found *is* the answer, no `ans` tracking and no off-by-one with `++max`.

### Code

```java
class Solution {
    public int minEatingSpeed(int[] piles, int h) {
        int max = 0;
        for (int p : piles) max = Math.max(max, p);

        for (int k = 1; k <= max; k++) {
            if (canFinish(piles, h, k)) return k;
        }
        return max;   // unreachable given h >= piles.length
    }

    private boolean canFinish(int[] piles, int h, int k) {
        int hours = 0;
        for (int p : piles) {
            hours += (p + k - 1) / k;
            if (hours > h) return false;   // early exit
        }
        return true;
    }
}
```

### Bugs I hit in my inline version

**(a) Verdict tested inside the inner loop.**

```java
for (int j = 0; j < piles.length; j++) {
    count += ...;
    if (count > h) break;
    if (count <= h) return k;    // ❌ WRONG
}
```

**Trace:** `piles = [3,6,7,11]`, `h = 8`, `k = 1`. At `j = 0`: `count = 3`. Is `3 > 8`? No. Is `3 <= 8`? Yes → `return 1`. But `k=1` needs 27 hours.

At `j = 0`, `count` is the cost of **one pile only**. Comparing a partial sum against the total budget is meaningless.

> **Feasibility is a property of the whole array.** The success verdict belongs *after* the loop. Only the *failure* check (`count > h`) can live inside, because a partial sum already exceeding `h` proves the full sum will too (all terms are positive).

My patch `if (count <= h && j == piles.length - 1)` works but is convoluted. `return count <= h;` after the loop is the clean form.

**(b) `Math.ceil` on integer division does nothing.**

```java
count += Math.ceil(piles[j] / k);   // ❌ both operands int
```

Java evaluates `piles[j] / k` **first**, with int semantics — truncating. `Math.ceil` then rounds an already-truncated integer.

`piles[j] = 3, k = 4` → `3/4` = `0` → `Math.ceil(0.0)` = `0.0` → a 3-banana pile costs **zero hours**.

Every candidate looks cheaper than reality, so the returned `k` is too small.
Counterexample: `piles = [30,11,23,4,20]`, `h = 6` returns `11`, but `11` truly needs `3+1+3+1+2 = 10 > 6`. Correct answer is `23`.

Fixes:
```java
Math.ceil((double) piles[j] / k)     // works, but see §7 on why to avoid
(piles[j] + k - 1) / k               // preferred: exact integer ceiling
```

### Complexity
- **Time: `O(m · n)`**, `m = max(piles)` up to 10^9, `n` up to 10^4 → 10^13 operations worst case.
- **Space: `O(1)`**

**Verdict:** TLE. Correct, and the right stepping stone — it isolates `canFinish` as the reusable piece.

---

## 5. Approach 3 — Binary Search on the Answer (optimal)

### Idea
`canFinish` is monotonic (`F...F T...T`). Binary search the boundary instead of walking it.

**Search space is speeds, not array indices.** There is no "target value" and no `return mid` on a match — the array being searched is conceptual (`1 .. max`).

### Code — my submitted version

```java
class Solution {
    public int minEatingSpeed(int[] piles, int h) {
        int max = Integer.MIN_VALUE;
        for (int i = 0; i < piles.length; i++) max = Math.max(max, piles[i]);

        int start = 1, end = max;
        int ans = max;                       // max always feasible since h >= n

        while (start <= end) {
            int mid = start + (end - start) / 2;
            int count = 0;
            for (int j = 0; j < piles.length; j++) {
                count += (piles[j] + mid - 1) / mid;
                if (count > h) {
                    start = mid + 1;         // too slow
                    break;
                }
                if (count <= h && j == piles.length - 1) {
                    end = mid - 1;           // feasible, try slower
                    ans = mid;
                }
            }
        }
        return ans;
    }
}
```
**Result: beats ~68%.** Optimal complexity.

### Code — interview-ready refactor (write this one)

```java
class Solution {
    public int minEatingSpeed(int[] piles, int h) {
        int lo = 1, hi = 0;
        for (int p : piles) hi = Math.max(hi, p);

        while (lo < hi) {                    // invariant: hi is always feasible
            int mid = lo + (hi - lo) / 2;
            if (canFinish(piles, h, mid)) hi = mid;
            else                          lo = mid + 1;
        }
        return lo;                            // lo == hi == first feasible speed
    }

    private boolean canFinish(int[] piles, int h, int k) {
        long hours = 0;
        for (int p : piles) {
            hours += (p + k - 1) / k;
            if (hours > h) return false;
        }
        return true;
    }
}
```

Why this is better under interview conditions:
- Separates **"is this speed feasible"** from **"how do I search"**. That separation is the transferable pattern.
- No `ans` variable, no `j == piles.length - 1` condition, no mutation of `lo`/`hi` from inside a nested loop.
- Fewer places to introduce an off-by-one.

### Complexity

| Step | Cost |
|------|------|
| Scan for `max` | `O(n)` |
| Binary search iterations | `O(log m)`, `m = max(piles)` |
| `canFinish` per iteration | `O(n)` |
| **Total time** | **`O(n) + O(n log m)` = `O(n log m)`** |
| **Space** | **`O(1)`** |

Concretely: `n = 10^4`, `m = 10^9` → `log₂(10^9) ≈ 30` → ~300,000 operations. Instant.

**This is the lower bound.** You must read all `n` piles to know the answer (`Ω(n)`), and `log m` is the information-theoretic cost of locating a boundary in a range of size `m`. Nothing below `O(n log m)` exists for this problem.

---

## 6. Dry Runs

### Dry Run A — `piles = [3,6,7,11]`, `h = 8` → expect `4`

`max = 11`, `lo = 1`, `hi = 11`

**Iteration 1:** `mid = 1 + (11-1)/2 = 6`
| pile | `ceil(p/6)` | running |
|------|-------------|---------|
| 3 | 1 | 1 |
| 6 | 1 | 2 |
| 7 | 2 | 4 |
| 11 | 2 | 6 |

`6 <= 8` ✓ feasible → `hi = 6`

**Iteration 2:** `lo=1, hi=6`, `mid = 1 + 5/2 = 3`
| pile | `ceil(p/3)` | running |
|------|-------------|---------|
| 3 | 1 | 1 |
| 6 | 2 | 3 |
| 7 | 3 | 6 |
| 11 | 4 | **10 > 8** → early exit |

✗ infeasible → `lo = 4`

**Iteration 3:** `lo=4, hi=6`, `mid = 4 + 2/2 = 5`
| pile | `ceil(p/5)` | running |
|------|-------------|---------|
| 3 | 1 | 1 |
| 6 | 2 | 3 |
| 7 | 2 | 5 |
| 11 | 3 | 8 |

`8 <= 8` ✓ feasible → `hi = 5`

**Iteration 4:** `lo=4, hi=5`, `mid = 4 + 1/2 = 4`
| pile | `ceil(p/4)` | running |
|------|-------------|---------|
| 3 | 1 | 1 |
| 6 | 2 | 3 |
| 7 | 2 | 5 |
| 11 | 3 | 8 |

`8 <= 8` ✓ feasible → `hi = 4`

`lo == hi == 4` → loop exits → **return 4** ✓

Verify boundary: `k=3` costs 10 hours (fails), `k=4` costs 8 (works). `4` is genuinely minimal.

---

### Dry Run B — `piles = [30,11,23,4,20]`, `h = 6` → expect `23`

`max = 30`, `lo = 1`, `hi = 30`

| Iter | lo | hi | mid | hours(mid) | verdict | update |
|------|----|----|-----|-----------|---------|--------|
| 1 | 1 | 30 | 15 | 2+1+2+1+2 = 8 | ✗ | lo = 16 |
| 2 | 16 | 30 | 23 | 2+1+1+1+1 = 6 | ✓ | hi = 23 |
| 3 | 16 | 23 | 19 | 2+1+2+1+2 = 8 | ✗ | lo = 20 |
| 4 | 20 | 23 | 21 | 2+1+2+1+1 = 7 | ✗ | lo = 22 |
| 5 | 22 | 23 | 22 | 2+1+2+1+1 = 7 | ✗ | lo = 23 |

`lo == hi == 23` → **return 23** ✓

Note how tight the boundary is: `k=22` needs 7 hours, `k=23` needs 6. `ceil(23/22) = 2` but `ceil(23/23) = 1` — a single pile flipping one hour decides the whole answer. **This is why you can't approximate or use floating point loosely here.**

---

### Dry Run C — edge cases

| Input | max | Behaviour | Output |
|-------|-----|-----------|--------|
| `[1]`, `h=1` | 1 | `lo=1, hi=1`, loop never entered | `1` ✓ |
| `[1]`, `h=2` | 1 | `lo=1, hi=1`, loop never entered | `1` ✓ |
| `[1,1,1]`, `h=100` | 1 | `lo=hi=1` | `1` ✓ |
| `[10,10,10]`, `h=3` | 10 | `h == n`, no slack, only `k=10` works | `10` ✓ |
| `[1000000000]`, `h=2` | 10^9 | ~30 iterations | `500000000` ✓ |

The `[1]` cases are exactly where `lo = 0` would have blown up — see §7.

---

## 7. Java Gotchas (the reusable takeaways)

### (a) Integer division happens before `Math.ceil`
```java
Math.ceil(a / b)            // ❌ a, b int → truncation already lost the fraction
Math.ceil((double) a / b)   // ✓ works
(a + b - 1) / b             // ✓ preferred: exact, no floats
```

### (b) Why `(a + b - 1) / b` beats `Math.ceil((double) a / b)`
1. **Precision.** `double` has 53 bits of mantissa. At `a = 10^9` we're fine, but relying on float precision for an exact integer result is a habit that breaks at `long` scale. Interviewers probe this.
2. **Speed.** `Math.ceil` costs an int→double conversion, a library call, and a double→int narrowing, **per pile per binary-search iteration**. Swapping it for integer arithmetic took my submission from **beats 6% → beats 68%**. Same asymptotic complexity, real constant-factor win.

### (c) `lo = 0` is a latent divide-by-zero
With `lo = 0` and `piles = [1], h = 1`: `mid = 0`, then `piles[j] / 0`.

- With `Math.ceil((double) p / 0)` → `Infinity` → narrows to `Integer.MAX_VALUE` → trips `count > h` → **accidentally correct via IEEE 754**.
- With `(p + 0 - 1) / 0` → **`ArithmeticException: / by zero`**.

So the float version *masked* the bug and the integer version exposed it. **Always `lo = 1`** — speed 0 is meaningless, she must eat at least one banana per hour.

> Lesson: a correct-looking submission can be surviving on accidental float semantics. When you change arithmetic, re-check the boundary inputs.

### (d) Overflow — both close, both worth saying out loud
- `p + k - 1` at `p = k = 10^9` → `1,999,999,999`. Under `Integer.MAX_VALUE` (`2,147,483,647`) by only **7%**. Safe, but barely.
- `count` as `int`: the `break` fires the moment `count > h`, so `count <= h <= 10^9` before the next addition, whose term is `<= 10^9`. Max ~`2 × 10^9`... **which exceeds `Integer.MAX_VALUE`.** Use `long hours` to be safe, or keep `int` only with the early exit and state the reasoning. Remove the early exit with `int` and it definitively overflows.

Interview line: *"count stays bounded because of the early exit, but I'll make it `long` so correctness doesn't depend on that."*

### (e) `mid = (lo + hi) / 2` overflows
At `lo, hi ≈ 10^9`, the sum exceeds `int`. **Always `lo + (hi - lo) / 2`.**

### (f) Loop-form / off-by-one pairing
| Loop condition | Feasible branch | Infeasible branch | Return |
|----------------|-----------------|-------------------|--------|
| `while (lo < hi)` | `hi = mid` | `lo = mid + 1` | `lo` |
| `while (lo <= hi)` | `hi = mid - 1`, `ans = mid` | `lo = mid + 1` | `ans` |

**Mixing them is the classic bug.** `while (lo <= hi)` with `hi = mid` (no `-1`) never terminates when `lo == hi` and `mid` is feasible.

---

## 8. Further Constant-Factor Optimizations

Asymptotically nothing is left, but these are legitimate:

### Tighten the lower bound
`k` must be at least `ceil(total / h)` — she can't clear `total` bananas any slower than that. Compute the sum in the same pass as `max`:

```java
long total = 0;
int max = 0;
for (int p : piles) { max = Math.max(max, p); total += p; }
int lo = (int) Math.max(1, (total + h - 1) / h);
int hi = max;
```
`total` **must be `long`** — `10^4 × 10^9 = 10^13`, far past `int`.

On skewed inputs (one huge pile among many small ones) this removes several iterations.

### Early exit
If `h == piles.length`, the answer is `max` — one hour per pile, zero slack. Skip the search.

### Where to stop
LeetCode percentiles swing 20–40% for identical submissions depending on server load. 68% with the optimal algorithm and no wasted work is where this problem lands. Chasing further teaches nothing an interviewer values.

---

## 9. The Reusable Template

Whenever a problem asks for **"minimum X such that some condition holds"** and the condition is monotonic in X:

```java
// 1. Identify the search space [lo, hi] of candidate ANSWERS (not indices)
// 2. Write a feasibility predicate: boolean ok(candidate)
// 3. Confirm monotonicity: ok(x) true ⇒ ok(x+1) true   (or the mirror)
// 4. Binary search the boundary

int lo = /* smallest sensible candidate */;
int hi = /* a candidate guaranteed to work */;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (ok(mid)) hi = mid;      // feasible → shrink toward smaller
    else         lo = mid + 1;  // infeasible → must go larger
}
return lo;
```

**Recognition signals:** "minimum capacity", "minimum speed", "smallest largest sum", "minimise the maximum", "minimum days", and an answer range far larger than the input array.

**For maximisation** ("largest X such that..."), the predicate flips to `T...T F...F` and you search for the last `T`: use `while (lo < hi)` with `mid = lo + (hi - lo + 1) / 2` (upper mid, to avoid an infinite loop), `lo = mid` on feasible, `hi = mid - 1` otherwise.

---

## 10. Related Problems (same pattern)

| Problem | Search space | Predicate |
|---------|--------------|-----------|
| **LC 1011** Ship Packages in D Days | ship capacity `[max(w), sum(w)]` | can ship in `d` days with this capacity? |
| **LC 410** Split Array Largest Sum | max subarray sum `[max, sum]` | can split into `<= k` parts under this cap? |
| **LC 1482** Min Days to Make Bouquets | day `[min(bloom), max(bloom)]` | enough adjacent bloomed flowers by this day? |
| **LC 2064** Minimized Maximum Products per Store | products per store `[1, max(quantities)]` | stores needed `<= n`? |
| **LC 1283** Smallest Divisor Given Threshold | divisor `[1, max(nums)]` | `Σ ceil(nums[i]/d) <= threshold`? **Near-identical to 875** |
| **LC 1552** Magnetic Force Between Balls | min gap (maximisation variant) | can place `m` balls with at least this gap? |
| **LC 1898** Max Removable Characters | removals count | is it still a subsequence? |

**LC 1283 is the same problem with the story removed.** Solve it right after this one to lock the pattern in.

---

## 11. Retrospective — what the journey actually taught

| Stage | Time | Space | Why it failed / worked |
|-------|------|-------|------------------------|
| Round-robin simulation + linear scan down | `O(m · h · n)` | `O(n)` | Simulated something with a closed form. 5 distinct bugs. |
| Closed-form `ceil` + linear scan up | `O(m · n)` | `O(1)` | Removed the simulation. Still walked a 10^9 range. TLE. |
| Binary search + `Math.ceil(double)` | `O(n log m)` | `O(1)` | Optimal complexity, poor constant factor. Beats 6%. |
| Binary search + integer `ceil` | `O(n log m)` | `O(1)` | Beats 68%. Also exposed the `lo = 0` bug the floats had hidden. |

**Transferable lessons:**

1. **Look for the closed form before simulating.** "Each hour eat from one pile" *sounds* sequential; the independence of piles makes it a sum.
2. **Partial sums can't be compared against a total budget** — except to prove failure early. Success verdicts go after the loop.
3. **`Math.ceil(int / int)` is a no-op.** Compute the `ceil` in integers: `(a + b - 1) / b`.
4. **A `continue` inside a `while` needs a progress guarantee**, or it's an infinite loop.
5. **"Minimum X satisfying a monotonic condition" ⇒ binary search the answer, not the array.**
6. **Name the monotonic property out loud.** It's the proof of correctness, and it's what the interviewer is listening for.
7. **Floating point can hide boundary bugs.** The `lo = 0` divide-by-zero only surfaced after switching to integer arithmetic.
8. **Verify the optimal solution exists before micro-tuning.** Going from 6% → 68% was one expression; going past 68% would be noise.