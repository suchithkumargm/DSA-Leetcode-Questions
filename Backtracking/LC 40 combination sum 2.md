# LC 40 — Combination Sum II

**Difficulty:** Medium
**Topics:** Backtracking, Recursion, Sorting
**Link:** https://leetcode.com/problems/combination-sum-ii/
**Final result:** Beats 99%

---

## Problem

Given `candidates[]` (may contain duplicates) and a `target`, return all **unique** combinations that sum to `target`.

Each number may be used **at most once**.
Combinations are sets, not sequences — `[1,2,5]` and `[2,1,5]` are the same answer.

```
Input:  candidates = [10,1,2,7,6,1,5], target = 8
Output: [[1,1,6],[1,2,5],[1,7],[2,6]]
```

Note there are **two** 1s in the input, and `[1,1,6]` legitimately uses both. But `[1,7]` must appear only **once**, even though either 1 could form it. That tension is the whole problem.

---

## The core difficulty

Two separate things are easy to confuse:

| | Meaning | Handled by |
|---|---|---|
| **Element reuse** | Can I pick `candidates[i]` twice? | `index + 1` in the recursive call |
| **Duplicate combinations** | Can two different index-sets produce the same value-list? | Skipping equal values at the same level |

**LC 39 (Combination Sum I)** = reuse allowed, no duplicates in input → only needs `index` (not `index+1`).
**LC 40 (this one)** = reuse forbidden, duplicates in input → needs `index+1` **and** the skip.

Getting `index+1` right does not solve duplicates. That was the mistake that cost the most iterations.

---

## Journey

### v1 — Brute force: pick / not-pick + HashSet

```java
Set<List<Integer>> ans = new HashSet<>();
Arrays.sort(candidates);
findCombination(0, target, candidates, new ArrayList<>(), ans);
return new ArrayList<>(ans);

private void findCombination(int index, int target, int[] candidates,
                             List<Integer> combination, Set<List<Integer>> ans) {
    if (index == candidates.length || target < 0) return;   // BUG: ordering
    if (target == 0) { ans.add(new ArrayList<>(combination)); return; }

    // pick
    int currEle = candidates[index];
    combination.add(currEle);
    findCombination(index + 1, target - currEle, candidates, combination, ans);
    combination.remove(combination.size() - 1);

    // don't pick
    findCombination(index + 1, target, candidates, combination, ans);
}
```

**Idea:** classic binary recursion — every element is either in or out. Generate everything, let a `HashSet` throw away duplicates.

**Why the set works at all:** because of `Arrays.sort`. Sorting makes every combination come out in ascending order, so two equal multisets serialise to *identical* lists, and `List.equals` catches them. Without the sort, `[1,2,5]` and `[5,2,1]` would be different keys and the set would fail.

#### Bug: base case ordering

```
candidates = [1], target = 1
  pick the 1 → findCombination(index=1, target=0)
    check 1: index == length → return          ← exits here
    check 2: target == 0                        ← never reached
Result: [] instead of [[1]]
```

`target == 0` is a **success** condition and must be tested before any **dead-end** condition. When you exhaust the array with target exactly 0, that's a win, not a failure.

> **General rule:** in backtracking, order base cases *success first, failure second*. Success is a fact about the state; failure is a fact about the remaining options.

#### Why v1 is slow

- Hashes every candidate list — walks all `k` elements to compute `hashCode`, then again for `equals` on collisions
- Still **generates** all duplicate branches before discarding them — the wasted work already happened
- Rebuilds a fresh `ArrayList` from the set at the end

Beats ~15%.

---

### v2 — Attempt the skip in the not-pick branch (broken)

**The reasoning that got here:** the array is sorted, so duplicates are adjacent. When I take the *not-pick* branch on `candidates[index]`, I'm committing to "no copy of this value appears in this subtree." So the not-pick branch should jump past **all** equal elements, not just one.

That reasoning is correct. The implementation was not.

```java
while (index + 1 < n && candidates[index] == candidates[index + 1]) index++;
findCombination(index, target, ...);      // BUG: lands ON the last duplicate
```

```
candidates = [1,1,1], index = 0
  loop: c[0]==c[1] → index=1
        c[1]==c[2] → index=2
        c[2]==c[3] → out of bounds / stops
  exits with index = 2  ← still pointing at a 1
  recurse(2, ...) → picks that 1 → duplicate [1] generated anyway
```

The loop leaves `index` on the **last** duplicate, not **past** the run. Needed `index + 1` in the recursive call, or `index++` after the loop.

Also `n` was undefined — didn't compile.

---

### v3 — Pivot to the for-loop pattern

Instead of pick / not-pick, loop over every possible **next** element:

