# LC 1552 — Magnetic Force Between Two Balls (Medium)
### *= Striver SDE #65, "Aggressive Cows"*

> **Problem:** Given `position[]` (basket coordinates) and `m` balls, place all `m` balls in
> distinct baskets. The **magnetic force** between two balls is `|x - y|`. Return the **maximum
> possible minimum** force between any two balls.
>
> Example: `position = [1,2,3,4,7]`, `m = 3` → **3** (place at 1, 4, 7)
> Example: `position = [5,4,3,2,1,1000000000]`, `m = 2` → **999999999**
>
> Constraints: `n ≤ 10⁵`, `position[i] ≤ 10⁹`, `2 ≤ m ≤ n`

**Session record — Sep 21**
| | |
|---|---|
| Outcome | solved-with-approach-hint, **60 min** |
| Blocker | could not comprehend the problem statement at all for the first stretch |
| Dominant error | **confused positions with gaps — 4 separate times** |
| What worked | abandoned the statement, solved a 5-element case by hand, built the feasibility table |

---

## 0. 🔴 Why this problem was on the list — the flip

I had just solved **LC 875 (Koko Eating Bananas)** and **LC 1482 (Bouquets)**. Both **minimise**.
This one **maximises the minimum**, which inverts the search.

```
LC 875 / 1482  — minimise:          LC 1552 — maximise the minimum:
speed:     1   2   3   4   5        gap:       1   2   3   4   5
feasible:  F   F   T   T   T        feasible:  T   T   T   F   F
                   ↑                                   ↑
           answer = FIRST true                 answer = LAST true
   feasible ⇒ try smaller ⇒ hi = mid-1   feasible ⇒ try bigger ⇒ lo = mid+1
```

> **The one-sentence flip:** in a minimise problem a `true` means *"maybe I can do better by going
> smaller"* → discard the right. In a maximise-the-minimum problem a `true` means *"maybe I can do
> better by going bigger"* → discard the left.

Same algorithm, mirrored. Writing this on autopilot from the previous two problems gets it
backwards.

---

## 1. Comprehending the problem (this took the longest)

The statement did not parse. What unlocked it was **abandoning the statement and solving a tiny
case by hand.**

### Worked example — `[1,2,3,4,7]`, m = 3

Place at 1, 4, 7. Pairwise forces: `|1-4|=3`, `|4-7|=3`, `|1-7|=6` → `[3,3,6]` → **min = 3**.

### 🔑 Insight 1 — only the MINIMUM gap is scored

My first instinct was *"place balls to maximise the force"*. Wrong. The score is the **minimum**
pairwise force. Every other gap is irrelevant.

Watch it on `[1,2,3,4,5,1000000000]`, m = 3, with balls pinned at 1 and 1000000000:

| middle ball at | gaps | **min** |
|---|---|---|
| 2 | 1, 999999998 | **1** |
| 3 | 2, 999999997 | **2** |
| 5 | 4, 999999995 | **4** ✅ |

The enormous right-hand gap **never matters** — it is never the minimum. The entire game is the
small gap on the left. You slide the middle ball right to enlarge the **worst** gap, not the best
one.

> **So the real objective: make the tightest neighbouring pair as wide as possible.**

### 🔑 Insight 2 — after sorting, only ADJACENT pairs matter

In the first example I listed `|1-7| = 6` as a pair. It can never be the minimum: the balls at
1 and 4 are already closer. A non-adjacent pair is always ≥ some adjacent pair between them.

> **Sort, then ignore every non-adjacent pair.** This is one of the two reasons sorting is
> mandatory (§4).

### 🔑 Insight 3 — stop searching for the arrangement, start verifying a number

This is the move that turns an impossible optimisation into an easy check.

> Instead of *"what is the best arrangement?"* ask:
> **"If I demand every gap be at least `g`, can I still fit all m balls?"** — a yes/no question.

Checking it is trivially greedy: ball at the leftmost basket, walk right, drop a ball at the
first basket ≥ `g` from the last one placed. Count them.

`[1,2,3,4,7]`, m = 3, done by hand:

