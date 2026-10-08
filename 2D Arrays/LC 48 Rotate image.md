# LC 48 — Rotate Image

**Difficulty:** Medium · **Pattern:** Decompose a rotation into two simple in-place reflections

---

## 1. Problem

Given an `n × n` matrix, rotate it **90° clockwise, in place**. You may not allocate another matrix.

```
[[1,2,3],        [[7,4,1],
 [4,5,6],   →     [8,5,2],
 [7,8,9]]         [9,6,3]]
```

---

## 2. Where each element goes

Write the index mapping down before touching code:

> **`matrix[i][j]` → `matrix[j][n-1-i]`**

Check it on the example: `matrix[0][0] = 1` lands at `matrix[0][2]` ✅. `matrix[2][0] = 7` lands at
`matrix[0][0]` ✅.

Equivalently, read it row-wise: **row `i` becomes column `n-1-i`.** The top row becomes the right
column, which is why the result's first column reads `7,8,9` bottom-up.

### Why you can't just assign

`matrix[j][n-1-i] = matrix[i][j]` overwrites a value that hasn't moved yet. Direct assignment
requires rotating **four cells at once in a cycle** (§5), which is correct but fiddly to index.

The easier route is to find two operations that compose into the rotation, each of which is a
simple swap.

---

## 3. The decomposition

> ### 🔑 The fact
> **Transpose, then reverse each row, equals a 90° clockwise rotation.**

**Transpose** reflects across the main diagonal: `matrix[i][j] ↔ matrix[j][i]`.

```
[[1,2,3],        [[1,4,7],
 [4,5,6],   →     [2,5,8],
 [7,8,9]]         [3,6,9]]
```

**Reverse each row** reflects across the vertical axis:

```
[[1,4,7],        [[7,4,1],
 [2,5,8],   →     [8,5,2],
 [3,6,9]]         [9,6,3]]
```

Which is the answer.

**Why two reflections make a rotation:** composing reflections across two axes gives a rotation
through twice the angle between them. The main diagonal and the vertical axis are 45° apart, so the
composition is a 90° rotation. That's the geometric reason, and it's what lets you derive the
counter-clockwise variant instead of memorising it (§6).

Each half is a swap of two cells, so both are trivially in place.

---

## 4. Solution

```java
class Solution {
    public void rotate(int[][] matrix) {
        int n = matrix.length;

        // 1. transpose — swap across the main diagonal
        for (int i = 0; i < n; i++) {
            for (int j = i + 1; j < n; j++) {        // upper triangle only
                int temp = matrix[i][j];
                matrix[i][j] = matrix[j][i];
                matrix[j][i] = temp;
            }
        }

        // 2. reverse each row
        for (int i = 0; i < n; i++) {
            reverseRow(matrix[i]);
        }
    }

    private void reverseRow(int[] row) {
        int start = 0, end = row.length - 1;
        while (start < end) {
            int temp = row[start];
            row[start] = row[end];
            row[end] = temp;
            start++;
            end--;
        }
    }
}
```

**O(n²) time · O(1) space.** Both optimal — every element must move, so O(n²) is a lower bound, and
in-place forbids auxiliary storage.

### ⚠️ `j = i + 1` is the line that matters

The inner loop must visit the **upper triangle only**. A transpose swaps `(i,j)` with `(j,i)`, so
each pair must be visited **once** — not once from each side.

Starting at `j = 0` swaps every pair twice, which undoes the first swap and leaves the matrix
**completely unchanged**. The symptom is "my code does nothing", which looks nothing like an index
bug.

`j = i + 1` also skips the diagonal itself (`i == j`), where a swap would be a no-op anyway.

### Use `n` everywhere

The matrix is guaranteed square, so `matrix.length` and `matrix[0].length` are equal and either
"works". But writing `matrix[0].length` — the **column** count — as the bound for a diagonal walk
hides the fact that the algorithm *depends on squareness*. Hoist `int n = matrix.length;` and use it
throughout.

### `reverseRow(matrix[i])` mutates the real row

