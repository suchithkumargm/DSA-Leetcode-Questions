# LC 90 — Subsets II

**Difficulty:** Medium
**Topic:** Backtracking / Recursion, Duplicate Handling
**Link:** https://leetcode.com/problems/subsets-ii/

Given an integer array `nums` that **may contain duplicates**, return all possible
subsets (the power set). The solution set must not contain duplicate subsets.
Order of output does not matter.

```
Input:  nums = [1,2,2]
Output: [[], [1], [1,2], [1,2,2], [2], [2,2]]
```

---

## Table of Contents

1. [Attempt 0 — Nested loops (wrong problem entirely)](#attempt-0)
2. [Attempt 1 — Take / Not-take + HashSet, no sort](#attempt-1)
3. [Attempt 2 — Sort + HashSet (accepted, but wasteful)](#attempt-2)
4. [Attempt 3 — Hybrid loop + take/not-take (broken)](#attempt-3)
5. [Final — Sort + prune duplicate branches](#final)
6. [Complexity, explained from scratch](#complexity)
7. [The loop-form alternative](#loop-form)
8. [Java gotchas that bit me](#gotchas)
9. [Related problems](#related)
10. [Recall checklist](#recall)

---

<a name="attempt-0"></a>
## Attempt 0 — Nested loops → contiguous subarrays

My first instinct was three nested index loops.

```java
Set<List<Integer>> ans = new HashSet<>();
for (int i = 0; i < nums.length; i++) {
    for (int j = i; j < nums.length; j++) {
        List<Integer> curr = new ArrayList<>();
        for (int k = i; k <= j; k++) curr.add(nums[k]);
        ans.add(curr);
    }
}
ans.add(List.of());
```

### Why it's wrong

This generates every **contiguous slice** `nums[i..j]` — that's *subarrays*, not
*subsets*.

Trace `[1,2,3]`:

```
[1], [1,2], [1,2,3], [2], [2,3], [3], []     → 7 results
Expected: 8. Missing [1,3].
```

`[1,3]` needs index 1 skipped while indices 0 and 2 are kept. No `i..j` window can
do that. **Any subset that isn't a continuous run is unreachable.**

> **Lesson.** Nested index loops generate *windows*. Subsets need a
> *per-element decision*. If the answer involves skipping arbitrary positions,
> loops-over-ranges is the wrong shape — reach for recursion.

Secondary bug: without sorting, `[4,4,1]` produces both `[4,1]` and `[1,4]`. Two
`ArrayList`s with different element order are **not** `equals()`, so the `HashSet`
keeps both. A set-of-lists only dedupes when elements are in a canonical order.

---

<a name="attempt-1"></a>
## Attempt 1 — Take / Not-take + HashSet, no sort

Right structure, wrong preprocessing.

```java
private void formSubset(int index, int[] nums,
                        List<Integer> combination, Set<List<Integer>> ans) {
    if (index == nums.length) {
        ans.add(new ArrayList<>(combination));
        return;
    }
    // choose
    combination.add(nums[index]);
    formSubset(index + 1, nums, combination, ans);
    combination.remove(combination.size() - 1);   // undo

    // don't choose
    formSubset(index + 1, nums, combination, ans);
}
```

This is the correct backtracking skeleton:

- **One index per call.** Each frame handles exactly `nums[index]`.
- **Base case at `index == nums.length`.** Every leaf is a complete subset.
- **Copy on record.** `new ArrayList<>(combination)` — without the copy, every
  entry in `ans` points at the same mutating list and you end up with N copies
  of `[]`.
- **Undo after recursing.** The `remove` restores state so the not-take branch
  starts clean.

### Why it fails

`nums = [4,1,4]`. Recursion picks in index order, so:

| indices kept | list built |
|---|---|
| `{0,1}` | `[4,1]` |
| `{1,2}` | `[1,4]` |

Same subset, different list order, so `equals()` says they differ and the `HashSet`
keeps both. **Fix: `Arrays.sort(nums)` at the top.**

---

<a name="attempt-2"></a>
## Attempt 2 — Sort + HashSet (accepted, but wasteful)

Adding `Arrays.sort(nums)` to Attempt 1 makes it **pass**. Sorting forces every
equal subset into the same order, so the `HashSet` can finally recognise them as
identical.

### Why it's still not the answer

The `Set` is a crutch. The algorithm still:

1. Walks all **2ⁿ leaves**,
2. Builds a **full O(n) list** at each one,
3. Pays **O(n) hashing** per list,
4. Then throws the duplicates away.

On `[1,1,1,1,1]` it constructs **32 lists to keep 6**.

> **What sorting does and doesn't do.**
> Sorting canonicalises the order *within* each subset — it kills the `[4,1]` vs
> `[1,4]` problem. It does **nothing** about generating the same subset twice
> from different index paths.

Proof that duplicates survive sorting — `nums = [1,2,2]`, already sorted:

```
take idx0, SKIP idx1, take idx2  →  [1,2]
take idx0, take idx1, SKIP idx2  →  [1,2]
```

Two distinct root-to-leaf paths, identical output list. The `HashSet` is silently
eating that. Remove the `Set` at this stage and `[1,2]` appears twice.

---

<a name="attempt-3"></a>
## Attempt 3 — Hybrid loop + take/not-take (broken)

Trying to prune, I bolted a loop onto the take/not-take body:

```java
for (; index < nums.length; index++) {
    if (index < nums.length - 1 && nums[index] == nums[index + 1]) continue;  // ✗
    combination.add(nums[index]);
    formSubset(index + 1, nums, combination, ans);
    combination.remove(combination.size() - 1);
    formSubset(index + 1, nums, combination, ans);   // ✗
}
```

**This merges two incompatible paradigms.** Take/not-take handles *one index per
call*. The loop form handles *a range per call*. Having both means the frame
re-runs take/not-take on indices it already finished.

Trace `[1,2,2]`:

```
f(0,[])  i=0: nums[0]=1 != nums[1] → no skip
           take 1  → f(1,[1])  ⇒ emits [1,2], [1]
           skip 1  → f(1,[])   ⇒ emits [2], []
         i=1: nums[1]==nums[2] → continue
         i=2: take 2 → f(3,[2]) ⇒ emits [2]   ← DUPLICATE
              skip 2 → f(3,[])  ⇒ emits []    ← DUPLICATE

Got:      [1,2], [1], [2], [], [2], []
Expected: [], [1], [1,2], [1,2,2], [2], [2,2]
```

Two separate failures:

**(a) Duplicates return.** After the frame at index 0 finishes both branches,
`combination` is back to `[]` — then the loop advances and starts a *fresh*
take/not-take from index 2 on that same empty list. Everything reachable there was
already reachable through the skip branch. The loop re-explores the subtree it
just finished.

**(b) `[1,2,2]` and `[2,2]` become unreachable.** The `continue` fires on the
**first** copy of a duplicate — index 1 is skipped outright because
`nums[1] == nums[2]`. If you never take the first `2`, you can never hold two of
them.

> **Lesson.** The guard was aimed at the wrong branch. It blocked the **take**,
> when what needs blocking is the **take-after-a-skip**.

---

<a name="final"></a>
## Final — Sort + prune duplicate branches

### The key insight

Go back to why `[1,2,2]` produces `[1,2]` twice:

```
take idx0, TAKE idx1, skip idx2   →  [1,2]
take idx0, SKIP idx1, take idx2   →  [1,2]
```

Both keep **one** `2`, but they pick a **different copy** of it. Same for `[2]` —
reachable from index 1 or index 2.

**That is the entire source of duplication.** When you keep k copies out of a run
of equal values, every choice of *which* k gives the same list.

So fix a convention:

> **If you keep any copies of a value, they are always the leftmost ones.**

Under that rule the first path above is legal, the second is illegal — it skipped
a `2` and then took a later `2`.

Enforcing it is one rule:

> **The moment you decline a value, decline every remaining copy of it.**

You can never come back for a later duplicate, so the "skip-then-take" path stops
existing. The **take branch needs no guard at all** — taking the first `2` and then
choosing freely on the second is legitimate, that's how you reach `[2,2]`.

### The code

```java
class Solution {
    public List<List<Integer>> subsetsWithDup(int[] nums) {
        List<List<Integer>> ans = new ArrayList<>();
        Arrays.sort(nums);                       // duplicates become adjacent
        formSubset(0, nums, new ArrayList<>(), ans);
        return ans;                              // no Set needed
    }

    private void formSubset(int index, int[] nums,
                            List<Integer> combination, List<List<Integer>> ans) {
        if (index == nums.length) {
            ans.add(new ArrayList<>(combination));
            return;
        }

        // ---- choose nums[index] : NO guard, this branch is always valid ----
        combination.add(nums[index]);
        formSubset(index + 1, nums, combination, ans);
        combination.remove(combination.size() - 1);

        // ---- don't choose nums[index] : nor ANY of its duplicates ----
        while (index < nums.length - 1 && nums[index] == nums[index + 1]) {
            index++;
        }
        formSubset(index + 1, nums, combination, ans);
    }
}
```

Bounds check `index < nums.length - 1` must come **first**, or `nums[index + 1]`
reads past the end on the last element.

### Verification trace — `[1,2,2]`

```
f(0,[])
 ├ take 1 → f(1,[1])
 │           ├ take 2 → f(2,[1,2])
 │           │           ├ take 2 → f(3,[1,2,2]) ⇒ [1,2,2]
 │           │           └ skip   → f(3,[1,2])   ⇒ [1,2]
 │           └ skip 2s: index 1→2, recurse f(3,[1]) ⇒ [1]
 └ skip 1 → f(1,[])
             ├ take 2 → f(2,[2])
             │           ├ take 2 → ⇒ [2,2]
             │           └ skip   → ⇒ [2]
             └ skip 2s → f(3,[]) ⇒ []

6 leaves, 6 distinct subsets, zero duplicates. No Set.
```

**Self-check before submitting:** run `[1,1,1]`. Should visit exactly **4** leaves
producing `[1,1,1], [1,1], [1], []`. If you see 8 leaves, the skip isn't firing.

---

<a name="complexity"></a>
## Complexity, explained from scratch

### Time — O(n · 2ⁿ)

**Step 1: how many leaves?**

At every index: take it or skip it. Two choices, n indices.

```
                    f(0)
              take /    \ skip
                f(1)      f(1)
             t /  \ s   t /  \ s
           f(2)  f(2)  f(2)  f(2)
           / \   / \   / \   / \
          L   L L   L L   L L   L      ← 8 leaves for n=3
```

2 × 2 × 2 = 2³. In general **2ⁿ leaves** — which matches the fact that a set of n
elements has 2ⁿ subsets. Each leaf *is* one subset.

**Step 2: work per leaf?**

```java
ans.add(new ArrayList<>(combination));
```

The copy is **not free**. It walks all elements of `combination` — that's **O(n)**.

> This is the part that's easy to miss. "Just one add, constant time" is wrong —
> you're duplicating a whole list. Saying plain "O(2ⁿ)" in an interview reads
> like you forgot the copy cost.

**Step 3: multiply.**

```
2ⁿ leaves  ×  O(n) per leaf  =  O(n · 2ⁿ)
```

Internal nodes do O(1) each, ~2ⁿ of them → O(2ⁿ), dominated. Drops out.

**Sanity check that this is a lower bound too.** For n = 3 the output is
`[] [1] [2] [3] [1,2] [1,3] [2,3] [1,2,3]` — **12 integers**. Any correct solution
writes those same 12 numbers. So O(n · 2ⁿ) isn't the algorithm being slow, it's
**the size of the thing you were asked to produce.** Unbeatable.

### Space — O(n) auxiliary

Space = **extra memory alive at one moment**, not total ever allocated.

Only one root-to-leaf path is active at a time:

```
f(0)  ← frame
 f(1) ← frame
  f(2)← frame
   f(3)← frame, base case, emits
```

- **Call stack:** at most n+1 frames (`index` advances by ≥1 per call), each a
  fixed size → **O(n)**
- **`combination`:** at most n elements → **O(n)**

They coexist, so O(n) + O(n) = **O(n)**.

**Why 2ⁿ doesn't appear here.** The tree has 8 leaves, but you never have 8
branches open simultaneously. You descend one path, hit the leaf, return — those
frames are freed before the next path starts. **Depth counts, not total nodes.**

### Why `ans` isn't counted

`ans` genuinely holds n·2ⁿ integers. But the convention is **auxiliary space** —
memory beyond the required output. The problem *asked* for all subsets, so storing
them isn't a choice you made.

> **Interview phrasing:** *"O(n) auxiliary space, O(n·2ⁿ) if you count the
> output."* Saying both shows you know the distinction rather than having picked
> one at random.

### What the pruning actually buys

Worst case is unchanged — all-distinct input is still O(n · 2ⁿ), and it has to be.

What changes: work now scales with the number of **distinct** subsets, not 2ⁿ.
For a value appearing k times, the pruned tree explores **k+1 branches instead of
2ᵏ**.

| Input | Unpruned leaves | Pruned leaves |
|---|---|---|
| `[1,2,3]` | 8 | 8 |
| `[1,1,2]` | 8 | 6 |
| `[1,1,1]` | 8 | 4 |
| `[1,1,1,1,1]` | 32 | 6 |

Real bound: **O(n · D)** where D = number of distinct subsets.

---

<a name="loop-form"></a>
## The loop-form alternative

Same problem, different skeleton. Worth knowing because **this is the form that
generalises to Combination Sum II and Permutations II.**

```java
private void backtrack(int start, int[] nums,
                       List<Integer> curr, List<List<Integer>> ans) {
    ans.add(new ArrayList<>(curr));          // record at EVERY node, not at a leaf

    for (int i = start; i < nums.length; i++) {
        if (i > start && nums[i] == nums[i - 1]) continue;   // skip sibling dupes
        curr.add(nums[i]);
        backtrack(i + 1, nums, curr, ans);
        curr.remove(curr.size() - 1);
    }
}
```

Three differences from take/not-take:

| | Take / Not-take | Loop form |
|---|---|---|
| Record where | at the leaf (`index == n`) | at **every node** (root gives `[]` free) |
| Frame handles | one index | a range `start..n-1` |
| Dupe guard | on the **not-take** branch | `i > start` inside the loop |

### Understanding `i > start`

This condition is the whole trick, and it's subtle. Take `[1,2,2]`:

- **Level with `start == 1`.** Loop sees `i=1` (first `2`) and `i=2` (second `2`).
  These are **siblings** — both start a branch from the same `curr`. Picking `i=2`
  reproduces exactly what `i=1` did. `i > start` is true at `i=2` → **skip**. ✓
- **Level with `start == 2`** (reached after already taking `nums[1]`). Only
  `i=2` available, and `i > start` is **false**. Not skipped → produces `[1,2,2]`,
  a genuinely new subset. ✓

**Same value, same index, different level, opposite decision.** That's why the
condition can't be just `nums[i] == nums[i-1]` — it needs `i > start` to tell
"sibling duplicate" apart from "child of one I already picked".

### Two equivalent ways to write the skip in take/not-take form

```java
// Form A — compare adjacent pairs (what I wrote)
while (index < nums.length - 1 && nums[index] == nums[index + 1]) index++;
formSubset(index + 1, ...);

// Form B — compare against the original value
int next = index + 1;
while (next < nums.length && nums[next] == nums[index]) next++;
formSubset(next, ...);
```

Both land on the same place. **Form B is easier to say out loud** — *"advance until
the value changes"* — and it mirrors the loop-form guard.

---

<a name="gotchas"></a>
## Java gotchas that bit me

**`remove(int)` vs `remove(Object)`**

```java
combination.remove(combination.size() - 1);   // remove(int index)  ✓
```

Resolves to remove-by-index because the argument is `int`. But on a
`List<Integer>`, passing an `Integer` object silently hits `remove(Object)` and
removes by **value**. Classic overload trap — interviewers ask about it.

**`List.of()` returns an immutable list.** Fine if you never mutate it, but
`add`/`remove` on it throws `UnsupportedOperationException`.

**Always copy when recording.** `ans.add(combination)` (no copy) stores a
*reference*. Every later mutation is visible through it, and you finish with N
identical entries. `ans.add(new ArrayList<>(combination))`.

**Don't clobber a loop variable mid-body.** My final version mutates the parameter
`index` inside the `while`. That's safe *here* because the take branch already ran
and nothing below reads `index` again. But in loop-form backtracking, mutating the
loop variable mid-body is a real bug source — introduce a separate `next` when the
original still has work to do.

---

<a name="related"></a>
## Related problems — same skip rule

| # | Problem | What changes |
|---|---|---|
| **78** | Subsets | No duplicates. Drop the sort and the guard entirely. |
| **90** | Subsets II | ← this one |
| **39** | Combination Sum | No dupes in input, but **unlimited reuse** → recurse on `i`, not `i+1`. Good contrast. |
| **40** | Combination Sum II | Sort + `i > start` skip, plus a target sum. Nearest neighbour to 90. |
| **47** | Permutations II | Order matters, so sorted-adjacent alone **isn't enough** — needs a `used[]` array. |

**LC 47 is the one people trip on.** Do it right after this while the skip logic is
fresh, and pay attention to why the guard becomes
`i > 0 && nums[i] == nums[i-1] && !used[i-1]`.

---

<a name="recall"></a>
## Recall checklist

Read these before re-solving; if any is fuzzy, re-read that section.

- [ ] Nested index loops give **subarrays**, not subsets. Subsets need a
  per-element take/skip decision.
- [ ] `Arrays.sort()` fixes `[4,1]` vs `[1,4]`. It does **not** fix the same
  subset arriving via two different index paths.
- [ ] Duplication source: **which copy** of a repeated value you kept.
- [ ] Convention that kills it: **if you keep copies, keep the leftmost ones.**
- [ ] Implementation: guard the **not-take** branch, never the take branch.
- [ ] Skip advances past the **entire run** of equal values.
- [ ] Bounds check first in the `while`, or you read past the array end.
- [ ] Copy the list when recording: `new ArrayList<>(combination)`.
- [ ] Time **O(n · 2ⁿ)** — 2ⁿ leaves × O(n) copy. The copy is the part you forget.
- [ ] That bound is also a **lower** bound: the output itself is n·2ⁿ integers.
- [ ] Space **O(n)** — stack depth + `combination`. Depth, not node count.
- [ ] Say "O(n) auxiliary, O(n·2ⁿ) with output" out loud.
- [ ] Loop form guard is `i > start`, and you record at **every** node.

### One-line summary

> Sort so duplicates are adjacent; recurse take/not-take on each index; when you
> decline a value, decline its entire run. O(n · 2ⁿ) time, O(n) space.