| demand `g` | greedy placement | balls fit | feasible? |
|---|---|---|---|
| 2 | 1, 3, 7 | 3 | ✅ |
| 3 | 1, 4, 7 | 3 | ✅ |
| 4 | 1, 7 | 2 | ❌ |

**Answer = 3 = the largest feasible `g`.**

---

## 2. Monotonicity — the property that licenses binary search

Extend the table:

```
gap:       1    2    3    4    5    6   ...
feasible:  T    T    T    F    F    F   ...
```

**Monotonic** means the answers flip direction **exactly once**. Never `T F T F`. Once it turns,
it stays turned.

> ### ⚠️ What "prove monotonicity" actually means
> Not *"the answer is the last true"* — that's **where the answer sits**, a *consequence*.
> The property itself is two claims:
>
> 1. **If gap `g` is feasible, every smaller gap is feasible.** A placement where all gaps ≥ `g`
     >    automatically satisfies all gaps ≥ anything smaller. Relaxing a requirement can't break a
     >    solution you already have.
> 2. **If gap `g` is infeasible, every larger gap is infeasible.** Tightening an already-impossible
     >    requirement can't make it possible.
>
> Together: `T…T F…F`. **That** is monotonicity.

### Why it's the whole licence

Binary search discards half the space after one test. That is only legitimate if the whole
discarded half is known to behave the same way. On a jumbled `T F T T F`, testing the middle
tells you nothing about either side. **No monotonicity ⇒ no binary search.**

Worth internalising as a checklist item: *before* writing a binary search over an answer space,
state the two implications above for that specific problem. If you can't, you don't have a
binary search.

---

## 3. 🔴 The dominant bug of this session — positions vs gaps

**Four separate times** I assigned a *coordinate* to a variable holding a *distance*:

| wrote | should be | why it's wrong |
|---|---|---|
| `low = position[0]` | `low = 1` | searching over **gaps**, not locations |
| `high = position[n-1]` | `high = position[n-1] - position[0]` | largest *number* ≠ largest *gap* |
| `possibleAns = position[0]` | `possibleAns = 0` | the answer is a distance |
| `ballsToPlace = gap` | `ballsToPlace = m` | a gap is not a ball count |

Counterexample that kills `high = position[n-1]`: **`position = [100, 200]`, m = 2.** That rule
gives `high = 200`, but the widest achievable gap is `100`.

> ### 🔑 Standing rule — the units check
> **Before naming any variable in a binary-search-on-answer problem, ask: is this a coordinate or
> a distance?** Every variable inside the search (`lo`, `hi`, `mid`, `possibleAns`) is a
> **distance**. No index, no coordinate, ever.
>
> This is a *units error*, the same class as adding metres to seconds. Catch it at naming time,
> not at debug time.

---

## 4. Why the array must be sorted

Two independent reasons — the second is the one I missed:

1. **"Only adjacent pairs matter" is only true in sorted order** (Insight 2). Unsorted, the
   minimum pairwise gap could be between any two elements.
2. **The greedy check depends on it.** *"Walk right and place at the first basket ≥ g away"* only
   makes sense when you traverse in increasing coordinate order.

My first answer — *"so I know where to place the balls"* — was hand-waving. Have the real reason.

---

## 5. The feasibility check — and its subtle bug

### 🚩 The bug: advancing the target instead of tracking the placement

```java
int minPosToFind = position[0] + gap;
for (int i = 1; i < position.length; i++) {
    if (position[i] >= minPosToFind) {
        minPosToFind += gap;        // ❌ WRONG
        ballsToPlace--;
    }
}
```

`minPosToFind += gap` advances from the **previous target**, not from **where the ball actually
landed**. These differ whenever you overshoot the target.

Dry run, `position = [1, 5, 6]`, gap = 3:

| step | target | position | action | target after |
|---|---|---|---|---|
| start | 4 | – | ball at 1 | 4 |
| i=1 | 4 | 5 | 5 ≥ 4 → place at **5** | `4+3 = 7` ❌ |
| | | | *ball is at 5, so next must be ≥ **8**, not ≥ 7* | |
| i=2 | 7 | 6 | 6 < 7 → skip | — |

