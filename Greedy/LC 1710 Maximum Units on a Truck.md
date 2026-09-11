# LC 1710 — Maximum Units on a Truck
**Pattern:** Greedy / Fractional Knapsack
**Difficulty:** Easy (deceptively — the sort key is where people trip)
**Language:** Java
**Final result:** Beats 95%

---

## 0. Why I ended up here

I wanted to solve **Fractional Knapsack** on LeetCode. It isn't there, and it never will be:
the answer is a floating-point value, which makes exact-match judging awkward, and it's a
textbook greedy-proof problem rather than an interview-style puzzle. It lives in the
GfG / CLRS world.

**LC 1710 is the closest thing on LeetCode** — it is fractional knapsack with integral items.
You take whole "piles" greedily and then *partially* fill the truck with the last pile.
That partial step is the fractional step.

Judged versions of the real fractional knapsack, if I want one:
- GeeksforGeeks Practice — "Fractional Knapsack" (canonical)
- Naukri Code360 (ex-Coding Ninjas)
- InterviewBit, HackerEarth

---

## 1. Problem statement

```
boxTypes[i] = [numberOfBoxes_i, numberOfUnitsPerBox_i]
truckSize   = max number of BOXES the truck can carry
```

Return the maximum total **units** loadable.

**Constraints (matter for the optimisation at the end):**
```
1 <= boxTypes.length <= 1000
1 <= numberOfBoxes_i, numberOfUnitsPerBox_i <= 1000
1 <= truckSize <= 10^6
```

**Example**
```
boxTypes = [[1,3],[2,2],[3,1]], truckSize = 4
→ 1 box @3 units, 2 boxes @2 units, 1 box @1 unit
→ 3 + 4 + 1 = 8
```

---

## 2. Mapping it to knapsack — the key insight

| Knapsack term | In this problem |
|---|---|
| Capacity | `truckSize` — measured in **boxes** |
| One item | One **box** (not one box *type*) |
| Weight of an item | **1** — every box costs exactly one slot |
| Value of an item | `boxTypes[i][1]` (units per box) |
| `boxTypes[i][0]` | **Quantity available** — a *cap*, not a weight |

### The single most important line in this doc

> Fractional knapsack sorts by **value / weight**.
> Here **weight is always 1**, so value/weight **collapses to value**.
> → Sort by `unitsPerBox` descending. **No ratio needed.**

The ratio idea isn't wrong, it's *degenerate*. It only earns its keep when items have
**different** weights (a 10-value item costing 5kg vs a 6-value item costing 1kg).

### The trap that cost me three attempts

`boxTypes[i][0]` (quantity) looks like it belongs in the sort key. **It doesn't.**

- "How many of this exist" → caps how much I can take
- "What does one unit cost / give" → decides the ranking

Only the second belongs in the comparator. Putting quantity in the denominator
*penalises a box type for being plentiful*, which is backwards — having lots of a good
box is never a reason to prefer a worse one.

---

## 3. The journey — every version, and why each broke

### v0 — First attempt (BROKEN, 3 separate bugs)

```java
double[] ratio = new double[boxTypes.length];
for (int i = 0; i < boxTypes.length; i++) {
    ratio[i] = (double) boxTypes[i][0] / boxTypes[i][1];   // BUG 2
}
Arrays.sort(ratio);                                         // BUG 1

for (int i = ratio.length - 1; i >= 0; i--) {
    if (boxTypes[i][0] <= capacity) {
        maxUnits += boxTypes[i][0] * boxTypes[i][1];
        capacity -= boxTypes[i][0];
    } else {
        maxUnits += (ratio[i] * boxTypes[i][0] * boxTypes[i][1]);  // BUG 3
        break;
    }
}
```

**Bug 1 — Sorting a key array detached from its data.**
`ratio` was sorted in isolation, then `boxTypes[i]` was indexed with the *sorted* `i`.
After the sort, position `i` in `ratio` has nothing to do with row `i` in `boxTypes`.
The pairing is destroyed silently — no compile error, no exception, just wrong answers.

> **General rule:** whenever the sort key lives in a different array from its data,
> the link breaks on sort. Either sort the rows themselves, or sort an index array.

**Bug 2 — Ratio inverted.** Computed `boxes / unitsPerBox`. That's neither value/weight
nor weight/value. It's meaningless.

**Bug 3 — Mixing units and boxes in the partial branch.**
`ratio[i] * boxes * unitsPerBox` isn't a quantity that means anything.
Also: `maxUnits += <double>` **compiles silently** via the implicit cast in compound
assignment — so a wrong `double` truncates instead of erroring. Dangerous.

---

### v1 — Fixed the pairing with a `double[][]` of `{index, ratio}`

```java
double[][] ratio = new double[boxTypes.length][2];
for (int i = 0; i < boxTypes.length; i++) {
    ratio[i][0] = i;                                          // index
    ratio[i][1] = (double) boxTypes[i][1] / boxTypes[i][0];   // still inverted
}
Arrays.sort(ratio, Comparator.comparingDouble(a -> a[1]));
```

