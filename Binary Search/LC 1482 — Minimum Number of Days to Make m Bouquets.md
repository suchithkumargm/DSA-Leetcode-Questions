# LC 1482 — Minimum Number of Days to Make m Bouquets

**Difficulty:** Medium
**Pattern:** Binary Search on the Answer (+ greedy feasibility check)
**Language:** Java
**Sheets:** Striver A2Z (Binary Search on Answers), NeetCode 150 adjacent, Love Babbar
**Link:** https://leetcode.com/problems/minimum-number-of-days-to-make-m-bouquets/

---

## 1. Problem Statement

Given an integer array `bloomDay`, and two integers `m` and `k`.

- Flower `i` blooms on day `bloomDay[i]` and stays bloomed forever after.
- To make **one bouquet** you need `k` **adjacent** bloomed flowers.
- Each flower can be used in **at most one** bouquet.

Return the minimum number of days to wait so that `m` bouquets can be made. If impossible, return `-1`.

### Constraints (these drive every design decision)

| Quantity | Range | Consequence |
|---|---|---|
| `bloomDay.length` = n | 1 … 10⁵ | O(n) scan is fine, O(n²) is not |
| `bloomDay[i]` | 1 … 10⁹ | **Cannot iterate day by day** → TLE |
| `m` | 1 … 10⁶ | `m * k` overflows `int` |
| `k` | 1 … n | — |

**The single most important constraint:** the answer lives in `[1, 10⁹]` but `n ≤ 10⁵`. Any solution that walks the day axis linearly is dead on arrival.

---

## 2. The Two Independent Sub-Problems

Recognising this split is what makes the problem tractable. Do not try to solve both at once.

```
┌─────────────────────────────────────────────┐
│  Q1: Given a FIXED day d, can I make m      │  → greedy scan, O(n)
│      bouquets?           canMake(d)         │
├─────────────────────────────────────────────┤
│  Q2: What is the SMALLEST d where           │  → binary search, O(log(max))
│      canMake(d) is true?                    │
└─────────────────────────────────────────────┘
```

Total: **O(n · log(max bloomDay))**

---

## 3. The Journey — Every Version and What Broke

### v0 — First attempt (BROKEN: infinite loop + state corruption)

```java
for(day = 1; day <= maxDays; day++){
    markBloomDays(bloomDay, day);        // writes -1 into input array

    while(m >= 0){                       // <-- BUG
        int adjacentCount = 0;
        for(int i = 0; i < bloomDay.length; i++){
            if(bloomDay[i] == -1) adjacentCount++;
            else adjacentCount = 0;
            if(adjacentCount >= k) m--;
        }
    }
    if(m == 0) return day;
}
```

**Bugs found:**

1. **The `while(m >= 0)` never terminates.** If the day's scan finds zero bouquets, `m` is unchanged, the condition is still true, and the identical scan runs forever. The scan is deterministic — re-running it cannot produce a different result.
2. **`m` is both the target and the counter.** It is decremented, so the original target is destroyed and day 2 starts from a corrupted target.
3. **`markBloomDays` mutates the caller's array.** After day 1, `findMaxDays` would return `-1` if called again.
4. **`if(i == length - 1) break;`** — dead code. The `for` condition already stops there.

> **Lesson:** a loop whose body cannot change its own condition is an infinite loop. Check this mechanically before running.

---

### v1 — Removed the `while` (still BROKEN: double-counting)

```java
int adjacentCount = 0;
for(int i = 0; i < bloomDay.length; i++){
    if(bloomDay[i] == -1) adjacentCount++;
    else adjacentCount = 0;
    if(adjacentCount >= k) m--;      // <-- fires on EVERY index past k
}
```

**Dry run — `bloomDay = [1,1,1,1]`, `m = 2`, `k = 2`, day 1 (all bloomed):**

| i | adjacentCount | `>= k`? | m |
|---|---|---|---|
| 0 | 1 | no | 2 |
| 1 | 2 | **yes** | 1 |
| 2 | 3 | **yes** | 0 |
| 3 | 4 | **yes** | -1 |

Three bouquets counted from four flowers with `k = 2`. But flowers 0 and 1 went into bouquet #1 — they cannot also be in bouquet #2. **Flowers were reused.**

Also note `m` ended at `-1`, so `if(m == 0)` would miss it. Symptom of the same root cause.

---

### v2 — Greedy consume (the fix)

```java
if(adjacentCount == k){
    m--;
    adjacentCount = 0;     // <-- consume the k flowers
}
```

Resetting to 0 says "those `k` flowers are spent; start a fresh run from the next flower."

