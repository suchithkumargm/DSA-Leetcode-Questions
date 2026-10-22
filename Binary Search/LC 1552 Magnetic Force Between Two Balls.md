# LC 1552 — Magnetic Force Between Two Balls
### *= Striver SDE #65, "Aggressive Cows"*

**Difficulty:** Medium · **Pattern:** Binary search on the answer (maximise the minimum)

---

## 1. Problem

Given `position[]` (basket coordinates) and `m` balls, place all `m` balls in distinct baskets. The
**magnetic force** between two balls is `|x − y|`. Return the **maximum possible minimum** force
between any two balls.

```
position = [1,2,3,4,7],            m = 3  →  3      (place at 1, 4, 7)
position = [5,4,3,2,1,1000000000], m = 2  →  999999999
```

Constraints: `n ≤ 10⁵`, `position[i] ≤ 10⁹`, `2 ≤ m ≤ n`.

---

## 2. The direction flip — read this before anything else

If you've just done **LC 875 (Koko)** or **LC 1482 (Bouquets)**, both **minimise**. This one
**maximises the minimum**, which inverts the search.

```
LC 875 / 1482  — minimise:          LC 1552 — maximise the minimum:
speed:     1   2   3   4   5        gap:       1   2   3   4   5
feasible:  F   F   T   T   T        feasible:  T   T   T   F   F
                   ↑                                   ↑
           answer = FIRST true                 answer = LAST true
   feasible ⇒ try smaller ⇒ hi = mid-1   feasible ⇒ try bigger ⇒ lo = mid+1
```

> **The flip in one sentence:** in a minimise problem a `true` means *"maybe I can do better by
> going smaller"* → discard the right. In a maximise-the-minimum problem a `true` means *"maybe I
> can do better by going bigger"* → discard the left.

Same algorithm, mirrored. Writing this on autopilot from the previous two problems gets it
backwards.

---

## 3. Understanding the problem

The statement is hard to parse. **Abandon it and solve a tiny case by hand.**

### `[1,2,3,4,7]`, m = 3

Place at 1, 4, 7. Pairwise forces: `|1−4|=3`, `|4−7|=3`, `|1−7|=6` → `[3,3,6]` → **min = 3**.

### 🔑 Insight 1 — only the MINIMUM gap is scored

The instinct *"place balls to maximise the force"* is wrong. The score is the **minimum** pairwise
force; every other gap is irrelevant.

On `[1,2,3,4,5,1000000000]`, m = 3, with balls pinned at 1 and 1000000000:

| middle ball at | gaps | **min** |
|---|---|---|
| 2 | 1, 999999998 | **1** |
| 3 | 2, 999999997 | **2** |
| 5 | 4, 999999995 | **4** ✅ |

The enormous right-hand gap **never matters** — it's never the minimum. The entire game is the small
gap on the left. You slide the middle ball right to enlarge the **worst** gap, not the best one.

> **The real objective: make the tightest neighbouring pair as wide as possible.**

### 🔑 Insight 2 — after sorting, only ADJACENT pairs matter

`|1−7| = 6` can never be the minimum: the balls at 1 and 4 are already closer. A non-adjacent pair is
always ≥ some adjacent pair between them. One of the two reasons sorting is mandatory (§5).

### 🔑 Insight 3 — stop searching for the arrangement, start verifying a number

This is the move that turns an impossible optimisation into an easy check.

> Instead of *"what is the best arrangement?"* ask:
> **"If I demand every gap be at least `g`, can I still fit all m balls?"** — a yes/no question.

Checking it is trivially greedy: ball at the leftmost basket, walk right, drop a ball at the first
basket ≥ `g` from the last one placed. Count them.

`[1,2,3,4,7]`, m = 3, by hand:

| demand `g` | greedy placement | balls fit | feasible? |
|---|---|---|---|
| 2 | 1, 3, 7 | 3 | ✅ |
| 3 | 1, 4, 7 | 3 | ✅ |
| 4 | 1, 7 | 2 | ❌ |

**Answer = 3 = the largest feasible `g`.**

---

## 4. Monotonicity — the property that licenses binary search

```
gap:       1    2    3    4    5    6   ...
feasible:  T    T    T    F    F    F   ...
```

**Monotonic** means the answers flip direction **exactly once**. Never `T F T F`.

