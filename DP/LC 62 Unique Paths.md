# 62. Unique Paths

**Difficulty:** Medium
**Topics:** Dynamic Programming, Recursion, Memoization
**Link:** https://leetcode.com/problems/unique-paths/

---

## Problem

A robot sits at the top-left of an `m x n` grid. It can only move **right** or **down**. Count the number of distinct paths to reach the bottom-right corner.

---

## First attempt — plain recursion (TLE)

The intuition came naturally: from any cell, the total number of paths is
`paths(right neighbour) + paths(down neighbour)`. Base cases are "reached the
destination" (1 path) and "walked off the grid" (0 paths).

```java
private int moveRobot(int[][] grid, int m, int n){
    if(m == grid.length-1 && n == grid[0].length-1) return 1;
    if(m >= grid.length || n >= grid[0].length) return 0;

    int rightMoves = moveRobot(grid, m, n+1);
    int downMoves  = moveRobot(grid, m+1, n);

    return rightMoves + downMoves;
}
```

**Result:** Time Limit Exceeded on `m = 23, n = 12`.

The logic was correct — it produces the right answer. The problem was purely speed.

---

## What went wrong

Every call branches into two, so the work is **O(2^(m+n))**.

The reason it's so wasteful: the same cell gets recomputed by different branches.
Starting at `(0,0)`, the call to `(0,1)` and the call to `(1,0)` **both**
independently compute `(1,1)` from scratch — along with its entire subtree.

```
                (0,0)
               /     \
           (0,1)      (1,0)
           /   \      /    \
       (0,2) (1,1) (1,1)  (2,0)     <-- (1,1) computed twice
```

That duplication compounds at every level. At 23x12 it's millions of calls.

**Key observation:** `moveRobot(m, n)` always returns the same value regardless
of which branch asked for it. The answer depends only on `(m, n)`. So compute it
once, store it, reuse it.

---

## Fix — memoization

The `int[][] grid` was already being allocated and never used. It's exactly the
right shape to cache "answer for cell `(m,n)`". Java zero-initialises it, and `0`
is never a valid answer for an in-bounds cell, so `0` works as a
"not computed yet" marker.

```java
class Solution {
    public int uniquePaths(int m, int n) {
        int[][] grid = new int[m][n];
        return moveRobot(grid, 0, 0);
    }

    private int moveRobot(int[][] grid, int m, int n){
        if(m == grid.length-1 && n == grid[0].length-1) return 1;
        if(m >= grid.length || n >= grid[0].length) return 0;

        if(grid[m][n] != 0) return grid[m][n];   // already solved

        int rightMoves = moveRobot(grid, m, n+1);
        int downMoves  = moveRobot(grid, m+1, n);

        grid[m][n] = rightMoves + downMoves;     // store before returning
        return grid[m][n];
    }
}
```

### Mistakes made on the way to this

1. **Ternary written backwards** — `cached != 0 ? recompute() : cached`, which is
   the opposite of the intent. Since every cell starts at `0`, it always took the
   else branch and returned `0`.
2. **Cached the children instead of self** — doing `grid[m][n+1]` at the call site
   reads the array *before* the recursive call, so the bounds check inside
   `moveRobot` can't protect it. Out-of-bounds crash on the last row/column.

**Rule that fixes both:** every guard in a call should be about the cell you're
currently standing on. Check my bounds, check my cache, store my answer. The
recursive calls stay plain and unconditional — the child protects itself.

**Order matters:** the cache check must come *after* the bounds checks, otherwise
you index the array with an out-of-range value.

---

## Complexity

| | |
|---|---|
| Time | `O(m*n)` |
| Space | `O(m*n)` |

- **Time:** each cell is *computed* exactly once. It's *asked for* at most twice
  (by the cell above and the cell to the left) — every repeat hits the cache and
  returns immediately. The tree collapses into a grid.
- **Space:** `O(m*n)` for the memo grid + `O(m+n)` for the recursion stack.
  Dominated by the grid.

> Stack space is the **maximum depth** of the call stack, not the number of calls.
> Frames that returned are already popped. The longest chain here is all-right
> then all-down = `m + n` frames.

---

## Takeaways

- Memoization = recursion + a cache. This *is* top-down DP.
- Before using a sentinel value, ask: **can the real answer ever equal my
  "not computed" marker?** `0` is safe here, but in many problems `0` is a valid
  answer and you need `Arrays.fill(row, -1)` or a separate `boolean[][]`.
- Stack space = height of the recursion tree, never the node count.

## Follow-ups

- **Bottom-up (tabulation):** fill the last row and last column with `1`, then
  work backwards with `grid[i][j] = grid[i+1][j] + grid[i][j+1]`. No recursion,
  no stack.
- **Space optimised:** each cell only reads the row below and the cell to its
  right, so a single row array is enough — `O(n)` space. Common interview
  follow-up.
- **Related:** LC 63 (Unique Paths II — with obstacles), LC 64 (Minimum Path Sum).
  Both are this problem with one line changed.