```java
for (; index < candidates.length - 1; index++) {          // BUG: bound
    int currEle = candidates[index];
    combination.add(currEle);
    findCombination(index + 1, target - currEle, candidates, combination, ans);
    combination.remove(combination.size() - 1);
    while (candidates[index] == candidates[index + 1]) index++;   // BUG: no guard
}
```

**Why this shape is better:** the chain of not-pick branches in v1 is exactly a loop. Flattening it means the dedup lives in the loop increment, where it's natural — after finishing a value, advance past all its copies.

Two bugs, pulling in **opposite directions**:

1. `length - 1` → last element never picked. `[1,2]`, target 2: at `index=0` picks the 1, recurses to `findCombination(1, 1)` whose loop is `1 < 1` = false, immediate return. Back at top `index=1`, loop exits. `[2]` never generated.
2. Inner `while` has no `index + 1 < length` guard → `ArrayIndexOutOfBoundsException` on `[1,1,1]`.

Fixing (1) makes (2) **more** reachable, not less. Both must be fixed together — a good reminder that two bugs can mask each other.

---

### v4 — Correct

```java
for (; index < candidates.length; index++) {
    int currEle = candidates[index];
    combination.add(currEle);
    findCombination(index + 1, target - currEle, candidates, combination, ans);
    combination.remove(combination.size() - 1);
    while (index + 1 < candidates.length && candidates[index] == candidates[index + 1]) {
        index++;
    }
}
```

Trailing `while` advances to the last duplicate, then the for-update `index++` steps past the run. Correct.

Beats ~36% — still carrying the now-useless `HashSet`.

---

### v5 — Drop the set

**Careful about the reason.** "I'm sorting, so I don't need the set" is *wrong* — v1 sorted too and absolutely needed it.

Sorting only makes duplicates **adjacent**, which is what makes the skip *possible*. The **skip** is what removes duplicates.

**The actual argument:** at each recursion level, the loop picks each *distinct* value at most once as the head of the remaining combination. Two different iterations therefore start from different values, so their subtrees can never produce the same list. No duplicate is ever **generated** → nothing left for a set to catch.

So sorting does two jobs, and only the first still matters:
- makes duplicates adjacent → enables the skip ✅ still needed
- canonicalises output order → let the set compare by value ❌ no longer relevant

`Set<List<Integer>>` → `List<List<Integer>>`, and return `ans` directly instead of `new ArrayList<>(ans)`.

---

### v6 — Final: the sorted-array prune

```java
if (candidates[index] > target) break;
```

**Intuition:** the array is sorted ascending. If `candidates[index]` alone already exceeds `target`, then so does every element to its right. The entire remaining tail is dead. One comparison kills all of it.

Before: add element → recursive call → callee checks `target < 0` → return → remove element. That's a wasted call *per remaining element*.
After: never enter the loop body at all.

Biggest impact on inputs like `[1,1,1,...,1,50]` with a small target — skips whole subtrees instead of exploring and discarding them.

**Cascade effect — two base cases become dead code:**

- `target < 0` — unreachable. `target - currEle` can't go negative if we never pick a `currEle` exceeding `target`.
- `index == candidates.length` — was already dead. If true, the loop simply doesn't execute and the method returns anyway.

Leaves `target == 0` as the **only** base case. That's the clean version.

Beats 99%.

---

## Final solution

```java
class Solution {
    public List<List<Integer>> combinationSum2(int[] candidates, int target) {
        List<List<Integer>> ans = new ArrayList<>();
        Arrays.sort(candidates);
        findCombination(0, target, candidates, new ArrayList<>(), ans);
        return ans;
    }

    private void findCombination(int index, int target, int[] candidates,
                                 List<Integer> combination, List<List<Integer>> ans) {
        if (target == 0) {
            ans.add(new ArrayList<>(combination));
            return;
        }

        for (int i = index; i < candidates.length; i++) {
            if (candidates[i] > target) break;              // prune: sorted ⇒ tail is dead

            combination.add(candidates[i]);
            findCombination(i + 1, target - candidates[i], candidates, combination, ans);
            combination.remove(combination.size() - 1);     // backtrack

            while (i + 1 < candidates.length && candidates[i] == candidates[i + 1]) {
                i++;                                        // skip duplicates
            }
        }
    }
}
```

> Using `int i = index` instead of mutating the `index` parameter keeps "where this call started" available — needed if you switch to the `i > index` dedup idiom below, and safer if anyone reorders the code later.

---

## Complexity

### Time — O(n log n + 2ⁿ × k) → **O(2ⁿ × k)**