Correct here **only by luck**. With `[1, 5, 7]` it would wrongly accept 7 (gap of 2 from the ball
at 5).

### ✅ The fix — anchor to the last actual placement

```java
minPosToFind = position[i] + gap;   // ✅ anchored to where the ball IS
```

Equivalently, and clearer: keep `lastPlaced` and test `position[i] - lastPlaced >= gap`.

> **General lesson:** when a greedy walk has a "next acceptable value", derive it from **state
> that actually happened**, never from the previous derived value. Otherwise the error compounds
> silently.

### 🚩 Forgot to pass `m` at all

The first draft never gave `isFeasible` the ball count — hence `ballsToPlace = gap`, which is
where that units error came from. **The feasibility function needs three things: the sorted array,
the candidate value, and the constraint being satisfied.** Write its signature before its body.

---

## 6. Final solution

```java
class Solution {
    public int maxDistance(int[] position, int m) {
        Arrays.sort(position);

        int low  = 1;                                          // smallest meaningful gap
        int high = position[position.length - 1] - position[0]; // widest possible gap
        int possibleAns = 0;

        while (low <= high) {
            int mid = low + (high - low) / 2;                  // candidate GAP
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
                if (--balls == 0) return true;   // early exit the instant we're done
            }
        }
        return balls == 0;
    }
}
```

### Notes on the shape

- **`possibleAns` is written only in the `true` branch**, immediately before `low = mid + 1`.
  Every later `true` is larger, so the last write wins. That's how you capture a *last-true*
  boundary. (In a *first-true* problem like LC 875, you'd capture in the `true` branch and move
  `high` instead.)
- **`balls--` before the loop** accounts for the ball at `position[0]`, which the loop never
  visits. My first version forgot this — decide explicitly whether the first ball is counted
  inside or outside the loop; don't let it fall out by accident.
- **Early exit placement.** My submitted version checked `ballsToPlace == 0` at the *top* of the
  next iteration, so it looped once more before returning. Check it **immediately after the
  decrement**.
- **`return balls == 0`** at the end catches the case where the final ball lands on the last
  basket.

---

## 7. ⚠️ The overflow question — get the arithmetic right

I claimed `low + high` could reach **4.2 × 10⁹** and overflow. **Wrong.**

```
high = maxPosition - minPosition  ≤  10⁹        (a GAP, not a coordinate)
low  ≤ high                       ≤  10⁹
low + high                        ≤  2 × 10⁹
Integer.MAX_VALUE                 =  2.147 × 10⁹
```

**No overflow here.** And notice *why*: `high` is a **gap**, so it's bounded by 10⁹ rather than
by anything larger. The positions-vs-gaps distinction that caused four bugs also happens to be
what keeps the arithmetic safe.

> **Still write `low + (high - low) / 2`.** Two reasons:
> 1. It costs nothing.
> 2. You shouldn't have to re-derive the bound every time — make it reflex.
>
> But know **which case you're in** rather than reciting "it always overflows." The version that
> genuinely breaks is when `high` approaches `Integer.MAX_VALUE` itself (e.g. binary searching
> over the full int range). Being able to say *"it doesn't overflow here, and here's the bound"*
> is stronger than blindly defending.

---

## 8. Complexity

| | | why |
|---|---|---|
| **Time** | `O(n log n + n log k)` | sort, plus `log k` binary-search iterations × `O(n)` per feasibility check, where `k = maxPos − minPos` |
| **Space** | **`O(log n)`** | ⚠️ **not O(1)** — `Arrays.sort` on primitives is dual-pivot quicksort and recurses |

**Neither time term dominates in general:** `n log n` wins when the coordinate range `k` is small;
`n log k` wins when positions spread to 10⁹. Quote both.

> 🔴 **Repeat offence:** I claimed O(1) space. This is the *exact* overclaim already written in the
> LC 169 doc as takeaway #7. **`Arrays.sort` is O(log n) stack space.** My own auxiliary usage is
> O(1) — the sort's is not, and the sort is part of my solution.

