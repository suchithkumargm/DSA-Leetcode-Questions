# LC 62 — Unique Paths

**Difficulty:** Medium · **Pattern:** 2-D grid DP — count paths by partitioning on the last move

---

## 1. Problem

A robot starts at the top-left of an `m × n` grid and may move only **right** or **down**. Count
the distinct paths to the bottom-right corner.

```
m = 3, n = 7  →  28
m = 3, n = 2  →  3
```

Constraints: `1 ≤ m, n ≤ 100`. The answer fits in a 32-bit int.

---

## 2. The recurrence

To arrive at the destination from a cell, the robot's **first** move is either right or down, and
those two sets of paths are disjoint.

> ### 🔑 The fact
> **`paths(r, c) = paths(r, c+1) + paths(r+1, c)`** — and the count depends only on the cell, never
> on how you got there.

That second clause is what makes the subproblem well-defined and the cache valid. A cell's answer
is a property of the cell.

**Base cases:**
- destination → `1` (exactly one path from a cell to itself: the empty path)
- off the grid → `0`

> ⚠️ "One path to itself" is a *count of paths*, not a distance. Don't say "it takes 0 moves" — the
> table stores path counts, and conflating the two will bite when the recurrence gets richer (LC 64
> stores sums, LC 63 stores counts with obstacles).

---

## 3. Stage 1 — Plain recursion (TLE)

```java
class Solution {
    public int uniquePaths(int m, int n) {
        return paths(0, 0, m, n);
    }

    private int paths(int row, int col, int m, int n) {
        if (row >= m || col >= n) return 0;          // off the grid
        if (row == m - 1 && col == n - 1) return 1;  // destination

        return paths(row, col + 1, m, n) + paths(row + 1, col, m, n);
    }
}
```

**Time: O(2^(m+n)).** Each call branches into two and the depth is `m + n`.
**Space: O(m + n)** — the recursion stack, which is the *longest path*, not the node count.

### Why it's wasteful

The same cell is recomputed by different branches:

```
                (0,0)
               /     \
           (0,1)      (1,0)
           /   \      /    \
       (0,2) (1,1) (1,1)  (2,0)     ← (1,1) computed twice, with its whole subtree
```

That duplication compounds at every level. TLEs around 23 × 12.

**The logic is right** — TLE rather than a wrong answer tells you the recurrence is correct and
only the repeated work is the problem.

### A cleaner base-case structure

Guarding each recursive call with `if (col < n-1)` / `if (row < m-1)` also works, but it pushes the
bounds logic to the *call site*. A single out-of-bounds base case returning 0 is less branching and
keeps a rule worth internalising:

> **Every guard in a call is about the cell you're standing on.** Check my bounds, check my cache,
> store my answer. Recursive calls stay plain and unconditional — the child protects itself.

---

## 4. Stage 2 — Memoisation (top-down DP)

> **What identifies a subproblem?** The cell — so the memo is a 2-D array indexed `[row][col]`.

```java
class Solution {
    public int uniquePaths(int m, int n) {
        return paths(0, 0, m, n, new int[m][n]);
    }

    private int paths(int row, int col, int m, int n, int[][] memo) {
        if (row >= m || col >= n) return 0;
        if (row == m - 1 && col == n - 1) return 1;

        if (memo[row][col] != 0) return memo[row][col];      // cache hit

        memo[row][col] = paths(row, col + 1, m, n, memo)
                       + paths(row + 1, col, m, n, memo);    // store MY result
        return memo[row][col];
    }
}
```

**Time: O(m × n) · Space: O(m × n).**

### How to compute the time of any memoised solution

> **Number of distinct states × work per state, excluding recursion.**

Here: `m × n` cells, each computed once (every later request is a cache hit), each doing O(1) work —
two lookups and an add. So **O(m × n)**.

Don't try to reason about the call tree's shape; the cache flattens it into the grid.

### Space has two contributions

- **O(m × n)** for the memo
- **O(m + n)** for the recursion stack — the longest chain is all-right then all-down

The memo dominates, so **O(m × n)** overall. Name both anyway.

### ⚠️ Justify the sentinel

`int[][]` zero-initialises, and 0 is never a valid answer for an in-bounds cell — every cell has at
least one path to the destination. So 0 is a safe "uncomputed" marker.

**Ask this every time:** *can the real answer ever equal my marker?* In many problems 0 **is** a
valid answer, and then you need `Arrays.fill(row, -1)` or a separate `boolean[][]`.

### ⚠️ Order the checks: bounds, then cache

The cache lookup indexes the array, so it must come **after** the bounds check. Reversed, you index
with an out-of-range value.