Pairing now survives the sort — **this part was correct.** But:

- Sort key still wrong (`unitsPerBox / numberOfBoxes`)
- **Index-space collision:** the else branch used `ratio[idx][1]`, mixing `idx`
  (a `boxTypes` index) with `ratio` (the *sorted* array). Two different index spaces.

**Failing case:** `[[5,10],[2,5]]`, `truckSize = 2` → key ranks the 5-unit box above
the 10-unit box. Should be 20, gets 10.

> **Realisation:** `boxTypes` is *already* a 2D array of matched pairs.
> Sorting the rows makes the pairing unbreakable by construction.
> The whole `double[][]` scaffold was unnecessary.

---

### v2 — Sort `boxTypes` directly, correct key (still one bug)

```java
Arrays.sort(boxTypes, (a, b) -> a[1] - b[1]);   // ascending, walk backwards

for (int i = boxTypes.length - 1; i >= 0; i--) {
    if (boxTypes[i][0] <= truckSize) {
        maxUnits += boxTypes[i][0] * boxTypes[i][1];
        truckSize -= boxTypes[i][0];
    } else {
        maxUnits += boxTypes[i][0] * boxTypes[i][1];   // IDENTICAL to the if branch
        truckSize = 0;
        break;
    }
}
```

**Bug — the two branches were the same expression.** I reach the `else` precisely
*because* the boxes don't fit, then loaded all of them anyway.

Also note: `Arrays.sort(boxTypes, Integer.compare(a -> a[1]))` — an earlier attempt —
**does not compile**. `Integer.compare` is a static two-int method, not a comparator
factory. Correct forms: `Comparator.comparingInt(a -> a[1])` or `(a, b) -> a[1] - b[1]`.

---

### v3 — Working (beats 59%)

```java
} else {
    maxUnits += (truckSize * boxTypes[i][1]);   // truckSize slots left, not all boxes
    truckSize = 0;
    break;
}
```

In the partial case: `truckSize` slots remain, each carries `boxTypes[i][1]` units.
`boxTypes[i][0]` doesn't appear at all. **This line is the fractional step.**

> Runtime percentile is noisy — resubmitting the same code swings 20–30 points.
> 59% → 95% here was mostly noise plus the cleanup below. Not worth chasing.

---

### v4 — FINAL (beats 95%)

```java
class Solution {
    public int maximumUnits(int[][] boxTypes, int truckSize) {
        Arrays.sort(boxTypes, (a, b) -> b[1] - a[1]);   // descending by unitsPerBox

        int maxUnits = 0;
        for (int i = 0; i < boxTypes.length; i++) {
            int take = Math.min(boxTypes[i][0], truckSize);
            maxUnits += take * boxTypes[i][1];
            truckSize -= take;
            if (truckSize == 0) break;
        }
        return maxUnits;
    }
}
```

**Why the if/else vanished:** the "take all" case is simply the one where
*available* happens to be the smaller number. `Math.min` covers both.

> ### The shape to memorise
> ```java
> int take = Math.min(available, remaining);
> total += take * valuePerUnit;
> remaining -= take;
> ```
> **This is the inner loop of every fractional-knapsack problem.** It reappears in
> resource allocation, scheduling, and bandwidth-distribution problems.

---

## 4. Dry run (final version)

```
boxTypes = [[5,10],[2,5],[4,7],[3,9]]
truckSize = 10
```

**After `Arrays.sort(boxTypes, (a,b) -> b[1] - a[1])`** — descending by units/box:

| idx | boxes | units/box |
|---|---|---|
| 0 | 5 | 10 |
| 1 | 3 | 9 |
| 2 | 4 | 7 |
| 3 | 2 | 5 |

**Loop trace:**

| i | boxes | u/box | truckSize (before) | take = min | units added | maxUnits | truckSize (after) |
|---|---|---|---|---|---|---|---|
| 0 | 5 | 10 | 10 | min(5,10)=**5** | 5×10 = 50 | 50 | 5 |
| 1 | 3 | 9  | 5  | min(3,5)=**3**  | 3×9 = 27  | 77 | 2 |
| 2 | 4 | 7  | 2  | min(4,2)=**2** ← *partial!* | 2×7 = 14 | **91** | 0 → `break` |
| 3 | — | —  | —  | never reached | — | — | — |

**Answer: 91** ✅

**Why this case is a good regression test:** the partial branch actually fires at `i = 2`
(4 boxes available, only 2 slots left). v0–v2 all returned 157 here by loading all 4.

**Second test — `[[1,3],[2,2],[3,1]]`, truckSize = 4:**

| i | boxes | u/box | truckSize | take | added | maxUnits |
|---|---|---|---|---|---|---|
| 0 | 1 | 3 | 4 | 1 | 3 | 3 |
| 1 | 2 | 2 | 3 | 2 | 4 | 7 |
| 2 | 3 | 1 | 1 | **1** (partial) | 1 | **8** |

**Answer: 8** ✅ (buggy versions gave 10)

---

## 5. Complexity