> ### ⚠️ What "prove monotonicity" actually means
> Not *"the answer is the last true"* — that's **where the answer sits**, a consequence.
>
> The property is two claims:
>
> 1. **If gap `g` is feasible, every smaller gap is feasible.** A placement where all gaps ≥ `g`
     >    automatically satisfies all gaps ≥ anything smaller. Relaxing a requirement can't break a
     >    solution you already have.
> 2. **If gap `g` is infeasible, every larger gap is infeasible.** Tightening an already-impossible
     >    requirement can't make it possible.
>
> Together: `T…T F…F`. **That** is monotonicity.

### Why it's the whole licence

Binary search discards half the space after one test — legitimate only if the whole discarded half
is known to behave the same way. On a jumbled `T F T T F`, testing the middle tells you nothing
about either side. **No monotonicity ⇒ no binary search.**

Make it a checklist item: *before* writing a binary search over an answer space, state both
implications for that specific problem. If you can't, you don't have a binary search.

---

## 5. ⚠️ Positions vs gaps — the dominant error in this problem

Every variable inside the search is a **distance**, never a coordinate:

| wrong | right | why |
|---|---|---|
| `low = position[0]` | `low = 1` | searching over **gaps**, not locations |
| `high = position[n-1]` | `high = position[n-1] − position[0]` | largest *number* ≠ largest *gap* |
| `possibleAns = position[0]` | `possibleAns = 0` | the answer is a distance |
| `ballsToPlace = gap` | `ballsToPlace = m` | a gap is not a ball count |

**The counterexample that kills `high = position[n-1]`:** `position = [100, 200]`, m = 2. That rule
gives `high = 200`, but the widest achievable gap is `100`.

> ### 🔑 The units check
> **Before naming any variable in a binary-search-on-answer problem, ask: is this a coordinate or a
> distance?** `lo`, `hi`, `mid`, `possibleAns` are all **distances**. No index, no coordinate, ever.
>
> This is a *units error*, the same class as adding metres to seconds. Catch it at naming time, not
> at debug time.

### Why the array must be sorted — two independent reasons

1. **"Only adjacent pairs matter" is only true in sorted order** (Insight 2). Unsorted, the minimum
   pairwise gap could be between any two elements.
2. **The greedy check depends on it.** *"Walk right and place at the first basket ≥ g away"* only
   makes sense traversing in increasing coordinate order.

*"So I know where to place the balls"* is hand-waving. Have the real reason.

---

## 6. The feasibility check — and its subtle bug

### 🚩 Advancing the target instead of tracking the placement

```java
int minPosToFind = position[0] + gap;
for (int i = 1; i < position.length; i++) {
    if (position[i] >= minPosToFind) {
        minPosToFind += gap;        // 🚩 WRONG
        ballsToPlace--;
    }
}
```

`minPosToFind += gap` advances from the **previous target**, not from **where the ball actually
landed**. These differ whenever you overshoot.

Dry run, `position = [1, 5, 6]`, gap = 3:

| step | target | position | action | target after |
|---|---|---|---|---|
| start | 4 | – | ball at 1 | 4 |
| i=1 | 4 | 5 | 5 ≥ 4 → place at **5** | `4+3 = 7` ❌ |
| | | | *ball is at 5, so next must be ≥ **8**, not ≥ 7* | |
| i=2 | 7 | 6 | 6 < 7 → skip | — |

Correct here **only by luck**. With `[1, 5, 7]` it would wrongly accept 7 (a gap of 2 from the ball
at 5).

### ✅ The fix — anchor to the last actual placement

```java
minPosToFind = position[i] + gap;   // anchored to where the ball IS
```

Clearer still: keep `lastPlaced` and test `position[i] - lastPlaced >= gap`.

> **General lesson:** when a greedy walk has a "next acceptable value", derive it from **state that
> actually happened**, never from the previous derived value. Otherwise the error compounds silently.

### 🚩 Forgetting to pass `m`

The feasibility function needs **three** things: the sorted array, the candidate value, and the
constraint being satisfied. **Write its signature before its body** — omitting the constraint
parameter is what produces the `ballsToPlace = gap` units error.

---

## 7. Solution