### ⚠️ Memoise your own result, not your children's

Writing `memo[row][col+1] = ...` at the call site reads the array *before* the recursive call, so
the callee's bounds check can't protect it — out-of-bounds crash on the last row and column. And it
leaves `memo[row][col]` unwritten, breaking the invariant: **after `paths(r,c)` returns,
`memo[r][c]` holds the answer.**

---

## 5. Stage 3 — Bottom-up table

Same recurrence, filled with loops. The fill order must guarantee each cell's dependencies already
exist — so going backwards from the destination, matching the recursion's direction.

```java
class Solution {
    public int uniquePaths(int m, int n) {
        int[][] dp = new int[m][n];

        // last column: only "down" is available → exactly one path
        for (int row = 0; row < m; row++) dp[row][n - 1] = 1;
        // last row: only "right" is available → exactly one path
        for (int col = 0; col < n; col++) dp[m - 1][col] = 1;

        for (int row = m - 2; row >= 0; row--) {
            for (int col = n - 2; col >= 0; col--) {
                dp[row][col] = dp[row + 1][col] + dp[row][col + 1];
            }
        }
        return dp[0][0];
    }
}
```

**Time: O(m × n) · Space: O(m × n)** — the table only, no call stack.

> The **entire last row and last column** are 1, not just the destination. From anywhere in the last
> row the robot can only go right, so there's exactly one path; likewise down the last column. Both
> loops also set `dp[m-1][n-1]`, so an explicit `dp[m-1][n-1] = 1` is redundant.

### ⚠️ `O(m × n)`, not `O(n²)`

The dimensions are independent variables. Write `n²` only when it's genuinely square. Easy to slip
into out of habit — and a tell that the notation came from reflex rather than from counting the
loops. Time and space are the *same quantity* here; if you write them differently, one of them is
wrong.

---

## 6. Stage 4 — One row, O(n) space

**The collapse rule, stated generally:**

> **A DP table collapses to the span the recurrence reaches back over.**

- LC 70's recurrence reads two cells back → **two variables**
- This recurrence reads `dp[row][col+1]` (same row) and `dp[row+1][col]` (**the row below**) → **one
  full row**

So it's O(n), not O(1). "I only need two values" is true for a single cell but false across the
loop — you need the whole row below available while processing the current row.

```java
class Solution {
    public int uniquePaths(int m, int n) {
        int[] dp = new int[n];
        Arrays.fill(dp, 1);                       // the last row

        for (int row = m - 2; row >= 0; row--) {
            for (int col = n - 2; col >= 0; col--) {
                dp[col] = dp[col] + dp[col + 1];
            }
        }
        return dp[0];
    }
}
```

**Why one array serves both roles:** when you read `dp[col]` it still holds the row below's value
(not yet overwritten this pass), and `dp[col + 1]` already holds the current row's value (written a
moment ago). The traversal order is what makes the aliasing safe — change the direction and it
breaks.

**Time: O(m × n) · Space: O(n)** — or `O(min(m, n))` if you iterate over the shorter dimension.

---

## 7. Stage 5 — Pure combinatorics, no table

Every path is exactly `m - 1` downs and `n - 1` rights, in some order. The path is determined
entirely by *which* of the `m + n - 2` moves are the downs.

> **Answer = C(m + n − 2, m − 1)**

```java
public int uniquePaths(int m, int n) {
    long result = 1;
    int k = Math.min(m, n) - 1;          // smaller k = fewer multiplications
    for (int i = 1; i <= k; i++) {
        result = result * (m + n - 2 - k + i) / i;   // stays integral at each step
    }
    return (int) result;
}
```

**Time: O(min(m, n)) · Space: O(1).**

⚠️ Compute it incrementally — `(m+n-2)!` overflows almost immediately. The interleaved
multiply-then-divide keeps every intermediate value a valid binomial coefficient, so it stays
integral and in range. Use `long` for the accumulator.

Interviewers sometimes ask for this *after* the DP. Offering it unprompted is a strong signal, but
lead with the DP — the DP generalises to obstacles and weights; the formula doesn't.

---

## 8. Progression summary

| Stage | Time | Space | Note |
|---|---|---|---|
| 1 — plain recursion | O(2^(m+n)) | O(m+n) stack | TLE |
| 2 — memoised | O(m×n) | O(m×n) memo + O(m+n) stack | the tree collapses into a grid |
| 3 — bottom-up table | O(m×n) | O(m×n) | no recursion |
| 4 — one row | O(m×n) | **O(min(m,n))** | the standard follow-up |
| 5 — combinatorics | **O(min(m,n))** | **O(1)** | doesn't generalise |

