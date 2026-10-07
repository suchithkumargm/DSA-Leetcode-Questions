# LC 70 — Climbing Stairs

**Difficulty:** Easy · **Pattern:** 1-D DP — the canonical introduction to the four-stage progression

---

## 1. Problem

You're climbing a staircase of `n` steps. Each move is either **1 step or 2 steps**. In how many
distinct ways can you reach the top?

```
n = 2  →  2      (1+1, 2)
n = 3  →  3      (1+1+1, 1+2, 2+1)
n = 4  →  5
```

Constraints: `1 ≤ n ≤ 45`.

---

## 2. The recurrence

To arrive at step `n`, your **last** move was either a 1-step (from `n−1`) or a 2-step (from `n−2`).
Those two sets of paths are disjoint and together they're everything.

> ### 🔑 The fact
> **`ways(n) = ways(n−1) + ways(n−2)`** — because the last move was either a 1 or a 2, and nothing
> else.

Base cases: `ways(1) = 1`, `ways(2) = 2`.

This is **Fibonacci, shifted by one**: `1, 2, 3, 5, 8, 13…` Saying that unprompted is a free signal
in an interview.

### Why this problem matters more than its rating

It's the smallest problem that exercises the whole DP progression. The four versions below are the
same recurrence with different machinery, and walking an interviewer through them is worth more
than landing on the last one directly.

---

## 3. Stage 1 — Plain recursion (exponential)

```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;
        return climbStairs(n - 1) + climbStairs(n - 2);
    }
}
```

**Time: O(2ⁿ).** More precisely Θ(φⁿ) with φ ≈ 1.618 — the call tree *is* the Fibonacci recurrence —
but 2ⁿ is the accepted bound to quote.

**Space: O(n)**, the recursion stack.

> ⚠️ **It is NOT O(log n).** Nothing halves here. Stack depth is the **longest root-to-leaf path in
> the call tree**, and following the `n−1` branch gives `n → n−1 → n−2 → … → 2`, i.e. **n** levels.
>
> The `n−2` branch returns before the `n−1` branch's frames are allocated — frames pop as subtrees
> finish, so the stack holds the current path only, never the whole tree. Same reasoning as tree
> traversals: *stack depth = longest path, not total node count.*

Note the asymmetry: **exponential time, linear space.** The call tree is enormously wide but no
deeper than n.

TLEs around n ≈ 40, which the constraint of 45 is deliberately set to catch.

---

## 4. Stage 2 — Memoisation (top-down DP)

### The waste

`climbStairs(n−2)` is computed inside the `n−1` branch *and* again directly. That duplication
compounds at every level.

> ### 🔑 Why caching pays off
> **Overlapping subproblems.** There are only `n` distinct subproblems, but the naive recursion
> solves them exponentially many times. Cache each one and the exponential tree collapses to `n`
> computations.
>
> Without overlap, memoisation buys nothing — merge sort recurses heavily but never repeats a
> subproblem, which is why it isn't DP.

```java
class Solution {
    public int climbStairs(int n) {
        return go(n, new int[n + 1]);
    }

    private int go(int n, int[] memo) {
        if (n <= 2) return n;
        if (memo[n] != 0) return memo[n];     // already computed

        memo[n] = go(n - 1, memo) + go(n - 2, memo);   // store MY result
        return memo[n];
    }
}
```

**Time: O(n)** — n distinct subproblems, each computed once, O(1) work each.
**Space: O(n) array + O(n) recursion stack = O(n).** Two separate contributions; name both.

### Is this DP?

Yes — **top-down DP** is exactly recursion plus memoisation. The two requirements:

- **Overlapping subproblems** — `f(n−2)` is needed by both `f(n−1)` and `f(n)`.
- **Optimal substructure** — the answer for `n` is built from the answers for `n−1` and `n−2`.

The other flavour is **bottom-up** (§5): fill the table from the base cases with a loop, no
recursion.

### Three things to get right

**Recurse into the helper, not the public method.** Calling `climbStairs(n-1)` allocates a *brand
new* array at every level, so the memo is discarded immediately and you're still at O(2ⁿ) — now
with allocation on top. The cache must be threaded through.

**A function memoises its own result, not its callees'.**