**Same dry run, fixed:**

| i | adjacentCount | `== k`? | m |
|---|---|---|---|
| 0 | 1 | no | 2 |
| 1 | 2 → reset 0 | **yes** | 1 |
| 2 | 1 | no | 1 |
| 3 | 2 → reset 0 | **yes** | **0 → return true** |

Bouquets = flowers {0,1} and {2,3}. Correct.

> Because the counter resets on every hit, it can never exceed `k`. So `== k` and `>= k` are equivalent here — prefer `== k`, it states the intent.

**No `boolean[] used` array is needed.** "Used" is implied by the left-to-right scan position: once consumed, the counter resets and those indices are never revisited. This is a common instinct to resist — O(n) extra space for information the scan already encodes.

---

### v3 — Extracted `canMake` (fixed cross-day state leak)

**Dry run of the bug — `bloomDay = [1,10,3,10,2]`, `m = 3`, `k = 1`** (expected answer 3):

| day | flowers bloomed | bouquets found | m before → after | verdict |
|---|---|---|---|---|
| 1 | {0} | 1 | 3 → 2 | not 0, continue |
| 2 | {0,4} | 2 | **2** → 0 | **returns 2 — WRONG** |

On day 2 only two flowers have bloomed; three bouquets are impossible. The day-1 decrement leaked into day 2.

**The invariant that was broken:** each day is a **completely independent question**. "Can I make 3 bouquets by day 2?" does not inherit partial progress from day 1. There is no carry-over — you re-scan from scratch with the full target every time.

The fix is structural: pass `m` **by value** into a helper.

```java
private boolean canMake(int[] bloomDay, int day, int m, int k){
    // m is a local copy — mutating it cannot affect the caller
}
```

Java passes primitives by value, so `m--` inside `canMake` touches only the local copy. The array is passed by reference, which is exactly why the array must not be mutated.

