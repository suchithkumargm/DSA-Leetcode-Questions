# LC 46 — Permutations

**Difficulty:** Medium
**Topic:** Recursion + Backtracking
**Sheets:** NeetCode 150 · Striver A2Z · Love Babbar 450 · Apna College

---

## Problem

Given an array `nums` of **distinct** integers, return *all possible permutations* in any order.

```
Input:  nums = [1,2,3]
Output: [[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]
```

Constraints: `1 <= nums.length <= 6`, all integers distinct.

> The tiny `n <= 6` is a hint in itself. `6! = 720`. When constraints are this small, the setter is telling you an exponential/factorial solution is expected. **Small n → think brute-force recursion.**

---

## 1. Building the intuition

The insight that unlocks it — think in terms of **slots to fill**, not elements to pick:

```
slot 0    slot 1    slot 2
  ?         ?         ?
```

For slot 0, I can place any of `1, 2, 3`.
Whatever I place there is now used up, so slot 1 has 2 remaining choices.
Slot 2 then has 1.

Drawn as a tree:

```
                    [ ]
        ┌────────────┼────────────┐
       1            2            3
    ┌───┴───┐    ┌───┴───┐    ┌───┴───┐
   12      13   21      23   31      32
    │       │    │       │    │       │
  123     132  213     231  312     321
```

Each **root-to-leaf path** is one permutation. There are 6 leaves = 6 answers.

This is the standard backtracking shape:

```
for each choice:
    make the choice
    recurse
    undo the choice     ← the "backtracking" part
```

---

## 2. The key difference from Subsets / Combination Sum

This tripped me up initially, so it's worth stating loudly:

| | Subsets (LC 78), Combination Sum (LC 39) | **Permutations (LC 46)** |
|---|---|---|
| Does order matter? | ❌ `[1,2] == [2,1]` | ✅ `[1,2] != [2,1]` |
| Loop start | `for (i = start; ...)` | `for (i = 0; ...)` — **all** elements |
| Prevents reuse via | `start` index bound | ❌ can't — need `used[]` or swapping |

**Why the `start` trick fails here:** `start` exists to enforce "only look forward," which kills duplicate *orderings*. But permutations are exactly the problem where different orderings are different answers. So we must loop over all `n` indices at every level — and therefore need a *separate* mechanism to skip already-placed elements.

That mechanism is the whole problem.

---

## 3. Approach 1 — `contains()` check (brute / naive)

Simplest thing that works. Since all numbers are distinct, just ask the running list whether it already holds this number.

```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> ans = new ArrayList<>();
        findPermutations(new ArrayList<>(), nums, ans);
        return ans;
    }

    private void findPermutations(List<Integer> perm, int[] nums, List<List<Integer>> ans) {
        if (perm.size() == nums.length) {
            ans.add(new ArrayList<>(perm));
            return;
        }
        for (int i = 0; i < nums.length; i++) {
            if (perm.contains(nums[i])) continue;   // ← the skip
            perm.add(nums[i]);
            findPermutations(perm, nums, ans);
            perm.remove(perm.size() - 1);
        }
    }
}
```

**Complexity**
- Time: `O(n! × n × n)` — `n!` leaves × `O(n)` copy, and `contains()` costs `O(n)` at every node
- Space: `O(n)` auxiliary

**Verdict:** passes on LeetCode, but the `O(n)` linear scan on every single node is wasteful and an interviewer will flag it. Also silently **breaks if duplicates are allowed** (LC 47) — `contains` can't distinguish "the same value at a different index."

---

## 4. Approach 2 — `boolean[] used` ⭐ **THE ONE TO MEMORISE**

Replace the `O(n)` scan with an `O(1)` array lookup.

```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> ans = new ArrayList<>();
        boolean[] used = new boolean[nums.length];
        findPermutations(new ArrayList<>(), used, nums, ans);
        return ans;
    }

    private void findPermutations(List<Integer> perm, boolean[] used,
                                  int[] nums, List<List<Integer>> ans) {
        // base case: n slots filled
        if (perm.size() == nums.length) {
            ans.add(new ArrayList<>(perm));   // COPY, not the reference
            return;
        }

        for (int i = 0; i < nums.length; i++) {
            if (used[i]) continue;

            used[i] = true;                   // ┐
            perm.add(nums[i]);                // ├─ make the choice
                                              // ┘
            findPermutations(perm, used, nums, ans);

            perm.remove(perm.size() - 1);     // ┐ undo — BOTH of them
            used[i] = false;                  // ┘
        }
    }
}
```