```java
memo[n-1] = climb1;        // 🚩 caching the child's answer
memo[n-2] = climb2;        // 🚩 the child should have done this itself
return climb1 + climb2;    // 🚩 memo[n] never written
```

This often still passes, because children happen to get cached on the way back up — but it breaks
the invariant the technique rests on: **after `f(k)` returns, `memo[k]` holds the answer.** Store
your own result at `memo[n]`; the recursive calls will have written their own entries already.

**Justify the sentinel.** `int[]` defaults to 0, and the answer is always ≥ 1 for n ≥ 1, so 0
unambiguously means "uncomputed". That reasoning should be deliberate — if a valid answer could be
0, you'd need `Integer[]` with null, or `Arrays.fill(memo, -1)`.

---

## 5. Stage 3 — Bottom-up table

Same recurrence, filled forward with a loop. No recursion, so no stack.

```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;

        int[] dp = new int[n + 1];
        dp[1] = 1;
        dp[2] = 2;

        for (int i = 3; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2];
        }
        return dp[n];
    }
}
```

**Time: O(n) · Space: O(n)** — the array only, no call stack.

> The `n <= 2` guard isn't cosmetic: `new int[n+1]` has only 2 slots when `n == 1`, so `dp[2] = 2`
> would be out of bounds without it.

**What you gained over §4:** same time, but the recursion stack is gone — so no stack-overflow risk
on large `n`, and no function-call overhead.

---

## 6. Stage 4 — Rolling variables, O(1) space ⭐ the follow-up answer

Look at the loop body: `dp[i] = dp[i-1] + dp[i-2]`. At any moment you read **exactly two cells**.
Everything below `i-2` is never touched again — the array is holding n values to serve two.

```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;

        int prev = 1;    // ways to reach step i-2
        int curr = 2;    // ways to reach step i-1

        for (int i = 3; i <= n; i++) {
            int next = prev + curr;   // ways to reach step i
            prev = curr;              // the window slides forward
            curr = next;
        }
        return curr;
    }
}
```

**Time: O(n) · Space: O(1).**

### ⚠️ The update order

`next` must be computed **before** either assignment. Then `prev = curr` must come **before**
`curr = next`, or `curr`'s old value is gone and `prev` gets the wrong one.

Trace `n = 4`:

| i | `next = prev + curr` | `prev` after | `curr` after |
|---|---|---|---|
| start | — | 1 | 2 |
| 3 | 1 + 2 = **3** | 2 | 3 |
| 4 | 2 + 3 = **5** | 3 | 5 |

Return **5** ✅

### ⚠️ Name the variables for the window, not for an offset

Names like `dpIMinus1` / `dpIMinus2` **break at the shift**: the instant you assign, the name refers
to the wrong `i`, and the code stops being readable exactly where it's hardest to get right. That's
where the self-assignment bug `dpIMinus1 = dpIMinus1;` comes from — a no-op that freezes one
variable while the other advances.

`prev` and `curr` describe **where the window is now**. After the slide they're still honest:
`curr` is "the latest step I know", whatever step that happens to be.

And check your initial values against the names. At `i = 3`, `dp[i-1]` is `dp[2] = 2` and `dp[i-2]`
is `dp[1] = 1`. Getting these backwards doesn't change the sum — addition commutes — but it
guarantees you'll write the shift wrong, because the names no longer tell you which value is older.

---

## 7. Progression summary

| Stage | Time | Space | Note |
|---|---|---|---|
| 1 — plain recursion | O(2ⁿ) | O(n) stack | TLEs past n ≈ 40 |
| 2 — memoised (top-down) | O(n) | O(n) array + O(n) stack | the exponential tree collapses |
| 3 — bottom-up table | O(n) | O(n) array | no stack, no recursion overhead |
| 4 — rolling variables | **O(n)** | **O(1)** | the follow-up answer |

**Walk this progression out loud in an interview.** Starting at stage 4 shows you memorised an
answer; walking 1→4 shows you can derive one.

---

## 8. Common pitfalls

