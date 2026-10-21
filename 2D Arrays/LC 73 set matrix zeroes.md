# LC 73 — Set Matrix Zeroes

**Difficulty:** Medium · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** In-place marking — using part of the input as scratch storage

---

## 1. Problem

Given an `m × n` matrix, if an element is `0`, set its **entire row and column** to `0`. Do it
**in place**.

```
[[1,1,1],          [[1,0,1],
 [1,0,1],    →      [0,0,0],
 [1,1,1]]           [1,0,1]]
```

**Follow-up:** a constant-space solution.

---

## 2. Why you can't just zero as you go

The naive instinct — walk the matrix, and whenever you find a 0 immediately zero its row and column
— **cascades the whole matrix to zero.**

The zeros you *write* are indistinguishable from the zeros that were *already there*. So the next
cell you inspect is a false positive, which zeroes another row and column, which creates more false
positives.

> **The general principle:** don't destroy the information you're still reading. Any solution must
> separate **detection** from **application** — first find out which rows and columns are doomed,
> *then* apply it.

Same idea as the restore pass in LC 138: a pass that reads state can't also be the pass that
overwrites it.

---

## 3. Approach 1 — two marker arrays

Record which rows and columns contain a zero, then apply.

```java
class Solution {
    public void setZeroes(int[][] matrix) {
        boolean[] zeroRows = new boolean[matrix.length];
        boolean[] zeroCols = new boolean[matrix[0].length];

        for (int i = 0; i < matrix.length; i++) {
            for (int j = 0; j < matrix[0].length; j++) {
                if (matrix[i][j] == 0) {
                    zeroRows[i] = true;
                    zeroCols[j] = true;
                }
            }
        }

        for (int i = 0; i < matrix.length; i++) {
            for (int j = 0; j < matrix[0].length; j++) {
                if (zeroRows[i] || zeroCols[j]) matrix[i][j] = 0;
            }
        }
    }
}
```

**O(m×n) time · O(m+n) space.**

> `boolean[]` beats `HashSet<Integer>` here: O(1) indexed lookup, no hashing, no autoboxing. Reach
> for a set only when the index space is sparse or unbounded.

This is the version to write first, state the complexity of, and then improve.

---

## 4. Approach 2 — O(1) space using the first row and column

### The idea

The two marker arrays hold `m + n` booleans. The matrix already contains a row of `n` cells and a
column of `m` cells that you can repurpose.

> ### 🔑 The fact that makes it safe
> **If the first row contains a zero, the entire first row is destined to become zero anyway — and
> the same for the first column.** So their original contents are already doomed. Once you've
> recorded *whether* they contain a zero, their cells are free to use as scratch storage.

Substitute directly:
- `zeroRows[i]` → `matrix[i][0]`
- `zeroCols[j]` → `matrix[0][j]`

For any inner cell `matrix[i][j] == 0` (with `i,j ≥ 1`), writing a 0 into `matrix[i][0]` and
`matrix[0][j]` destroys nothing you still need — those cells were going to be zeroed regardless.

### ⚠️ The collision at `matrix[0][0]`

`matrix[0][0]` belongs to **both** the first row and the first column. It would have to mean two
independent things at once:
- "row 0 contains a zero"
- "column 0 contains a zero"

**One cell, two facts.** Resolve it by pre-scanning: compute both facts into separate booleans
*before* any marking begins. Two extra booleans are still O(1).

### Solution

```java
class Solution {
    public void setZeroes(int[][] matrix) {
        int m = matrix.length, n = matrix[0].length;

        // STEP 1 — capture the two border facts BEFORE any marking corrupts them
        boolean firstColHasZero = false;
        for (int i = 0; i < m; i++) {
            if (matrix[i][0] == 0) { firstColHasZero = true; break; }
        }

        boolean firstRowHasZero = false;
        for (int j = 0; j < n; j++) {
            if (matrix[0][j] == 0) { firstRowHasZero = true; break; }
        }

        // STEP 2 — mark, using row 0 and column 0 as the marker arrays
        for (int i = 1; i < m; i++) {
            for (int j = 1; j < n; j++) {
                if (matrix[i][j] == 0) {
                    matrix[i][0] = 0;
                    matrix[0][j] = 0;
                }
            }
        }

        // STEP 3 — apply to the inner cells, while the markers are still intact
        for (int i = 1; i < m; i++) {
            for (int j = 1; j < n; j++) {
                if (matrix[i][0] == 0 || matrix[0][j] == 0) matrix[i][j] = 0;
            }
        }

        // STEP 4 — the border, LAST, because this destroys the markers
        if (firstRowHasZero) {
            for (int j = 0; j < n; j++) matrix[0][j] = 0;
        }
        if (firstColHasZero) {
            for (int i = 0; i < m; i++) matrix[i][0] = 0;
        }
    }
}
```