| Term | Source |
|---|---|
| `n log n` | `Arrays.sort` |
| `2ⁿ` | recursion tree — each element is in or out, so 2ⁿ candidate subsets |
| `× k` | `new ArrayList<>(combination)` copies `k` elements, **once per valid combination** |

**The `k` is a multiplier, not an addend.** Common mistake: writing `2ⁿ + k`. The copy happens at every leaf that succeeds, so it multiplies the leaf count. `k` = length of the combination being copied.

**Also:** `ArrayList.add(e)` appends at the tail in amortised O(1) — there is no shifting. The `k` cost comes from the **copy constructor walking the source list**, nothing to do with insertion position.

`2ⁿ` dominates `n log n`, so the sort is normally dropped from the stated answer.

**Nuance to have ready:** `2ⁿ` is a *loose upper* bound — the size of the full subset lattice. The real bound is tighter, because dedup avoids exploring the full lattice when duplicates exist, and the `break` prunes further. But you can't state a tighter bound without knowing the input distribution. Say `2ⁿ`, note it's an upper bound, mention pruning helps in practice.

**"Where does 2ⁿ come from?"** — each element is either in or out of a given combination → 2ⁿ candidate subsets → the recursion tree has that many leaves in the worst case (all elements distinct, no pruning possible).

### Space — auxiliary **O(n)**

| Component | Cost |
|---|---|
| `combination` list | at most `n` deep → O(n) |
| recursion call stack | one frame per level → O(n) |
| **total auxiliary** | **O(n)** |

State O(n), not O(k) — `k ≤ n` always, so O(n) is the honest bound. Don't forget the **call stack**; it's easy to only count the explicit data structure.

Output space (usually excluded, but be ready): O(2ⁿ × k) for the answer list itself.

---

## Patterns to carry forward

**1. Base case ordering: success before failure.**
`target == 0` before `index == length`. Reaching the array's end with target exactly 0 is a win.

**2. Sorting ≠ dedup.**
Sorting makes duplicates adjacent (enabling dedup) and canonicalises output order. Removing duplicates is a separate mechanism.

**3. Skip at the *same level*, not inside the subtree.**
Duplicates arise when two iterations *at the same recursion depth* start with the same value. The dedup belongs in the loop's advance, never in the recursive call.

**4. Two equivalent dedup idioms — same thing, opposite direction:**

```java
// forward — what I wrote (advance past the run after using a value)
while (i + 1 < candidates.length && candidates[i] == candidates[i + 1]) i++;

// backward — the more common idiom (skip if this value was just used)
if (i > index && candidates[i] == candidates[i - 1]) continue;
```

The backward form needs the original `index` preserved, which is why `int i = index` matters. It also groups all guards at the top of the loop next to the `break`.

**5. Sorted array ⇒ `break`, not `continue`.**
If `candidates[i]` overshoots, so does the whole tail. `continue` would pointlessly test every remaining element.

**6. Prune *before* the recursive call, not after.**
Catching an overshoot via `target < 0` inside the callee costs an add + a call + a check + a remove. Checking `candidates[i] > target` before entering costs one comparison.

**7. Pick/not-pick and the for-loop are the same tree.**
The for loop flattens the chain of not-pick branches into iteration. Interviewers sometimes ask you to convert between them — know both.

**8. When a prune makes a base case unreachable, delete it.**
Dead conditions are noise that hides the real structure. After the `break`, both `target < 0` and `index == length` are dead.

---

## Related problems

| Problem | Diff from this code |
|---|---|
| **LC 39 — Combination Sum** | Unlimited reuse, no input duplicates. Change `i + 1` → `i` in the recursive call. Drop the dedup while-loop. **One character of real difference.** |
| **LC 46 — Permutations** | Order matters → use a `visited[]` array instead of an index cursor |
| **LC 47 — Permutations II** | Permutations + duplicates → same sort-then-skip idea, but the skip condition also checks `visited[i-1]` |
| **LC 78 — Subsets** | No target — collect at every node instead of only at `target == 0` |
| **LC 90 — Subsets II** | Subsets + duplicates → identical skip pattern to this problem |

LC 40 → LC 90 is the tightest pair. If you can do this one, LC 90 is the same skeleton with the target check removed.

---

## Trace to verify a rewrite

```
candidates = [10,1,2,7,6,1,5], target = 8
sorted     = [1,1,2,5,6,7,10]

expected: [1,1,6], [1,2,5], [1,7], [2,6]
```

Check specifically that:
- `[1,1,6]` appears — both 1s used, so the skip must not block *nested* reuse of an equal value
- `[1,7]` appears exactly **once** — not twice, one per 1
- `[10]` is never explored — `break` fires

That pair of conditions on the 1s is the whole test. Blocking too little gives duplicate `[1,7]`; blocking too much loses `[1,1,6]`.