**Complexity**
- Time: `O(n! × n)`
- Space: `O(n)` auxiliary — recursion stack + `used[]` + `perm`

**Why this is the default answer:** it's the same template as subsets/combination sum, it's `O(1)` to check, and — most importantly — **it extends directly to LC 47** (duplicates). Write this one in an interview.

### Mistakes I actually made writing this

**a) Forgot the base case entirely.**
The recursion still terminated (eventually all `used[i]` are `true`, loop skips everything, frame falls off the end) — but there was no designated moment to say *"this is a complete permutation, record it."* Without the base case there's nowhere correct to put `ans.add`.

**b) Put `ans.add(...)` inside the loop, after the undo.**

```java
perm.remove(perm.size() - 1);
used[i] = false;
ans.add(new ArrayList<>(perm));   // ✗ WRONG — two bugs
```

- Snapshots the state *after* backing out — records what you just discarded
- Runs at every depth, so collects partial lists of size 0, 1, 2 — not just complete ones

The snapshot must happen at the **leaf**, before any undo.

**c) `count++` mutating a parameter.**

```java
count++;
findPermutations(count, ...);   // ✗ increment persists into the next loop iteration
findPermutations(count + 1, ...); // ✓ scoped to the child, unwinds for free
```

**Rule:** never mutate a depth parameter — pass `param + 1`. (Better still: `perm.size()` already *is* the depth, so drop the parameter.)

**d) Undoing only one of the two things.**
`perm.remove(...)` **and** `used[i] = false` are a pair. Miss the second and one branch works while the rest starve.

---

## 5. Approach 3 — In-place swapping (space-optimal)

Idea: instead of tracking *which* elements are used, **physically move** the chosen element into the current slot. The array's own layout becomes the bookkeeping.

- Everything left of `index` = frozen, already assigned
- Everything at/right of `index` = still available

```java
class Solution {
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> ans = new ArrayList<>();
        findPermutations(0, nums, ans);
        return ans;
    }

    private void findPermutations(int index, int[] nums, List<List<Integer>> ans) {
        if (index == nums.length) {
            List<Integer> temp = new ArrayList<>();
            for (int num : nums) temp.add(num);
            ans.add(temp);
            return;
        }

        for (int i = index; i < nums.length; i++) {   // ← starts at index, not 0
            swap(nums, index, i);
            findPermutations(index + 1, nums, ans);
            swap(nums, index, i);                     // undo (self-inverse)
        }
    }

    private void swap(int[] nums, int a, int b) {
        int t = nums[a]; nums[a] = nums[b]; nums[b] = t;
    }
}
```

**Complexity**
- Time: `O(n! × n)` — unchanged
- Space: `O(n)` for the recursion stack **only**. No `used[]`, no running list. Constant-factor win, same asymptotic class.

### Why `i = index` is correct here (but wasn't in Approach 2)

In Approach 2, `perm` was a separate list and `nums` never moved — so index `i` always referred to the same value, and skipping forward would have lost permutations. Here the array is rearranged so that all *unused* elements physically occupy `[index, n)`. Looping from `index` covers exactly the available ones. Different mechanism, so a different loop bound is right.

### Notes

- `swap(index, index)` on the first iteration is a **no-op** — but it's a real branch: the "keep this element where it is" choice. Remove it and you lose `[1,2,3]`.
- The undo is the *same swap again*. Swapping twice = identity, so the undo is free.
- **Invariant:** every call returns the array exactly as it received it.

---

## 6. Dry run — swap approach on `[1,2,3]`

Watch the rightmost column. After every level-0 undo it must return to `[1,2,3]`.

