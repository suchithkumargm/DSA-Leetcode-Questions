# LC 37 — Sudoku Solver

**Difficulty:** Hard · **Sheets:** Striver SDE | Love Babbar 450
**Pattern:** Backtracking that returns a **boolean** — find one solution, then stop

---

## 1. Problem

Fill a partially completed 9×9 Sudoku board in place. Empty cells are `'.'`. Each digit `1`–`9` must
appear exactly once in every **row**, every **column**, and every **3×3 box**.

The input is `char[][]` — digits arrive as the characters `'1'`…`'9'`, not as ints. A solution is
guaranteed to exist and to be unique.

The method signature returns `void`: you mutate the board and the caller reads it.

---

## 2. The shape

Standard backtracking, with one structural difference from the combination problems:

> ### 🔑 The fact
> **Enumerate-all returns `void`; find-one returns `boolean`.**
>
> LC 39/40 want *every* combination, so there's nothing to stop for — you record and keep going.
> Sudoku wants **one** solution and must **freeze it**. The boolean does two jobs:
>
> 1. tells the caller whether its placement led to a complete solution, so it knows whether to undo
     >    and try the next digit;
> 2. propagates `true` all the way up on success, so every frame returns immediately — no further
     >    search, and crucially **no undoing**, which would unwind the solved board on the way out.

Per cell: if filled, move on. If empty, try `'1'`…`'9'`; for each that's legal, place it, recurse,
and on failure **reset the cell to `'.'`** before trying the next.

### You never walk backwards yourself

A natural first description is *"if no digit fits, go back and increment the previous cell."* **You
never write that code.** When a cell exhausts all nine digits it returns `false`, which drops you
back into the *previous* cell's digit loop, which continues to its next candidate naturally. The
call stack does the walking.

---

## 3. Solution

```java
class Solution {
    public void solveSudoku(char[][] board) {
        solve(board);
    }

    private boolean solve(char[][] board) {
        for (int row = 0; row < 9; row++) {
            for (int col = 0; col < 9; col++) {
                if (board[row][col] == '.') {

                    for (char ch = '1'; ch <= '9'; ch++) {
                        if (isSafe(board, row, col, ch)) {
                            board[row][col] = ch;
                            if (solve(board)) return true;   // solved downstream → freeze
                            board[row][col] = '.';           // undo and try the next digit
                        }
                    }
                    return false;        // all nine failed → this branch is dead
                }
            }
        }
        return true;                     // no empty cell left → board is complete
    }

    private boolean isSafe(char[][] board, int row, int col, char ch) {
        int startRow = (row / 3) * 3;
        int startCol = (col / 3) * 3;

        for (int i = startRow; i < startRow + 3; i++)          // box
            for (int j = startCol; j < startCol + 3; j++)
                if (board[i][j] == ch) return false;

        for (int i = 0; i < 9; i++)                            // row
            if (board[row][i] == ch) return false;

        for (int i = 0; i < 9; i++)                            // column
            if (board[i][col] == ch) return false;

        return true;
    }
}
```

### Where each `return` goes — the three that matter

| statement | position | meaning |
|---|---|---|
| `return true` after `solve(board)` | inside the digit loop | a solution exists downstream; stop everything |
| `return false` | **after** the digit loop, **inside** the empty-cell branch | nine failures ⇒ this branch can't work; let the caller try its next digit |
| `return true` | after both outer loops | no empty cell was found ⇒ complete |

Get the second one's placement wrong — outside the `if`, or inside the digit loop — and you either
abandon the search after one digit or never report failure at all.

### `for (char ch = '1'; ch <= '9'; ch++)`

`char` is integral in Java, so this walks the digit characters directly. No `Character.forDigit(i, 10)`
conversion, and no method call per candidate.

### No `else` needed around the undo

`if (solve(board)) return true;` already exits on success, so the line after it only runs on failure.
An `else` adds a nesting level for nothing.

---

## 4. The box index — `(row / 3) * 3`

The only non-obvious arithmetic in the problem. Given a cell, find the top-left corner of its 3×3
block:

```
row:        0  1  2  3  4  5  6  7  8
row / 3:    0  0  0  1  1  1  2  2  2      ← which block
(row/3)*3:  0  0  0  3  3  3  6  6  6      ← the block's starting row
```

**Integer division floors to the block number; multiplying back scales it to the block's origin.**
`(5/3)*3 = 1*3 = 3`, not 5, because division discards the remainder.

> **This "round down to a multiple" trick appears whenever you need "which block does this index fall
> in".** Worth recognising on sight rather than deriving each time — and note it's the *same*
> operation as unflattening an index (§6), just used in the other direction.

**You only need the start.** The end is `start + 3`, so there's no second computation.

---

## 5. Common pitfalls

### 🚩 Checking "the current cell" instead of the box

The three constraints are **row, column, box**. The current cell is where you're *placing* — there's
nothing to validate against it. Forgetting the box is the single most common miss, and the board will
still look plausible for a while.

### 🚩 Not resetting the cell before returning false

```java
for (char ch = '1'; ch <= '9'; ch++) {
    if (isSafe(...)) {
        board[row][col] = ch;
        if (solve(board)) return true;
        // 🚩 missing: board[row][col] = '.';
    }
}
return false;
```

A failed digit left on the board poisons every later attempt — including the caller's retry, which
now sees a cell that isn't empty and skips it entirely.

### 🚩 Undoing on success

The undo must happen **only** on the failure path. If `true` propagates up through frames that reset
their cells, you unwind the solved board and return an empty grid.

### 🚩 `return false` in the wrong place

Outside the empty-cell branch, it fires after the very first cell. Inside the digit loop, it fires
after the first failing digit without trying the rest.