---

## 9. Common pitfalls

- **`O(n²)` for an `m × n` grid.** Independent dimensions.
- **Caching the children's results instead of your own** — crashes on the boundary and leaves your
  own cell unwritten.
- **Cache check before the bounds check** → out-of-range index.
- **Unjustified 0 sentinel.** Safe here; not safe in general.
- **Thinking the table collapses to O(1).** The recurrence reaches back a full row.
- **Reversing the one-row loop direction** — the aliasing that makes it work depends on the order.
- **Overflowing in the combinatorics version** by computing factorials separately.
- **Confusing "paths" with "moves"** in the base case. The destination has 1 path, 0 moves.

---

## 10. Dry run — `m = 3, n = 3`, bottom-up

Initialise last row and last column to 1:

```
[ .  .  1 ]
[ .  .  1 ]
[ 1  1  1 ]
```

Fill backwards, `dp[r][c] = dp[r+1][c] + dp[r][c+1]`:

| row | col | below | right | value |
|---|---|---|---|---|
| 1 | 1 | `dp[2][1]`=1 | `dp[1][2]`=1 | **2** |
| 1 | 0 | `dp[2][0]`=1 | `dp[1][1]`=2 | **3** |
| 0 | 1 | `dp[1][1]`=2 | `dp[0][2]`=1 | **3** |
| 0 | 0 | `dp[1][0]`=3 | `dp[0][1]`=3 | **6** |

```
[ 6  3  1 ]
[ 3  2  1 ]
[ 1  1  1 ]
```

Answer `dp[0][0] = 6` ✅ — and `C(4, 2) = 6` confirms it.

---

## 11. Edge cases

| case | expected | ✓ |
|---|---|---|
| `m = 1` or `n = 1` | 1 | ✅ the base-case loops cover it; the main loops don't run |
| `m = n = 1` | 1 | ✅ start is the destination |
| `m = 1, n = 100` | 1 | ✅ only one way: all right |
| `m = n = 100` | fits in `int` | ✅ C(198, 99) overflows, but the constraints guarantee the *answer* fits |

---

## 12. Interview Q&A

**Q: Why is the naive recursion exponential?**
Two branches per call, depth `m + n`, and cells are recomputed across branches.

**Q: Why can you memoise — what makes the subproblem well-defined?**
The number of paths from a cell depends only on the cell, not on the route taken to reach it.

**Q: How do you compute the time of a memoised solution?**
States × work per state. `m × n` states, O(1) each.

**Q: Why is the stack O(m + n) rather than O(m × n)?**
Stack depth is the longest root-to-leaf path. Frames pop as subtrees finish.

**Q: Can you do better than O(m × n) space?**
One row — O(min(m, n)). Or O(1) with the binomial formula.

**Q: Why doesn't it collapse to O(1) like Climbing Stairs?**
That recurrence reaches back two cells; this one reaches back an entire row.

**Q: What changes with obstacles?**
LC 63. An obstacle cell gets 0 paths, and the base-case rows stop at the first obstacle rather than
being uniformly 1 — one of the few places the "last row is all 1s" shortcut breaks.

**Q: What if you wanted the minimum-cost path instead of a count?**
LC 64. Same traversal, `min` plus the cell's cost instead of a sum.

---

## 13. Pattern

> **Count paths to a state by partitioning on the last (or first) move — the ways to reach a cell
> are the sum of the ways to reach the cells that can step into it.**

Same move as LC 70, one dimension up. The question that writes the recurrence is always: *what were
the possible last decisions, and what state did each come from?*

And the structural half:

> **A DP table collapses to the span the recurrence reaches back over.** Two cells back → two
> variables. One row back → one row.

---

## 14. Takeaways

1. **TLE with correct output means the recurrence is right** — only the repeated work needs fixing.
2. **A subproblem is identified by the state**, here `(row, col)` — that's what the memo is indexed
   by.
3. **Memoised time = states × work per state.** Don't reason about the call tree.
4. **Every guard is about the cell you're standing on.** Bounds, then cache, then store. Children
   protect themselves.
5. **Justify the sentinel** — 0 only works when 0 is never a valid answer.
6. **`O(m × n)`, never `O(n²)`,** for independent dimensions. If time and space are the same
   quantity, write them the same way.
7. **Stack depth is the longest path** (`m + n`), not the number of calls.
8. **The table collapses to the recurrence's reach-back span** — a row here, two variables in LC 70.
9. **The one-row trick depends on traversal order**; the aliasing is deliberate, not incidental.
10. **Know the combinatorial answer exists**, but lead with the DP — only the DP generalises.