Java's 2-D arrays are arrays of row references, so `matrix[i]` hands the helper the actual row
object, not a copy. The mutation lands in the matrix. (Had this been a flattened 1-D array or a
language with value-semantics rows, you'd need index arithmetic instead.)

---

## 5. Alternative — four-way cyclic swap

One pass, rotating four cells at a time around the centre. Worth knowing because it shows you can
work the index mapping directly.

```java
public void rotate(int[][] matrix) {
    int n = matrix.length;

    for (int layer = 0; layer < n / 2; layer++) {
        int first = layer, last = n - 1 - layer;

        for (int i = first; i < last; i++) {
            int offset = i - first;
            int top = matrix[first][i];                      // save top

            matrix[first][i]          = matrix[last - offset][first];  // left  → top
            matrix[last - offset][first] = matrix[last][last - offset]; // bottom → left
            matrix[last][last - offset]  = matrix[i][last];             // right  → bottom
            matrix[i][last]           = top;                            // top    → right
        }
    }
}
```

**The idea:** peel the matrix into concentric rings. Within a ring, each position belongs to a cycle
of four, and one temp variable rotates all four.

Same O(n²)/O(1). **Prefer the transpose-reverse version** — it's four lines of obvious code versus
four lines of index arithmetic that are easy to get wrong under pressure, and it generalises to the
variants in §6 by changing one step.

> ⚠️ Note the inner loop is `i < last`, not `i <= last`. The last position of each ring is the first
> position of the cycle that's already been handled — including it rotates the ring one step too far.

---

## 6. The variants — derive, don't memorise

Each is "two reflections", with one of them changed:

| rotation | step 1 | step 2 |
|---|---|---|
| **90° clockwise** | transpose (main diagonal) | reverse each **row** |
| **90° counter-clockwise** | transpose (main diagonal) | reverse each **column** |
| **90° counter-clockwise** (alt) | reverse each **row** | transpose |
| **180°** | reverse each row | reverse each column |

**Sanity-check the counter-clockwise one** rather than trusting the table. After transposing,
`[[1,4,7],[2,5,8],[3,6,9]]`; reversing each column (swap row 0 with row 2) gives
`[[3,6,9],[2,5,8],[1,4,7]]` — and `3` is top-left, which is where the top-right element belongs
after a counter-clockwise turn ✅

**Reversing columns** means swapping whole rows: `matrix[i] ↔ matrix[n-1-i]`, which in Java is a
reference swap — cheaper than reversing each row element by element.

**180°** can also be done as a single pass swapping `matrix[i][j]` with `matrix[n-1-i][n-1-j]` over
half the matrix.

---

## 7. Common pitfalls

- **`j = 0` in the transpose loop** → every pair swapped twice → matrix unchanged. The single most
  common bug here.
- **Allocating a result matrix.** Explicitly forbidden; `O(n²)` space fails the point of the
  problem.
- **Direct assignment via the index formula** without a four-cycle — overwrites values that haven't
  moved yet.
- **Reversing columns when you meant rows** (or vice versa) → you've rotated counter-clockwise.
  Check one corner before submitting: after a clockwise turn, the **old bottom-left** must be at
  **top-left**.
- **Using `matrix[0].length` as the bound** for a diagonal walk. Works only because the matrix is
  square, and obscures that dependency.
- **`i <= last` in the cyclic version** → rotates one position too many per ring.
- **Assuming it works for non-square matrices.** It doesn't — an `m × n` transpose has different
  dimensions, so in-place rotation is impossible for a rectangle. Say so if asked.

---

## 8. Dry run — `[[1,2,3],[4,5,6],[7,8,9]]`

### Transpose, upper triangle only

| i | j | swap | matrix after |
|---|---|---|---|
| 0 | 1 | `[0][1]=2 ↔ [1][0]=4` | `[[1,4,3],[2,5,6],[7,8,9]]` |
| 0 | 2 | `[0][2]=3 ↔ [2][0]=7` | `[[1,4,7],[2,5,6],[3,8,9]]` |
| 1 | 2 | `[1][2]=6 ↔ [2][1]=8` | `[[1,4,7],[2,5,8],[3,6,9]]` |
| 2 | — | inner loop doesn't run (`j` starts at 3) | — |

Three swaps for a 3×3 — the upper triangle's size, `n(n-1)/2`. If you counted nine, you ran the
whole matrix and undid your own work.

### Reverse each row

```
[1,4,7] → [7,4,1]
[2,5,8] → [8,5,2]
[3,6,9] → [9,6,3]
```

Result `[[7,4,1],[8,5,2],[9,6,3]]` ✅

---

## 9. Edge cases

| case | expected | ✓ |
|---|---|---|
| `n = 1` | unchanged | ✅ transpose loop never runs; row reverse is a no-op |
| `n = 2` | corners cycle | ✅ one swap, then two row reversals |
| odd `n` | centre element fixed | ✅ it's on the diagonal, and the middle of its own row |
| all identical values | unchanged | ✅ |
| negative values | fine | ✅ nothing compares values |

---

## 10. Interview Q&A

**Q: Why does transpose-then-reverse give a rotation?**
Composing two reflections gives a rotation through twice the angle between their axes. The main
diagonal and the vertical axis are 45° apart, so the composition is 90°.

**Q: Why `j = i + 1`?**
A transpose swaps each pair once. Running the full inner range swaps each pair twice and leaves the
matrix unchanged.

**Q: Counter-clockwise?**
Transpose, then reverse each **column** (equivalently, reverse rows *then* transpose).

**Q: Can you do it in one pass?**
Yes — the four-way cyclic swap, §5. Same complexity, trickier indexing. The two-step version is
easier to get right and easier to adapt.

**Q: Does this work for a non-square matrix?**
No. An `m × n` matrix rotates to `n × m`, so the result doesn't fit in the original allocation.
In-place rotation requires a square.

**Q: Could you beat O(n²)?**
No. Every element changes position, so every element must be touched.

**Q: How do you verify a rotation quickly?**
Check one corner. After a clockwise turn the old **bottom-left** sits at **top-left**. One cell
catches the clockwise/counter-clockwise mix-up instantly.

---

## 11. Pattern

> **A rotation is two reflections — transpose, then reverse each row.**

The transferable half: **when a transformation is awkward to apply directly in place, look for two
simpler transformations that compose into it.** Each reflection is a plain two-cell swap; the
rotation on its own is a four-cell cycle. Decomposing turned a fiddly index problem into two obvious
loops.

---

## 12. Takeaways

1. **Write the index mapping first** — `matrix[i][j] → matrix[j][n-1-i]` — then look for a
   decomposition rather than implementing it directly.
2. **Transpose + reverse rows = clockwise.** Transpose + reverse columns = counter-clockwise.
3. **`j = i + 1`** — upper triangle only. `j = 0` swaps everything twice and does nothing.
4. **Count your swaps when debugging a transpose:** `n(n-1)/2`, not `n²`.
5. **Hoist `int n = matrix.length`** and use it for both dimensions; it documents the squareness
   dependency.
6. **Verify with one corner.** Old bottom-left → top-left for clockwise.
7. **Java 2-D arrays are arrays of row references**, so passing `matrix[i]` to a helper mutates the
   real row.
8. **Decomposition over direct manipulation** when the direct version needs a multi-cell cycle.