**O(m×n) time · O(1) space.**

### The ordering, and why it's forced

| step | reads | writes | must come |
|---|---|---|---|
| 1 — pre-scan | row 0, col 0 | nothing | **first** — marking corrupts what it reads |
| 2 — mark | inner cells | row 0, col 0 | before step 3 |
| 3 — apply inner | row 0, col 0 (markers) | inner cells | **before step 4** — step 4 destroys the markers |
| 4 — apply border | the two booleans | row 0, col 0 | **last** |

> Step 4 must come last precisely *because* it overwrites the markers — and by then nothing needs to
> read them. Reverse steps 3 and 4 and every inner cell reads a corrupted marker.

**Steps 4a and 4b can run in either order** — they're independent. `matrix[0][0]` gets zeroed if
either flag is true, which is correct: if row 0 has a zero then `matrix[0][0]` is in a doomed row,
and likewise for column 0.

**Why the two flags never collide with the markers:** both were captured before any marking
happened, so whatever `matrix[0][0]` ends up holding, neither flag depends on it.

---

## 5. Common pitfalls

**Zeroing rows and columns during the detection pass** → cascade to all zeros (§2).

**Reading `matrix[0][0]` as the row-0 or column-0 flag.** It holds one cell's value, not the answer
to *"does this row/column contain a zero?"* Both flags need their own scan.

**Marking from `i = 0, j = 0`.** Then row 0 and column 0 mark themselves, and you can no longer
distinguish an original border zero from a marker. Mark from `i = 1, j = 1` only.

**Skipping the pre-scan of row 0 and column 0.** The marking loop starts at 1, so it never examines
them — original zeros there would be missed entirely.

**Applying the border before the inner cells.** Destroys the markers mid-read.

**Applying the border from the markers instead of the flags.** The border's fate is decided by
`firstRowHasZero` / `firstColHasZero`, not by whatever marker values ended up there.

**Omitting the inner-cell apply pass.** It's easy to write the marking pass and the border pass and
think you're done — but nothing has yet read the markers to clear the body of the matrix. **Four
passes:** pre-scan, mark, apply inner, apply border. Count them before submitting.

---

### 🔴 Row/column mix-ups — the dominant failure mode here

Almost every bug in this problem is a row/column confusion rather than a logic error. The algorithm
is simple; keeping track of which index means what is not. The three concrete forms:

**Swapped loop bounds in the pre-scan.**

```java
for (int j = 0; j < matrix.length; j++)      // 🚩 matrix.length is the ROW count
    if (matrix[0][j] == 0) ...               //    but j indexes columns
```

Scanning **row 0** means `j` runs to `matrix[0].length`. Scanning **column 0** means `i` runs to
`matrix.length`. Reversed, that's an out-of-bounds crash on a non-square matrix, or a silently
missed cell.

**Swapped border blocks.** `firstRowHasZero` means "row 0 contains a zero", so it clears **row 0** —
`matrix[0][j]` across all `j`. Writing `matrix[i][0]` there clears a column instead. The names read
ambiguously under pressure, which is exactly why they should be `firstRowHasZero` /
`firstColHasZero` rather than `zeroRow` / `zeroCol` — "zeroRow" can be parsed as either *"row 0 has
a zero"* or *"the row to zero"*.

**Mixed-up index names generally.** Use `i` for rows and `j` for columns everywhere, including in
single-dimension loops. `for (int i = 0; i < matrix[0].length; i++)` walking *columns* is where
these bugs start.

> **The discipline:** before writing any bound or index in a 2-D problem, say out loud whether the
> thing is a row or a column. Same class of error as mixing coordinates with distances in a
> binary-search-on-answer problem — it's a units mistake, and it's caught at naming time or not at
> all.

---

## 6. Dry run

```
input:       [[1, 1, 1],
              [1, 0, 1],
              [1, 1, 1]]
```

**Step 1** — pre-scan. Column 0 is `[1,1,1]` → `firstColHasZero = false`. Row 0 is `[1,1,1]` →
`firstRowHasZero = false`.

**Step 2** — mark, `i,j ≥ 1`. Only `matrix[1][1] == 0`, so set `matrix[1][0] = 0` and
`matrix[0][1] = 0`:

```
             [[1, 0, 1],
              [0, 0, 1],
              [1, 1, 1]]
```