```java
class Solution {
    public int maxDistance(int[] position, int m) {
        Arrays.sort(position);

        int low  = 1;                                           // smallest meaningful gap
        int high = position[position.length - 1] - position[0]; // widest possible gap
        int possibleAns = 0;

        while (low <= high) {
            int mid = low + (high - low) / 2;                   // candidate GAP
            if (isFeasible(position, mid, m)) {
                possibleAns = mid;    // record it — the answer is the LAST true
                low = mid + 1;        // feasible ⇒ try bigger  ← the flip
            } else {
                high = mid - 1;       // infeasible ⇒ must go smaller
            }
        }
        return possibleAns;
    }

    // can all `balls` be placed with every adjacent gap >= `gap`?
    private boolean isFeasible(int[] position, int gap, int balls) {
        int lastPlaced = position[0];
        balls--;                      // first ball occupies position[0]

        for (int i = 1; i < position.length; i++) {
            if (position[i] - lastPlaced >= gap) {
                lastPlaced = position[i];
                if (--balls == 0) return true;   // exit the instant we're done
            }
        }
        return balls == 0;
    }
}
```

### Notes on the shape

- **`possibleAns` is written only in the `true` branch**, immediately before `low = mid + 1`. Every
  later `true` is larger, so the last write wins. That's how you capture a *last-true* boundary. (In
  a *first-true* problem like LC 875, you capture in the `true` branch and move `high` instead.)
- **`balls--` before the loop** accounts for the ball at `position[0]`, which the loop never visits.
  See the convention warning below — this is the single easiest thing to get half-right.
- **Early exit immediately after the decrement**, not at the top of the next iteration.
- **`return balls == 0`** catches the case where the final ball lands on the last basket.

### 🚩 Pick ONE ball-counting convention

There are two valid shapes, and mixing them double-counts the first ball:

| | first ball counted | loop starts at | pass to the helper |
|---|---|---|---|
| **A** | outside the loop (`balls--` up front) | `i = 1`, with `lastPlaced = position[0]` | `m` |
| **B** | inside the loop (the `i = 0` test succeeds immediately) | `i = 0`, with the target set to `position[0]` | `m` |

Half of each — decrementing up front **and** letting the loop count `position[0]` — reports success
one ball early.

Trace `[1,2,3,4,7]`, `m = 3`, `gap = 3`, with `balls` pre-decremented to 2 *and* a loop that counts
the first ball:

```
i=0: target=1, position[0]=1 → 1<=1 → balls=1, target=4
i=3: position[3]=4          → 4<=4 → balls=0 → return true
```

Two placements registered, `true` returned. The ball at 7 was never checked.

It often still passes, because the error only matters when the over-reported gap exceeds `hi` —
and `hi = max − min` keeps those candidates out of the search range. **Correct by range, not by
logic.** Check `[1,2]` with `m = 2`: a pre-decremented count returns `true` for *any* gap.

### 🚩 Binary-search form: `low <= high` with the `ans` variable

Two shapes get conflated:

| form | loop | on false | needs `ans`? |
|---|---|---|---|
| **record-and-narrow** | `while (low <= high)` | `high = mid - 1` | ✅ yes |
| **converge-on-boundary** | `while (low < high)` | `high = mid` | ❌ `low` is the answer |

Mixing them — `while (low < high)` with `high = mid - 1` — can exit having skipped a candidate. With
`low = 3, high = 4` and both feasible: `mid = 3`, `low = 4`, loop exits immediately, `ans = 3` when
it should be 4.

Use `while (low <= high)` with `high = mid - 1` and the recorded `ans`. And `ans = mid` suffices —
`Math.max(ans, mid)` is redundant, since `low` only moves right so each feasible `mid` exceeds the
last recorded one.

### 🚩 `Math.ceil` on integer division does nothing

`Math.ceil(low + (high - low) / 2)` — the division truncates to an `int` before `ceil` ever sees it.
A ceiling-mid is only needed in the `while (low < high)` / `high = mid` form, to avoid an infinite
loop when `low` and `high` are adjacent; with `low <= high` it's unnecessary.

### 🚩 Stopping the loop with a guard instead of returning

`for (...; i < n && balls >= 0; i++)` lets one extra iteration run after `balls` hits 0 — the
condition is checked at the *top* of the next pass. Return the instant the count reaches zero.

---

## 8. ⚠️ Overflow — get the arithmetic right

```
high = maxPosition − minPosition  ≤  10⁹        (a GAP, not a coordinate)
low  ≤ high                       ≤  10⁹
low + high                        ≤  2 × 10⁹
Integer.MAX_VALUE                 =  2.147 × 10⁹
```

**No overflow here.** And note *why*: `high` is a **gap**, so it's bounded by 10⁹. The
positions-vs-gaps distinction that causes four bugs also happens to keep the arithmetic safe.