> **Distinction worth internalising:** *within* one day, consumption must persist (v2's reset). *Across* days, nothing persists. Two different scopes, easy to conflate.

---

### v4 — Killed `markBloomDays`, switched to binary search

Two-phase marking (`mark`, then `scan for -1`) only worked because days were visited in increasing order, so `-1`s accumulated monotonically. Binary search probes **out of order** — day 500, then day 250 — which destroys that assumption instantly.

Replace the mutating marker with a stateless predicate:

```java
if(bloomDay[i] <= day)   // bloomed by `day`?
```

One expression, no writes, no ordering dependency, works for any probe order. This is what makes binary search legal.

> A mutation-free checker is not merely tidier — it is a **precondition** for binary search. Flagging this in an interview shows you understand *why* the refactor is required.

**Trap hit here:** `canMake` was left testing `bloomDay[i] == -1` after `markBloomDays` was deleted. Nothing ever writes `-1`, so `adjacentCount` stayed 0 forever, `canMake` always returned false, and the function returned `max(bloomDay)` for every input. Silent wrong answer, not a crash.

---

### v5 — Final micro-optimisations

- `low = min(bloomDay)` instead of `1` — days below the minimum bloom day trivially fail.
- min and max in **one pass** instead of two.
- Moved `adjacentCount == k` **inside** the `<= day` branch so it isn't evaluated for unbloomed flowers.

None of these change the complexity. They shaved constants.

---

## 4. Final Solution

```java
class Solution {
    public int minDays(int[] bloomDay, int m, int k) {
        long needed = (long) m * k;               // MUST cast an operand
        if(needed > bloomDay.length){
            return -1;
        }

        int low = Integer.MAX_VALUE;
        int high = Integer.MIN_VALUE;
        for(int d : bloomDay){                    // min & max in one pass
            if(d < low)  low = d;
            if(d > high) high = d;
        }

        int ans = high;
        while(low <= high){
            int mid = low + (high - low) / 2;     // overflow-safe midpoint
            if(canMake(bloomDay, mid, m, k)){
                ans = mid;
                high = mid - 1;                   // try to do better
            }else{
                low = mid + 1;                    // need more days
            }
        }
        return ans;
    }

    private boolean canMake(int[] bloomDay, int day, int m, int k){
        int adjacentCount = 0;
        for(int i = 0; i < bloomDay.length; i++){
            if(bloomDay[i] <= day){
                adjacentCount++;
                if(adjacentCount == k){
                    if(--m == 0) return true;     // early exit
                    adjacentCount = 0;            // consume the k flowers
                }
            }else{
                adjacentCount = 0;                // run broken
            }
        }
        return false;
    }
}
```

**Result: beats 97%.** (Caveat below in §9 — the percentile is mostly noise.)

---

## 5. Why Binary Search Is Valid — The Monotonicity Argument

This is the part interviewers actually probe. Memorise the reasoning, not the code.

**Claim:** if `canMake(d)` is true, then `canMake(d + 1)` is true.

**Proof:** a flower bloomed by day `d` is still bloomed on day `d + 1` (flowers never un-bloom). So the set of bloomed flowers only **grows** with `d`. Any valid selection of bouquets on day `d` remains valid on day `d + 1`. ∎

So the predicate over the day axis has the shape:

```
day:      1  2  3  4  5  6  7  8  9 ...
canMake:  F  F  F  F  T  T  T  T  T ...
                     ↑
                  answer
```

It can **never** look like `F T F T`. That shape — a sorted boolean array — is what binary search needs. Searching for the **first `T`** is the "lower bound" template:

```java
if(predicate(mid)){ ans = mid; high = mid - 1; }   // record & shrink right
else              { low  = mid + 1;            }   // move right
```

> **Generalise this:** any problem asking for "minimum X such that P(X) holds", where P is monotone in X, is a binary-search-on-answer problem. The only real work is writing P.

---

## 6. Why the Greedy `canMake` Is Optimal

**Claim:** for a maximal run of `L` consecutive bloomed flowers, the max bouquets obtainable is `⌊L / k⌋`, and taking the leftmost `k` first never costs anything.

**Proof sketch (exchange argument):**

- *Upper bound:* each bouquet consumes `k` flowers from the run, and bouquets cannot span across an unbloomed flower. So at most `⌊L / k⌋`.
- *Achievable:* take flowers `[0, k)`, then `[k, 2k)`, etc. — this yields exactly `⌊L / k⌋`.
- *Leftmost is safe:* suppose an optimal solution's first bouquet starts at index `j > 0`. Shifting it left to start at 0 keeps it within the run and frees flowers at the right end. This can only help later bouquets. So no optimal solution is lost by going leftmost-first.

Runs are independent (separated by unbloomed flowers), so summing `⌊Lᵢ / k⌋` over all runs is the global maximum. The single-pass counter with reset computes exactly this sum, with early exit once `m` is reached.

---

## 7. Complexity Analysis

### Final solution

| | Complexity | Intuition |
|---|---|---|
| **Time** | **O(n · log(max − min))** | `canMake` is one O(n) pass; binary search halves a range of size up to 10⁹, so ~30 probes. 10⁵ × 30 = 3×10⁶ operations. |
| **Space** | **O(1)** | Only scalars. Input is read, never copied or mutated. |

### Evolution of complexity across versions

| Version | Time | Why |
|---|---|---|
| v0 | ∞ | Infinite `while` loop |
| v1–v3 (linear day scan) | O(n · max) | Up to 10⁵ × 10⁹ = 10¹⁴ → **TLE** |
| v4–v5 (binary search) | O(n · log max) | ~3×10⁶ → passes comfortably |

The jump from `max` to `log max` is a factor of ~3×10⁷. That is the entire difference between TLE and 97%.

---

## 8. Full Dry Run of the Final Solution

**Input:** `bloomDay = [1,10,3,10,2]`, `m = 3`, `k = 1`  → **expected 3**

**Step 0 — impossibility check:** `needed = 3 × 1 = 3`, `n = 5`. `3 > 5` false → continue.

**Step 1 — range:** `low = 1`, `high = 10`, `ans = 10`.

**Step 2 — binary search:**

| iter | low | high | mid | bloomed mask (`<= mid`) | bouquets | canMake | action |
|---|---|---|---|---|---|---|---|
| 1 | 1 | 10 | 5 | `T F T F T` | 3 | **true** | `ans=5`, `high=4` |
| 2 | 1 | 4 | 2 | `T F F F T` | 2 | false | `low=3` |
| 3 | 3 | 4 | 3 | `T F T F T` | 3 | **true** | `ans=3`, `high=2` |
| — | 3 | 2 | — | — | — | — | `low > high`, exit |

**Return `ans = 3`.** ✓

**Step 3 — inside `canMake([1,10,3,10,2], day=3, m=3, k=1)`:**

| i | bloomDay[i] | `<= 3`? | adjacentCount | `== k(1)`? | m |
|---|---|---|---|---|---|
| 0 | 1 | yes | 1 | **yes** → reset 0 | 3 → 2 |
| 1 | 10 | no | 0 | — | 2 |
| 2 | 3 | yes | 1 | **yes** → reset 0 | 2 → 1 |
| 3 | 10 | no | 0 | — | 1 |
| 4 | 2 | yes | 1 | **yes** | 1 → **0 → return true** |

**Contrast — `canMake(day=2)`:** bloomed mask is `T F F F T`, giving only 2 bouquets. `m` ends at 1, returns false. Correct: on day 2 only flowers 0 and 4 exist.

### Second dry run — the `k > 1` adjacency case

**Input:** `bloomDay = [1,10,2,9,3,8,4,7]`, `m = 4`, `k = 2` → **expected -1**

`needed = 4 × 2 = 8`, `n = 8`. `8 > 8` false → proceeds to binary search.

At `day = 10` (everything bloomed) there is one run of length 8: `⌊8/2⌋ = 4` bouquets. So `canMake(10)` is true and the answer is 10, not -1.

Now change to `m = 5, k = 2`: `needed = 10 > 8` → returns `-1` immediately. ✓

### Third dry run — `ans = high` is always safe

Why can `ans` be initialised to `high` without ever being wrong?

If we got past the `-1` guard then `m·k ≤ n`. At `day = max(bloomDay)` every flower is bloomed — a single run of length `n` — yielding `⌊n/k⌋ ≥ m` bouquets (since `n ≥ m·k`). So `canMake(high)` is **guaranteed true**, and the search always finds *some* answer. `ans = high` is a valid fallback that in practice is always overwritten.

---

## 9. Bugs & Traps Catalogue

### Trap 1 — `int` overflow in the impossibility check ⚠️ HIGH VALUE

```java
long needed = m * k;          // WRONG
long needed = (long) m * k;   // RIGHT
```

**Why the first fails:** Java evaluates the right-hand side **before** assignment. Both operands are `int`, so it performs 32-bit multiplication, **wraps**, and only then widens the already-corrupted result to `long`. Numeric promotion happens **per-operation**, not per-assignment. The cast must sit on an **operand**.

**Concrete breaking input:** `m = 65536`, `k = 65536`, `n = 70000`.
True product = 2³² = 4,294,967,296. As `int` this wraps to exactly **0**.
`0 > 70000` is false → skips the `-1` return → binary search runs → returns `max(bloomDay)` instead of `-1`.

**Same family of bug:** `long ms = seconds * 1000;` with `int seconds`. Classic Java interview question.

### Trap 2 — Reusing the parameter as a mutable counter

`m` was simultaneously the input target and the running counter. Once `canMake` takes `m` by value this resolves itself, but the general lesson stands: **do not mutate a parameter that represents an input specification.**

### Trap 3 — Mutating the input array

`markBloomDays` wrote `-1` into `bloomDay`. This broke `findMaxDays` on subsequent calls and made the whole algorithm order-dependent. Arrays are passed by reference in Java — the caller sees every write.

### Trap 4 — Deleting a producer but not its consumer

`markBloomDays` was deleted while `canMake` still tested `bloomDay[i] == -1`. No compile error, no crash — just silently wrong answers on every input. **When removing a mutation, grep for everything that read the mutated state.**

### Trap 5 — `>= k` instead of `== k` without the reset

Double-counts flowers. With the reset in place both work, but `== k` documents the "consume exactly k" intent.

### Trap 6 — Unnecessary `boolean[] used`

O(n) space for information already encoded by the scan position.

### Trap 7 — Midpoint overflow

`(low + high) / 2` can overflow when both approach `Integer.MAX_VALUE`. Here `high ≤ 10⁹` and `2 × 10⁹ > 2³¹−1 ≈ 2.147 × 10⁹` — uncomfortably close. Use `low + (high - low) / 2` **always**; it costs nothing.

### Trap 8 — Reading LeetCode percentiles as signal

Identical submissions scored 49%, then 30%, then 97%. The runtime buckets on this problem are a couple of milliseconds wide and dominated by judge load. **Do not optimise against the percentile** — verify the complexity class instead, then stop.

---

## 10. Rejected Alternative — Binary search over sorted distinct days

```java
int[] days = bloomDay.clone();
Arrays.sort(days);

int lo = 0, hi = days.length - 1, ans = days[hi];
while(lo <= hi){
    int mid = lo + (hi - lo) / 2;
    if(canMake(bloomDay, days[mid], m, k)){ ans = days[mid]; hi = mid - 1; }
    else                                   { lo = mid + 1; }
}
return ans;
```

**The valid observation:** the answer is always a value **present in the array** — the answer day must be some flower's bloom day, since nothing changes on days when no flower blooms.

**Probe count:** log(10⁵) ≈ 17 instead of log(10⁹) ≈ 30.

**Why it's rejected:** `Arrays.sort` on 10⁵ ints costs more than the 13 saved `canMake` calls. Net slower, and it adds O(n) space for the clone.

**Interview use:** state the observation to show you understand the answer space, then explicitly say you're rejecting it on cost. Demonstrating a rejected optimisation with reasoning is stronger than not noticing it.

---

## 11. Pattern Card — Binary Search on the Answer

### Recognition signals

- "Minimum / maximum X such that …"
- The answer range is huge, but checking one candidate is cheap (O(n) or O(n log n))
- The feasibility predicate is **monotone** in X
- Brute force would iterate the answer space linearly → TLE

### Template

```java
int low = <smallest plausible answer>;
int high = <largest plausible answer>;
int ans = high;

while(low <= high){
    int mid = low + (high - low) / 2;
    if(feasible(mid)){
        ans = mid;
        high = mid - 1;      // minimising → shrink right
    }else{
        low = mid + 1;
    }
}
return ans;
```

For **maximisation**, flip: on `feasible(mid)` do `ans = mid; low = mid + 1;`.

### Checklist before writing code

1. ☐ What exactly am I binary searching **over**? (Here: the day, not the array index.)
2. ☐ Is the predicate monotone? **State the proof in one sentence.**
3. ☐ What are `low` and `high`? Are both ends actually reachable?
4. ☐ Is `feasible()` mutation-free? (Required — probes are out of order.)
5. ☐ Minimising or maximising? Which branch records `ans`?
6. ☐ Any overflow in the midpoint or in setup arithmetic?

---

## 12. Related Problems (same skeleton)

| Problem | Binary search over | Feasibility predicate |
|---|---|---|
| **LC 875** Koko Eating Bananas | eating speed | hours needed ≤ h |
| **LC 1011** Ship Packages in D Days | ship capacity | days needed ≤ D |
| **LC 410** Split Array Largest Sum | max allowed subarray sum | splits needed ≤ k |
| **LC 1283** Smallest Divisor Given Threshold | divisor | sum of ceilings ≤ threshold |
| **LC 1760** Minimum Limit of Balls in a Bag | max balls per bag | operations needed ≤ maxOps |
| **LC 2064** Minimized Maximum Products per Store | products per store | stores needed ≤ n |
| **LC 774** Minimize Max Distance to Gas Station | distance (float — careful) | stations needed ≤ k |
| **LC 1552** Magnetic Force Between Two Balls | minimum gap (**maximise**) | can place m balls |
| **LC 1539 / 540 / 33** | — | plain binary search variants, worth revisiting |

**Drill order suggestion:** 875 → 1011 → 1283 → 1482 → 1760 → 410 → 1552 (maximisation flip) → 774 (floating point).

Once the monotone-predicate shape is recognised on sight, each of these is ~10 minutes: write the checker, plug in the template.

---

## 13. One-Minute Recall

1. Answer range is 10⁹, so **never** walk the day axis → binary search on the answer.
2. Monotone because **flowers never un-bloom**. Predicate shape is `F…F T…T`.
3. `canMake(day)` = one O(n) pass, counting consecutive `bloomDay[i] <= day`.
4. On hitting `k`: `m--` **and reset the counter to 0** — flowers get consumed.
5. `canMake` must be **mutation-free** (binary search probes out of order) and take `m` **by value** (each day is an independent question).
6. `-1` when `(long) m * k > n`. **Cast the operand, not the destination.**
7. `low + (high - low) / 2`, `ans = mid; high = mid - 1` for minimisation.
8. **O(n log max) time, O(1) space.**

---

## 14. Interview Talking Points

Things to say out loud that earn credit beyond a working solution:

- "The answer range is 10⁹ but n is only 10⁵, so I can't iterate days — that's the signal for binary search on the answer."
- "Monotonicity holds because flowers never un-bloom, so the bloomed set only grows with the day."
- "The checker has to be mutation-free because binary search probes days out of order."
- "The greedy leftmost-first consumption is optimal by an exchange argument — shifting a bouquet left only frees flowers on the right."
- "`m * k` reaches 10¹¹, so I need `(long)` on an operand — promotion is per-operation, not per-assignment."
- "The answer is always a value present in the array, so I *could* binary search the sorted distinct days for ~17 probes instead of 30 — but the sort costs more than the 13 probes saved, so I'll skip it."
- Production tie-in: this is the same structure as capacity planning / rate-limit tuning — a monotone feasibility check plus bisection to find the minimum viable parameter.