| Version | Time | Space | Note |
|---|---|---|---|
| v1 (`double[][]` pairing) | O(n log n) | **O(n)** | extra pairing array |
| v4 (sort in place) | O(n log n) | **O(1)** aux | mutates input — see below |
| Counting sort variant | **O(n + V)** | O(V), V = 1001 | see §7 |

**Time:** dominated by the sort. The greedy walk is a single O(n) pass —
each box type visited at most once, and we `break` early on a full truck.

**Space:** `Arrays.sort` on an object array (`int[][]` is an array of objects) uses
**TimSort**, which needs O(n) auxiliary space in the worst case — *not* O(1).
Only primitive arrays get the in-place dual-pivot quicksort. Worth knowing for an
interview follow-up; the commonly given "O(1) space" answer is a simplification.

**Caveat:** sorting `boxTypes` in place mutates the caller's input. Fine on LeetCode,
worth flagging in a code review at work.

---

## 6. Why greedy is correct here (exchange argument)

Suppose an optimal solution loads a box of type B while some box of a strictly better
type A (`unitsA > unitsB`) is still available and unloaded.

Swap one B for one A. Truck usage is **unchanged** (both cost exactly 1 slot, since all
weights are 1), and total units **increases** by `unitsA - unitsB > 0`.
Contradiction — the original wasn't optimal.

Therefore always take from the highest-units type first. ∎

> **The proof depends on all weights being equal.** That's exactly why no ratio is
> needed, and exactly why this argument **fails for 0/1 knapsack** — there, items have
> different weights and swapping changes capacity usage, so greedy breaks and you need DP.

---

## 7. Optional micro-optimisation — counting sort

`unitsPerBox ≤ 1000`, so bucket instead of comparison-sort:

```java
public int maximumUnits(int[][] boxTypes, int truckSize) {
    int[] count = new int[1001];              // count[u] = boxes with u units each
    for (int[] b : boxTypes) count[b[1]] += b[0];

    int maxUnits = 0;
    for (int u = 1000; u >= 1 && truckSize > 0; u--) {
        int take = Math.min(count[u], truckSize);
        maxUnits += take * u;
        truckSize -= take;
    }
    return maxUnits;
}
```

**O(n + 1000)** time, O(1001) space. Only beats the comparator sort for large `n`
(n ≤ 1000 here, so log n ≈ 10 — the gain is marginal).

> **Interview framing:** mention this as a follow-up ("since units are bounded by 1000,
> a counting sort makes it linear"), don't write it by default. Showing you noticed the
> constraint is the point.

---

## 8. Bug checklist — my recurring greedy mistakes

- [ ] **Did I sort a key array separately from its data?** → pairing destroyed
- [ ] **Is my sort key actually value/weight?** → if weight is 1, it's just value
- [ ] **Did I put "quantity available" in the sort key?** → it's a cap, never a ranking field
- [ ] **Are my two branches actually different?** → read them side by side
- [ ] **Am I mixing index spaces?** (sorted-array index vs original index)
- [ ] **Am I mixing units?** (boxes vs units; capacity vs value)
- [ ] **`maxUnits += <double>` compiles silently** — compound assignment hides the cast
- [ ] **Removed debug `System.out.println`** before submitting (costs runtime)

---

## 9. Related problems (same greedy skeleton)

| # | Problem | Connection |
|---|---|---|
| **1710** | Maximum Units on a Truck | this one — fractional knapsack, weight = 1 |
| **502** | IPO | greedy + heap under a capital constraint |
| **1029** | Two City Scheduling | sort by cost *difference*, same "which first" reasoning |
| **871** | Min Number of Refueling Stops | greedy with deferred choice (heap) |
| **455** | Assign Cookies | two-pointer greedy after sorting both sides |
| **0/1 Knapsack** | (GfG / CSES) | **contrast:** unequal weights → greedy fails, needs DP |

---

## 10. Reference — true Fractional Knapsack in Java

For when weights genuinely differ. Note the `double` return and the real ratio sort.

```java
static double fractionalKnapsack(int capacity, int[] values, int[] weights) {
    int n = values.length;
    Integer[] idx = new Integer[n];
    for (int i = 0; i < n; i++) idx[i] = i;

    // descending by value/weight — the ratio is NOT degenerate here
    Arrays.sort(idx, (a, b) ->
        Double.compare((double) values[b] / weights[b],
                       (double) values[a] / weights[a]));

    double total = 0;
    int remaining = capacity;
    for (int i : idx) {
        if (remaining == 0) break;
        int take = Math.min(weights[i], remaining);           // same min() shape
        total += (double) values[i] * take / weights[i];      // fractional value
        remaining -= take;
    }
    return total;
}
```

**Edge cases to test:** capacity 0 · zero-weight items (division by zero) ·
all items fit · single item larger than capacity.

Note the `Integer[]` index-sort — that's the *other* correct way to keep a key paired
with its data when you can't reorder the source arrays.

---

## 11. One-line takeaway

> **Truck capacity counts boxes, so every box weighs 1, so value/weight = value.
> Sort descending by units-per-box, then `take = min(available, remaining)`.**