> **Still write `low + (high − low) / 2`.** It costs nothing and you shouldn't re-derive the bound
> every time — make it reflex.
>
> But know **which case you're in** rather than reciting "it always overflows." The version that
> genuinely breaks is when `high` approaches `Integer.MAX_VALUE` itself (binary searching over the
> full int range). Saying *"it doesn't overflow here, and here's the bound"* is stronger than blindly
> defending.

---

## 9. Complexity

| | | why |
|---|---|---|
| **Time** | `O(n log n + n log k)` | sort, plus `log k` search iterations × `O(n)` per check, where **`k = maxPos − minPos`** (the *range*, not the largest value) |
| **Space** | **O(log n)** | ⚠️ **not O(1)** — `Arrays.sort` on primitives is dual-pivot quicksort and recurses |

**Do not merge the two time terms.** `O(n log n) + O(n log k)` is not `O(n log nk)`. Neither
dominates: `n log n` wins when the coordinate range is small, `n log k` when positions spread to
10⁹. Quote both.

> `O(n log n + n log k)` **is** the intended complexity — there's no better bound for this problem.
> Runtime percentiles on it are mostly sort time plus JIT variance; don't chase them.

---

## 10. Edge cases

| case | expected | ✓ |
|---|---|---|
| `m == 2` | `maxPos − minPos` (both extremes) | ✅ `high` is exactly this, and it's feasible |
| `m == n` | min adjacent gap in the sorted array | ✅ every basket used |
| duplicate positions | gap 0 between them | ✅ `low = 1` means a 0-gap answer is never returned |
| unsorted input | must still work | ✅ sorted first |
| `[100, 200]`, m = 2 | 100 | ✅ the counterexample that kills `high = position[n-1]` |

---

## 11. Interview Q&A

**Q: Why can you binary search here — the array isn't sorted by the thing you're searching for.**
Because you're not searching the array. You're searching the *answer space* (gaps 1 to
maxPos−minPos), and feasibility over that space is monotonic.

**Q: Prove the monotonicity.**
Two implications, §4. A placement satisfying "all gaps ≥ g" satisfies "all gaps ≥ anything smaller";
and tightening an impossible requirement can't make it possible.

**Q: Why is the greedy feasibility check optimal — couldn't a different placement fit more balls?**
No. Placing each ball at the earliest legal basket leaves the maximum possible room for the
remaining balls. Any other choice is at least as far right, so it can only fit fewer.

**Q: Why place the first ball at `position[0]`?**
Moving it right only shrinks the gap to its neighbour; there's never a reason not to use the
leftmost basket.

**Q: Does `mid` overflow?**
Not here — `high` is a gap bounded by 10⁹, so `low + high ≤ 2 × 10⁹ < Integer.MAX_VALUE`. Write the
safe form regardless.

**Q: Is the space O(1)?**
Your own usage is, but `Arrays.sort` recurses — O(log n) stack.

---

## 12. Pattern

> **If a value is achievable, everything easier than it is achievable too — so the feasible/
> infeasible boundary is a single flip, and you can binary search for it.**

Note it says nothing about balls, baskets, or cows. That's why it transfers.

---

## 13. Takeaways

1. **When a statement won't parse, abandon it.** Take the smallest input, solve it by hand, then ask
   what your hand was optimising.
2. **Maximise-the-minimum ⇒ only the worst gap is scored.** Every other gap is noise.
3. **Invert optimisation into verification.** "What's the best?" is hard; "is X achievable?" is
   usually greedy and easy. This inversion *is* binary-search-on-answer.
4. **Monotonicity is two implications**, not "where the answer sits". State both before writing the
   search.
5. **Units check: coordinate or distance?** Ask it at *naming* time.
6. **Anchor greedy steps to what actually happened**, not to the previously derived target.
7. **Write the feasibility function's signature first** — array, candidate, constraint.
8. **Capture the boundary in the branch matching the direction:** last-true ⇒ record on `true` then
   `low = mid + 1`; first-true ⇒ record on `true` then `high = mid − 1`.
9. **Pick one ball-counting convention and keep it.** Decrementing up front *and* counting the
   first ball in the loop reports success one ball early — and often still passes, because the
   search range hides it.
10. **Match the loop form to the discard step:** `low <= high` with `high = mid - 1` and a recorded
    `ans`, or `low < high` with `high = mid` and no `ans`. Mixing them can skip a candidate.
11. **`Arrays.sort` is O(log n) space**, not O(1).
12. **Don't merge separate time terms.** `O(n log n) + O(n log k)` stays as two terms.
13. **Know your overflow bound rather than reciting it** — but write the safe form anyway.