### On the 7% runtime percentile

Mostly `Arrays.sort` plus JIT variance. `O(n log n + n log k)` **is** the intended complexity —
there is no better bound for this problem. Fix the early-exit because it's cleaner, not to chase
the number. **Don't over-index on LeetCode percentiles.**

---

## 9. Edge cases checklist

| case | expected | handled? |
|---|---|---|
| `m == 2` | `maxPos − minPos` (place at both extremes) | ✅ `high` is exactly this, and it's feasible |
| `m == n` | min adjacent gap in the sorted array | ✅ every basket used |
| duplicate positions | gap 0 between them | ✅ `low = 1` means a 0-gap answer is never returned; baskets are distinct anyway |
| unsorted input | must still work | ✅ sorted first |
| `position = [100, 200]`, m=2 | 100 | ✅ — **the counterexample that kills `high = position[n-1]`** |

---

## 10. One-line pattern

> **When asked to maximise a minimum (or minimise a maximum), binary search the *answer space*,
> not the array: write a yes/no feasibility check for "is value X achievable?", verify the
> feasibility sequence is monotonic, and search for the boundary where it flips.**

Note what this sentence does *not* say: anything about balls, baskets, or cows. That's why it
transfers. *"Find the monotonicity and see where it turns false"* would fit a hundred problems
and help with none of them.

---

## 11. Takeaways

1. **When a statement won't parse, abandon it.** Take the smallest input, solve it by hand, then
   ask what your hand was actually optimising. That is what unlocked this problem — not re-reading.
2. **Maximise-the-minimum ⇒ only the worst gap is scored.** Every other gap is noise.
3. **Invert optimisation into verification.** "What's the best?" is hard; "is X achievable?" is
   usually greedy and easy. This inversion *is* binary-search-on-answer.
4. **Monotonicity is two implications**, not "where the answer sits". State both before writing
   the search.
5. **🔴 Units check: coordinate or distance?** Ask it at *naming* time. Four bugs this session.
6. **Anchor greedy steps to what actually happened,** not to the previously derived target.
7. **Write the feasibility function's signature first** — array, candidate, constraint. Forgetting
   the constraint parameter is what caused the units error.
8. **Capture the boundary in the branch that matches the direction:** last-true ⇒ record on `true`
   then `low = mid + 1`; first-true ⇒ record on `true` then `high = mid - 1`.
9. **`Arrays.sort` is O(log n) space.** Second time flagged. Stop claiming O(1).
10. **Know your overflow bound, don't recite it.** Write the safe form anyway.
11. **Ignore LeetCode percentiles** when you're already at the intended complexity.

---

## 12. Related problems (same muscles)

| # | Problem | Direction | Shared idea |
|---|---|---|---|
| 875 | Koko Eating Bananas | **min**imise | ⭐ done just before — the mirror image; compare the discard directions side by side |
| 1482 | Min Days to Make m Bouquets | **min**imise | done just before; same mirror |
| 410 | Split Array Largest Sum | **min**imise the max | greedy feasibility check, near-identical shape |
| 1011 | Capacity to Ship Packages in D Days | **min**imise the max | same as 410 with different dressing |
| 774 | Minimize Max Distance to Gas Station | **min**imise the max | real-valued answer space — binary search on doubles |
| 1231 | Divide Chocolate | **max**imise the min | ⭐ **the closest twin to 1552** — do this next |
| 2064 | Minimized Maximum Products | **min**imise the max | |
| 4 | Median of Two Sorted Arrays | — | binary search on a *partition*, not on the answer |

**Recommended next: LC 1231 (Divide Chocolate)** — the other maximise-the-minimum, to confirm the
flip is internalised rather than memorised for one problem.

---

*Sep 21 — solved-with-approach-hint, 60 min. Zero comprehension at the start; the monotonicity
table was derived independently once the problem was reframed as a yes/no check. The habit to
attack: **positions vs gaps — ask "coordinate or distance?" before naming any variable.***