```
call(index=0)                                  nums = [1,2,3]
│
├─ i=0: swap(0,0)                              [1,2,3]   no-op: keep 1 in front
│   ├─ call(index=1)
│   │  ├─ i=1: swap(1,1)                       [1,2,3]
│   │  │   └─ call(index=2)
│   │  │      └─ i=2: swap(2,2)                [1,2,3]
│   │  │         └─ call(index=3) → 3==3       ✅ ADD [1,2,3]
│   │  │         undo swap(2,2)                [1,2,3]
│   │  ├─ undo swap(1,1)                       [1,2,3]
│   │  ├─ i=2: swap(1,2)                       [1,3,2]   3 moved into slot 1
│   │  │   └─ call(index=2)
│   │  │      └─ i=2: swap(2,2)                [1,3,2]
│   │  │         └─ call(index=3)              ✅ ADD [1,3,2]
│   │  └─ undo swap(1,2)                       [1,2,3]   restored
└──┴─ undo swap(0,0)                           [1,2,3]
│
├─ i=1: swap(0,1)                              [2,1,3]   2 into slot 0
│   ├─ call(index=1)
│   │  ├─ i=1: swap(1,1)                       [2,1,3]
│   │  │      └─ ... call(index=3)             ✅ ADD [2,1,3]
│   │  ├─ undo                                 [2,1,3]
│   │  ├─ i=2: swap(1,2)                       [2,3,1]
│   │  │      └─ ... call(index=3)             ✅ ADD [2,3,1]
│   │  └─ undo swap(1,2)                       [2,1,3]
└──┴─ undo swap(0,1)                           [1,2,3]   restored
│
├─ i=2: swap(0,2)                              [3,2,1]   3 into slot 0
│   ├─ call(index=1)
│   │  ├─ i=1: swap(1,1)                       [3,2,1]
│   │  │      └─ ... call(index=3)             ✅ ADD [3,2,1]
│   │  ├─ undo                                 [3,2,1]
│   │  ├─ i=2: swap(1,2)                       [3,1,2]
│   │  │      └─ ... call(index=3)             ✅ ADD [3,1,2]
│   │  └─ undo swap(1,2)                       [3,2,1]
└──┴─ undo swap(0,2)                           [1,2,3]   restored
│
loop ends → return
```

**Result:** `[[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,2,1], [3,1,2]]` — 6 = 3! ✅

### Three takeaways from the trace

1. **The restore invariant.** After each level-0 undo the array is back to `[1,2,3]`. Delete the undo line and `i=1` would start from `[1,3,2]` → garbage output.
2. **No-op swaps are real branches.** `swap(i,i)` = "leave it in place." Skipping them loses permutations.
3. **Output is NOT lexicographic.** Last two are `[3,2,1]` then `[3,1,2]`. The level-0 swap disturbs the relative order of the remaining elements. The `used[]` version *does* emit in sorted order (given sorted input) because it scans `i` from 0 every time.

---

## 7. Complexity intuition — the reusable recipe

I kept guessing `2^n` out of habit. That's the **subsets** answer (each element: in or out → `2 × 2 × ... = 2^n`). Permutations have a different branching shape.

### Counting the leaves

```
level 0 (0 placed)  →  n   choices
level 1 (1 placed)  →  n-1 choices   (one used up)
level 2 (2 placed)  →  n-2 choices
...
level n-1           →  1   choice
                       ─────────────
       multiply:  n × (n-1) × ... × 1  =  n!
```

For n=3 → 6 leaves → 6 answers. ✅

### The 4-step recipe for ANY backtracking problem

1. Draw the tree 2–3 levels deep. **How many branches per node?**
2. Multiply branches down to the max depth → **number of leaves**
3. Multiply by the **work done at each leaf** (usually the `O(n)` copy)
4. **Space** = recursion depth + helper arrays. *Exclude the output.*

> Internal nodes always sum to less than the leaf level in these trees, so they never change the answer. Only count the leaves.

### Applied

| Problem | Branches/node | Leaves | Leaf work | **Time** |
|---|---|---|---|---|
| Subsets (78) | 2 (in/out) | `2^n` | `O(n)` copy | `O(2^n × n)` |
| **Permutations (46)** | n, n-1, n-2… | `n!` | `O(n)` copy | **`O(n! × n)`** |
| Combination Sum (39) | ~n, unbounded depth | — | `O(k)` | `~O(2^target × k)` |
| N-Queens (51) | n, n-1… | ≤ `n!` | `O(n)` check | `O(n! × n)` |