### 🚩 `void` instead of `boolean`

Then there's no way to distinguish "this placement worked out" from "it didn't", and no way to stop
once solved. (Workable with a field flag, but the boolean is the natural form.)

### 🚩 `int[][]` instead of `char[][]`, or comparing against `1`–`9` as ints

LeetCode hands you `char[][]`. Fighting the types costs more than it saves.

### 🚩 Writing the walk-backwards logic by hand

See §2 — the call stack does it.

---

## 6. Improving the runtime

Both are independent; the first is five lines and gets most of the win.

### 6a. Stop rescanning from (0,0)

Every recursive call walks the board from the start looking for an empty cell — **O(81) per level,
81 levels deep.** Pass the position down as a flat index instead:

```java
private boolean solve(char[][] board, int pos) {
    if (pos == 81) return true;                               // past the last cell → solved
    int row = pos / 9, col = pos % 9;

    if (board[row][col] != '.') return solve(board, pos + 1);  // skip filled

    for (char ch = '1'; ch <= '9'; ch++) {
        if (isSafe(board, row, col, ch)) {
            board[row][col] = ch;
            if (solve(board, pos + 1)) return true;
            board[row][col] = '.';
        }
    }
    return false;
}
```

`pos / 9` and `pos % 9` unflatten the index — the same divide-and-mod pair as §4, used to *recover*
coordinates rather than to find a block.

### 6b. Bitmasks — `isSafe` from O(27) to O(1)

Keep `rows[9]`, `cols[9]`, `boxes[9]` as int bitmasks, where bit `d` is set if digit `d+1` is already
used there.

```java
int bit = 1 << (ch - '1');
int box = (row / 3) * 3 + col / 3;

boolean unsafe = ((rows[row] | cols[col] | boxes[box]) & bit) != 0;
```

> ⚠️ **Parenthesise.** `&` binds tighter than `|` in Java, so `a | b | c & bit` is
> `a | b | (c & bit)` — not what you want.

Place: OR the bit into all three. Undo: XOR it out of all three.

**The cost:** you now have state that must stay in sync with the board through *every* place and
undo, plus a seeding pass over the given digits. That's where the bugs live. It's the real
optimisation, but take 6a first.

### 6c. Heuristic — most-constrained cell first

Instead of the next empty cell in scan order, pick the empty cell with the **fewest legal
candidates**. Fails fast on dead branches and prunes enormously on hard boards. Standard in real
solvers; overkill for LeetCode.

---

## 7. Complexity

| | |
|---|---|
| **Time** | **O(9^m)** worst case, m = number of empty cells |
| **Space** | **O(m)** recursion stack — bounded by 81, so effectively O(1) |

Each empty cell has up to 9 candidates and there are up to 81 of them, so the bound is astronomical
on paper. **In practice the constraint checks prune almost everything** — a valid Sudoku has a unique
solution and the row/column/box rules eliminate most branches within a few levels.

> The honest answer: *"Exponential, O(9^m) in the number of empty cells. The pruning from the three
> constraints makes it fast in practice, and on a fixed 9×9 board the input size is constant anyway —
> so this is really about how aggressively you prune."*

The board itself is mutated in place, so there's no auxiliary O(n²) storage.

---

## 8. Dry run — the mechanism on a small fragment

Suppose cells `(0,0)` and `(0,1)` are empty and `isSafe` allows only a narrow set:

```
solve() at (0,0)
  try '1' → safe → place → recurse
      solve() at (0,1)
        try '1' → unsafe (same row)
        try '2' → unsafe (same box)
        ... all nine fail
        return false                      ← (0,1) is a dead end
  back in (0,0): undo → board[0][0] = '.'
  try '2' → safe → place → recurse
      solve() at (0,1)
        try '1' → safe → place → recurse
            ... eventually returns true
        return true
  return true                             ← propagates up, no undo
```

**The two things to see:** `(0,0)` never reaches backwards — it simply continues its own loop after
`false` comes back. And once `true` starts propagating, **no frame undoes anything.**

---

## 9. Edge cases

| case | behaviour |
|---|---|
| already-complete board | both outer loops finish, `return true` immediately |
| empty board (all `'.'`) | solves it — many solutions exist, you get the first found |
| board with one empty cell | one digit survives `isSafe`, placed immediately |
| unsolvable board | returns `false`; the board is restored to its original state by the undos |
| invalid given digits | undefined — `isSafe` only validates *placements*, never the pre-filled cells |

> That last row is worth knowing: this solver assumes the given board is valid. Checking the input
> is LC 36 (Valid Sudoku), a separate problem.

---

## 10. Interview Q&A

**Q: Why does the recursion return `boolean` rather than `void`?**
You need one solution and must stop. The boolean tells the caller whether to undo and continue, and
propagates success upward without unwinding.

**Q: What's different from Combination Sum?**
That enumerates *all* answers, so it's `void` and always backtracks. This finds *one* and must
freeze it — hence boolean, and hence no undo on the success path.

**Q: Where does the "go back to the previous cell" logic live?**
Nowhere. Returning `false` drops into the previous frame's digit loop, which continues naturally.

**Q: How do you find the 3×3 box?**
`(row/3)*3, (col/3)*3` — integer division floors to the block, multiplying back gives its origin.

**Q: Complexity?**
O(9^m) in the number of empty cells, with heavy pruning in practice. O(m) stack, bounded by 81.

**Q: How would you speed it up?**
Pass the cell position down instead of rescanning; bitmasks for O(1) constraint checks; and
most-constrained-cell-first ordering.

**Q: How would you validate a board rather than solve it?**
LC 36 — one pass with three sets of seen-digits, no recursion.