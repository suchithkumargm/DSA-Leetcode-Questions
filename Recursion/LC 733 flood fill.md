# LC 733 — Flood Fill

**Difficulty:** Easy · **Sheets:** NeetCode | Striver SDE | Love Babbar 450
**Pattern:** Grid DFS — guards return early, then the work happens unconditionally

---

## 1. Problem

Given an `m × n` grid `image`, a starting pixel `(sr, sc)` and a `color`, repaint the starting pixel
and every pixel **connected to it 4-directionally** that shares its original colour. Return the
modified image.

```
image = [[1,1,1],[1,1,0],[1,0,1]], sr = 1, sc = 1, color = 2
→ [[2,2,2],[2,2,0],[2,0,1]]
```

The bottom-right `1` is **not** repainted — it touches the region only diagonally, and the
connectivity is 4-directional.

---

## 2. The shape

> ### 🔑 The fact
> **In a grid DFS, the guards `return`; they don't merely skip the work.** Every reason to stop —
> out of bounds, wrong colour, already visited — exits the function before any recursion happens.

The body is then unconditional: paint, then recurse in all four directions.

**There is no separate "visited" set.** Repainting a cell *is* marking it visited — after the paint
its colour no longer matches `originalColour`, so the guard turns it away on any later arrival. The
grid is its own bookkeeping.

---

## 3. Solution

```java
class Solution {
    public int[][] floodFill(int[][] image, int sr, int sc, int color) {
        if (image[sr][sc] != color) {                       // no-op guard, §5
            fill(image, sr, sc, color, image[sr][sc]);
        }
        return image;
    }

    private void fill(int[][] image, int sr, int sc, int color, int originalColour) {
        if (sr < 0 || sr >= image.length
         || sc < 0 || sc >= image[0].length
         || image[sr][sc] != originalColour) {
            return;
        }

        image[sr][sc] = color;

        fill(image, sr + 1, sc, color, originalColour);
        fill(image, sr - 1, sc, color, originalColour);
        fill(image, sr, sc + 1, color, originalColour);
        fill(image, sr, sc - 1, color, originalColour);
    }
}
```

**O(m × n) time · O(m × n) space** — see §7.

### Why `originalColour` is a parameter

It must be captured **once, before any painting**, and carried down. Reading `image[sr][sc]` inside
the recursion would read a value that may already have been repainted by an earlier branch.

### Order of the combined guard

The bounds checks must come **before** the array access, and `||` short-circuits left to right — so
the single combined condition is safe. Same reasoning as `fast != null && fast.next != null` in the
linked-list problems.

Four separate `if` statements are equally correct; the combined form is the conventional one and
makes "the set of reasons to stop" read as one thing.

---

## 4. ⚠️ Infinite loop #1 — guarding the work instead of the recursion

The natural first version:

```java
if (image[sr][sc] == originalColour) {
    image[sr][sc] = color;              // 🚩 only the PAINT is guarded
}

fill(image, sr + 1, sc, ...);           // 🚩 these run regardless
fill(image, sr - 1, sc, ...);
fill(image, sr, sc + 1, ...);
fill(image, sr, sc - 1, ...);
```

The recursion is **outside** the `if`, so an already-painted cell skips the paint and then recurses
into its neighbours anyway.

Trace two adjacent cells A and B, both colour 1, painting to 2:

```
fill(A): A == 1 → paint A (now 2) → recurse into B
  fill(B): B == 1 → paint B (now 2) → recurse into A
    fill(A): A == 1? NO → skip the paint
             recurse into B           ← ⚠️ still runs
      fill(B): B == 1? no → skip
               recurse into A         ← and this
        ... forever
```

A and B bounce between each other indefinitely → `StackOverflowError`.

**The fix:** make the condition control whether the function *continues at all*.

```java
if (image[sr][sc] != originalColour) return;   // not eligible, or already done
image[sr][sc] = color;
// four recursive calls
```

> **The general rule for grid DFS: guards return early, then the work is unconditional.** Guarding
> the work while leaving the recursion open is the classic version of this bug, and it shows up as a
> stack overflow rather than a wrong answer.

---

## 5. ⚠️ Infinite loop #2 — when the new colour equals the old

Even with the guard fixed, this still hangs:

```
image = [[0,0,0],[0,0,0]], sr = 0, sc = 0, color = 0
```

`originalColour` is 0 and `color` is 0, so painting **changes nothing**. The guard
`image[sr][sc] != originalColour` therefore never fires, nothing is ever marked visited, and
neighbours recurse into each other forever.

**This is an actual LeetCode test case** — it exists specifically to catch this.

**The fix — one line, before the recursion starts:**

```java
if (image[sr][sc] != color) {
    fill(image, sr, sc, color, image[sr][sc]);
}
```

If the pixel is already the target colour, the region is already correct; return the image untouched.

> **Placement matters.** The check can also live inside the helper as
> `if (image[sr][sc] == color) return;` — correct, because an eligible cell can never equal `color`
> when the two colours differ, so it's a no-op in the normal case. But that's **one comparison per
> cell** for a condition decidable **once**. Put it in the wrapper.