### Space breakdown for LC 46

| What | Size |
|---|---|
| Recursion call stack | `n` frames deep → `O(n)` |
| `used[]` | `O(n)` |
| `perm` running list | ≤ n → `O(n)` |
| **Auxiliary total** | **`O(n)`** |
| *(Output `ans`, if asked)* | `O(n! × n)` |

**Interview phrasing:** *"Auxiliary space is O(n) — the recursion stack and the used array. The output itself is O(n! × n), but that's required by the problem, not something I chose."*

---

## 8. Can it be optimised further?

**Time: no.** The output alone is `n!` lists of `n` integers — you must *write* `O(n! × n)` values. Any correct algorithm does at least that. **`O(n! × n)` is a hard lower bound.**

Saying this out loud in an interview is worth a point on its own — it shows you know when to stop optimising.

**Space: yes, constants only.** Swapping removes `used[]` and the running list, but the recursion stack keeps it at `O(n)`.

---

## 9. Which version to use when

| | `used[]` (Approach 2) | Swap (Approach 3) |
|---|---|---|
| Extends to LC 47 (duplicates) | ✅ | ❌ |
| Lexicographic output | ✅ (sorted input) | ❌ |
| Extra space beyond stack | `O(n)` | none |
| Fits the standard template | ✅ | ✗ different shape |

**Strategy:** write the `used[]` version, then *mention* the swap version as a space optimisation. That usually earns the point without needing to write it — and leaves you set up for the "now handle duplicates" follow-up.

---

## 10. Java gotchas hit in this problem

**`List.remove(int)` vs `remove(Object)`** — the recurring one.

```java
perm.remove(perm.size() - 1);   // ✓ removes by INDEX — what we want
perm.remove(nums[i]);           // ✗ nums[i] is int → resolves to remove(INDEX)
perm.remove(Integer.valueOf(x)); // removes by VALUE (needs the boxing)
```

With an `int` argument, Java picks `remove(int index)` — silently deletes the wrong element or throws `IndexOutOfBoundsException`.

**Copy at the leaf, always.**

```java
ans.add(new ArrayList<>(perm));  // ✓ snapshot
ans.add(perm);                   // ✗ shares the reference — every entry ends up empty
```

**`Arrays.asList` on `int[]` is a trap.**

```java
Arrays.asList(nums)              // ✗ gives List<int[]> of size 1
for (int num : nums) temp.add(num);  // ✓ manual loop, autoboxes correctly
```

**Never mutate a depth parameter** — pass `count + 1`, don't write `count++`.

---

## 11. Next problems

| Problem | What's new |
|---|---|
| **LC 47 — Permutations II** | Duplicates allowed. Sort first, then `if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;` — reason out why it's `!used[i-1]` and not `used[i-1]`. **Must use the `used[]` version**: swapping destroys the sorted order the guard depends on. |
| **LC 31 — Next Permutation** | Not backtracking at all — an `O(n)` in-place trick. Good contrast. |
| **LC 60 — Permutation Sequence** | Find the k-th directly using factorial number system; don't generate all. |
| **LC 78 / 90 — Subsets I & II** | ✅ done — compare the `start` bound against `used[]` |
| **LC 39 / 40 — Combination Sum I & II** | ✅ done — same `i > start` dedup family as LC 47 |
| **LC 51 — N-Queens** | The heavyweight application of this exact template |

---

## Quick revision card

- **Slots, not elements.** Fill slot 0, then slot 1, …
- Order matters ⇒ **loop `i` from 0 over all elements**; `start` doesn't work here
- Skip used elements via `boolean[] used` (`O(1)`) or by swapping
- Base case: `perm.size() == nums.length` → **copy** into `ans`, then `return`
- **Undo everything you did** — both the list `remove` *and* `used[i] = false`
- Time `O(n! × n)`, auxiliary space `O(n)`
- Time is **optimal** — output size forces it
- Leaves = multiply branches per level: `n × (n-1) × … × 1 = n!`