**Step 3** — apply to inner cells:

| cell | `matrix[i][0]` | `matrix[0][j]` | zero it? |
|---|---|---|---|
| (1,1) | 0 | 0 | ✅ |
| (1,2) | 0 | 1 | ✅ (row marker) |
| (2,1) | 1 | 0 | ✅ (col marker) |
| (2,2) | 1 | 1 | no |

```
             [[1, 0, 1],
              [0, 0, 0],
              [1, 0, 1]]
```

**Step 4** — both flags false, so the border is untouched.

Final: `[[1,0,1],[0,0,0],[1,0,1]]` ✅

### A case where the flags matter — `[[0,1],[1,1]]`

Step 1: column 0 is `[0,1]` → `firstColHasZero = true`. Row 0 is `[0,1]` → `firstRowHasZero = true`.
Step 2 finds no inner zeros. Step 3 does nothing. Step 4 zeroes both border lines:

```
[[0,1],      →     [[0,0],
 [1,1]]             [0,1]]
```
✅ Without the pre-scan, that zero would have been invisible to every later step.

---

## 7. Edge cases

| case | note |
|---|---|
| single row `[[1,0,1]]` | `m = 1`; the marking loop never runs; `firstRowHasZero` carries it |
| single column | `n = 1`; same in the other dimension |
| `1 × 1` matrix `[[0]]` | both flags true; step 4 zeroes the one cell |
| no zeros at all | flags false, no markers set, matrix unchanged |
| all zeros | everything is doomed; all passes agree |
| zero only at `matrix[0][0]` | both flags true — this is exactly the collision case |
| zeros only in row 0 | `firstRowHasZero` true; columns get marked by step 2? **No** — step 2 skips row 0, so column markers for those zeros come from the row-0 scan being applied in step 4. Verify this on `[[1,0],[1,1]]`: expected `[[0,0],[1,0]]` |

⚠️ That last row is the subtle one. **Trace `[[1,0],[1,1]]` by hand** — a zero in row 0 must still
zero its *column*, and the mechanism for that is `matrix[0][j]` already being 0 when step 3 reads
it, which happens to work because the original zero *is* the marker.

---

## 8. Interview Q&A

**Q: Why can't you zero rows and columns as you find zeros?**
Written zeros are indistinguishable from original zeros, so the scan produces false positives and
cascades. Detection and application must be separate passes.

**Q: Why is it safe to overwrite the first row and column?**
Because if either contains a zero, it's already destined to be all zeros. Once you've recorded
*whether* it contains one, its cells carry no information you still need.

**Q: What's special about `matrix[0][0]`?**
It's in both the first row and the first column, so it would have to encode two independent facts.
One of them goes into a separate boolean.

**Q: Why must the border be zeroed last?**
It destroys the markers. Any inner cell still needing to read them must be handled first.

**Q: Can you use just one extra boolean instead of two?**
Yes — let `matrix[0][0]` stand for one of the two facts (say, row 0) and keep a single boolean for
column 0. The structure is the same; two booleans is clearer.

**Q: Is O(1) actually better here?**
Asymptotically yes, practically marginal — `m + n` booleans is small next to `m × n` cells. It's
asked because it tests whether you can reason about destructive in-place updates, not because the
memory matters.

---

## 9. Pattern

> **The first row and column are already destined to be all-zero if they contain a zero — so they
> are free storage for the markers.**

The transferable half: **when you need O(m+n) bookkeeping, look for a part of the input whose
contents you're allowed to destroy.** Same instinct as LC 138's interleaving (the list becomes its
own map) and LC 41's index-as-hash.

---

## 10. Takeaways

1. **Separate detection from application.** A pass that reads state can't also overwrite it.
2. **In-place marking is safe only where the data is already doomed.** State that argument before
   writing into the input.
3. **A cell that must mean two things needs one extra variable.** Capture the fact before the cell
   is corrupted.
4. **Order the passes by what each one destroys** — the pass that destroys the markers goes last.
5. **`boolean[]` over `HashSet<Integer>`** for dense, bounded index spaces.
6. **`i` for rows, `j` for columns**, in every loop including one-dimensional ones. Say which one a
   bound refers to *before* writing it — row/column confusion is the dominant failure mode here,
   and it's a units error caught at naming time or not at all.
7. **Count the passes: pre-scan, mark, apply inner, apply border.** Forgetting the third is easy
   because the other three all look like work.
8. **Write the O(m+n) version first**, state its cost, then improve it. That's the sequence an
   interviewer is scoring.