- **Claiming O(log n) stack** for the recursion. Nothing halves; depth is n.
- **Recursing into the public method**, re-allocating the memo each call.
- **Caching the children's results instead of your own**, so `memo[n]` is never written.
- **Not justifying the 0 sentinel.** It works only because 0 is never a valid answer.
- **Omitting the `n <= 2` guard** in the table version → `dp[2]` out of bounds when `n == 1`.
- **Self-assignment in the shift** (`prev = prev`). A symptom of offset-based variable names.
- **Wrong shift order** — clobbering `curr` before `prev` has taken its value.
- **Forgetting that `ways(2) = 2`, not 1.** The base cases are the one place this differs from
  textbook Fibonacci.

---

## 9. Edge cases

| case | expected | ✓ |
|---|---|---|
| `n = 1` | 1 | ✅ the `n <= 2` guard |
| `n = 2` | 2 | ✅ guard again — **not** 1 |
| `n = 3` | 3 | ✅ first value the loop computes |
| `n = 45` (max) | 1836311903 | ✅ fits in `int` — but only just; n = 46 would overflow |

> `fib(46)` exceeds `Integer.MAX_VALUE`. The constraint `n ≤ 45` exists precisely so `int` suffices.
> Worth noticing: it's a hint that the setter thought about overflow, and a sign you should too.

---

## 10. Interview Q&A

**Q: Why is the naive recursion exponential?**
Each call spawns two, and subproblems repeat — `f(n−2)` is recomputed inside the `f(n−1)` branch.

**Q: Why is its space only O(n) when time is O(2ⁿ)?**
The stack holds one root-to-leaf path at a time, not the whole call tree. Frames pop as subtrees
finish.

**Q: What makes memoisation work here?**
Overlapping subproblems — only n distinct inputs exist, so caching collapses the exponential tree to
n computations.

**Q: Top-down or bottom-up — which is better?**
Same time. Bottom-up avoids the call stack and the recursion overhead; top-down only computes
subproblems that are actually reached, which matters when the state space is sparse. Here it's
dense, so bottom-up wins.

**Q: Can you do O(1) space?**
Yes — the recurrence only ever reads two cells. §6.

**Q: What if you could climb 1, 2, or 3 steps?**
`ways(n) = ways(n−1) + ways(n−2) + ways(n−3)`, three rolling variables. The structure is unchanged —
only the number of predecessors moves.

**Q: What if each step had a cost and you wanted the cheapest climb?**
LC 746. Same recurrence shape with `min` and a cost term instead of a sum: `dp[i] = cost[i] +
min(dp[i−1], dp[i−2])`.

**Q: Could you do better than O(n)?**
Yes — matrix exponentiation or Binet's formula gives O(log n). Worth mentioning; floating-point
precision makes Binet unreliable for large n, so matrix power is the real answer. Overkill at n ≤ 45.

---

## 11. Pattern

> **The last move was either a 1 or a 2 — so the ways to reach a step are the sum of the ways to
> reach the two steps behind it.**

The transferable half: **count paths to a state by partitioning on the final move.** That's the
move behind nearly every counting DP — Unique Paths, Decode Ways, Coin Change (number of ways),
House Robber. Ask *"what were the possible last decisions, and what state did each come from?"* and
the recurrence writes itself.

And the structural half, which is the real lesson of this problem:

> **If the recurrence only reads a fixed number of recent cells, the table can collapse to that many
> variables.**

---

## 12. Takeaways

1. **Derive the recurrence by partitioning on the last move**, not by pattern-matching to Fibonacci.
2. **Recursion stack depth is the longest path in the call tree** — here n, never log n.
3. **Memoisation pays off because of overlapping subproblems.** No overlap, no benefit.
4. **A function caches its own result**, not its callees'. The invariant is *after `f(k)` returns,
   `memo[k]` holds the answer*.
5. **Thread the cache through the recursive calls** — allocating it inside the recursion defeats it.
6. **Justify your sentinel value.** 0 works only when 0 is never a valid answer.
7. **Collapse the table when the recurrence has a fixed lookback.** Two cells read ⇒ two variables.
8. **Name rolling variables for the window (`prev`/`curr`), not for an offset.** Offset names go
   stale at the exact moment you shift.
9. **Compute `next` first, then slide `prev` before `curr`.**
10. **Walk the whole progression** — exponential → memoised → table → rolling. The derivation is
    what's being assessed.