> **The underlying lesson:** this solution uses "the cell changed colour" as its visited-marker. Any
> time a marker can coincide with a legitimate value, the marker stops working — the same class of
> problem as a `0` memo sentinel when `0` is a valid answer (LC 70, LC 62).

---

## 6. Other pitfalls

- **Reading `originalColour` inside the recursion** instead of passing it down — by then it may
  already be repainted.
- **Leaving the helper `public`.** It's an implementation detail.
- **Diagonal neighbours.** The problem is 4-directional; adding the four diagonals solves a different
  problem.
- **Accessing `image[sr][sc]` before the bounds check** in a combined guard.
- **Using `image[0].length` where you mean `image.length`** — rows vs columns. Say which dimension a
  bound refers to before writing it.
- **Building an explicit `visited[][]`.** Unnecessary here, because the repaint is the mark. It *is*
  needed in problems where cells aren't modified (e.g. counting islands without mutating the grid).

---

## 7. Complexity

| | | why |
|---|---|---|
| **Time** | **O(m × n)** | each cell is painted at most once; later arrivals are turned away by the guard in O(1) |
| **Space** | **O(m × n)** | ⚠️ the **recursion stack** |

> ⚠️ **The space is not O(1).** On a grid where every cell shares the starting colour, the DFS can
> descend through all `m × n` cells before unwinding — a single path visiting everything. With a
> large grid that's a genuine stack-overflow risk, which is the argument for the iterative version.

**The iterative form** replaces the call stack with an explicit one:

```java
Deque<int[]> stack = new ArrayDeque<>();
stack.push(new int[]{sr, sc});
while (!stack.isEmpty()) {
    int[] cell = stack.pop();
    // same guards, then push the four neighbours
}
```

Same O(m × n) both ways — but on the heap rather than the thread stack, which is the same
recursive-versus-iterative trade as LC 104.

**BFS with a queue** works identically here. Flood fill doesn't care about visit order, so DFS and
BFS produce the same result; BFS's O(w) frontier can be smaller on wide, shallow regions.

---

## 8. Dry run — `[[1,1,1],[1,1,0],[1,0,1]]`, `sr=1, sc=1, color=2`

`originalColour = 1`. Start at `(1,1)`.

| call | guard | action |
|---|---|---|
| `(1,1)` | in bounds, is 1 | paint → 2, recurse |
| ↳ `(2,1)` | in bounds, is **0** | **return** — not part of the region |
| ↳ `(0,1)` | in bounds, is 1 | paint → 2, recurse |
| ↳↳ `(1,1)` | is **2** now | **return** — already done |
| ↳↳ `(-1,1)` | **out of bounds** | return |
| ↳↳ `(0,2)` | is 1 | paint → 2 |
| ↳↳ `(0,0)` | is 1 | paint → 2 |
| ↳ `(1,2)` | is **0** | return |
| ↳ `(1,0)` | is 1 | paint → 2, recurse → `(2,0)` is 1 → paint → 2 |

Result:

```
[[2,2,2],
 [2,2,0],
 [2,0,1]]
```

✅ The `(2,2)` cell keeps its 1 — reachable only diagonally.

**The row that matters** is `(1,1)` being revisited and returning on `is 2 now`. That's the repaint
doubling as the visited mark, and it's exactly what §4's buggy version fails to do.

---

## 9. Edge cases

| case | expected | ✓ |
|---|---|---|
| `color` already equals the start colour | unchanged | ✅ the wrapper guard (§5) |
| single-cell grid | that cell repainted | ✅ |
| entire grid one colour | all repainted | ✅ — and the **deepest** recursion, O(m×n) stack |
| start pixel isolated by different colours | only that pixel | ✅ all four neighbours fail the guard |
| region touching a grid edge | correct | ✅ bounds checks |
| diagonal-only connection | **not** repainted | ✅ 4-directional |

---

## 10. Interview Q&A

**Q: Why no visited array?**
Repainting the cell *is* the visited mark — after the paint it no longer matches `originalColour`,
so the guard rejects it on re-entry.

**Q: Then why does it infinite-loop when the new colour equals the old?**
Because the mark becomes invisible. Painting changes nothing, so no cell ever looks visited. §5.

**Q: Why pass `originalColour` down instead of reading it at each cell?**
By the time a nested call runs, the starting cell has already been repainted.

**Q: Complexity?**
O(m × n) time; O(m × n) space for the recursion stack in the worst case, when the whole grid is one
region.

**Q: DFS or BFS?**
Either — order doesn't affect the result. BFS bounds the frontier by the region's width rather than
its size, and an explicit stack or queue avoids the call-stack depth limit.

**Q: What changes for 8-directional connectivity?**
Add the four diagonal recursions. Usually expressed as a `directions` array of `{dr, dc}` pairs,
iterated in a loop, which scales better than eight hand-written calls.

**Q: How does this relate to counting islands?**
LC 200 is the same DFS with two differences: an outer loop launching a fill from every unvisited
land cell, and a counter. The fill itself is this function.