# LC 40 — Combination Sum II

**Difficulty:** Medium · **Topics:** Backtracking, Recursion, Sorting
**Pattern:** Backtracking with structural deduplication

---

## 1. Problem

Given `candidates[]` (which **may contain duplicates**) and a `target`, return all **unique**
combinations summing to `target`. Each number may be used **at most once**.

Combinations are sets, not sequences — `[1,2,5]` and `[2,1,5]` are the same answer.

```
candidates = [10,1,2,7,6,1,5], target = 8
→ [[1,1,6], [1,2,5], [1,7], [2,6]]
```

Note the input has **two** 1s, and `[1,1,6]` legitimately uses both. But `[1,7]` must appear
**once**, even though either 1 could form it.

> **That tension is the whole problem:** block too little and `[1,7]` appears twice; block too much
> and `[1,1,6]` disappears.

---

## 2. The core difficulty — two things that look alike

| | Question | Handled by |
|---|---|---|
| **Element reuse** | Can I pick `candidates[i]` twice? | `index + 1` in the recursive call |
| **Duplicate combinations** | Can two different index-sets produce the same value-list? | Skipping equal values **at the same level** |

- **LC 39** = reuse allowed, distinct input → only needs `index` (not `index+1`)
- **LC 40** = reuse forbidden, duplicate input → needs `index+1` **and** the skip

> **Getting `index + 1` right does not solve duplicates.** They're independent mechanisms, and this
> is the confusion that costs the most time on this problem.

---

## 3. The trap: filtering instead of pruning

The natural first attempt is to generate everything and let a `HashSet` discard duplicates.

```java
Set<List<Integer>> ans = new HashSet<>();
Arrays.sort(candidates);
// ... standard pick / skip, index + 1 on both branches
return new ArrayList<>(ans);
```

It's **correct** — and it **TLEs**.

### Why it works at all

Only because of `Arrays.sort`. Sorting makes every combination come out in ascending order, so two
equal multisets serialise to *identical* lists and `List.equals` catches them. Unsorted, `[1,2,5]`
and `[5,2,1]` are different keys and the set fails.

### ⚠️ Why it TLEs rather than just running a bit slower

This is the important part, and it isn't about hashing cost.

> **The work is exponential while the output stays tiny.**
>
> On input like `[1,1,1,…,1]` with a small target there are 2ⁿ root-to-leaf paths producing a
> handful of distinct combinations. The set deduplicates the *results* but does nothing about the
> *paths* — every one is still walked.

**Filtering fixes the output. Pruning fixes the work.** A `HashSet` on the answers can never
recover the time already spent generating them.

Secondary costs, real but not the reason for the TLE: hashing walks all `k` elements per list, then
again for `equals` on collisions, and the final `new ArrayList<>(set)` rebuilds everything.

> **General rule for enumeration problems: dedup by construction, never by filtering.**

---

## 4. Where duplicates come from

Sort first, so equal values are adjacent: `[1, 1, 2, 5]`.

At index 0 you **pick** the first 1. Separately, at index 0 you **skip** it, then at index 1 you
**pick** the second 1. Both paths produce combinations starting with a single 1 — identical,
because the two 1s are interchangeable.

> ### 🔑 The fact
> **At a given recursion level, don't start a combination with a value you've already tried at that
> level.** Concretely: after *declining* a value, skip every copy of it — not just one.
>
> Equivalently: **nested** reuse of an equal value is fine (that's `[1,1,6]`); **sibling** reuse at
> the same depth is the duplicate.

The dedup therefore belongs in the loop's advance, or in the skip branch — **never inside the
recursive call.**

---

## 5. Formulation A — pick / skip

The binary-recursion shape. The dedup goes in the **skip** branch.

```java
class Solution {
    public List<List<Integer>> combinationSum2(int[] candidates, int target) {
        List<List<Integer>> ans = new ArrayList<>();
        Arrays.sort(candidates);
        findCombination(candidates, target, 0, new ArrayList<>(), ans);
        return ans;
    }

    private void findCombination(int[] candidates, int target, int index,
                                 List<Integer> combination, List<List<Integer>> ans) {
        if (target == 0) {                                  // SUCCESS first
            ans.add(new ArrayList<>(combination));
            return;
        }
        if (index >= candidates.length || candidates[index] > target) {
            return;                                         // sorted ⇒ the whole tail is dead
        }

        // 1. PICK — index + 1, no reuse
        combination.add(candidates[index]);
        findCombination(candidates, target - candidates[index], index + 1, combination, ans);
        combination.remove(combination.size() - 1);

        // 2. SKIP — advance past EVERY copy of the declined value
        int skipped = candidates[index];
        int next = index + 1;
        while (next < candidates.length && candidates[next] == skipped) next++;
        findCombination(candidates, target, next, combination, ans);
    }
}
```

### ⚠️ Two bugs specific to this shape

**Reading backwards from the current index.**

```java
while (index < candidates.length && candidates[index] == candidates[index - 1]) index++;   // 🚩
```

At `index == 0` this is `candidates[-1]` → `ArrayIndexOutOfBoundsException`.

**Anchoring the comparison to a mutating variable.** Even with the bounds fixed, the loop above
compares `candidates[index]` against `candidates[index-1]` while `index` is being incremented — so
after one step it's comparing *consecutive pairs*, not against the value originally declined.

**Fix both by capturing the value first and reading forward from a fixed anchor:**

```java
int skipped = candidates[index];     // anchor — doesn't move
int next = index + 1;                // separate cursor — index isn't mutated
while (next < candidates.length && candidates[next] == skipped) next++;
```

---

## 6. Formulation B — the for-loop

The chain of skip branches *is* a loop. Flattening it puts the dedup in the loop's advance, where
it reads more naturally.

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

The trailing `while` advances to the **last** duplicate; the for-update `i++` then steps past the
run.

> Use `int i = index` rather than mutating the `index` parameter — it keeps "where this call
> started" available, which the alternative dedup idiom below needs.

### ⚠️ Bugs specific to this shape

**`i < candidates.length - 1`** — the last element is never picked. On `[1,2]` with target 2: at
`i=0` pick the 1, recurse into `findCombination(1, 1)` whose loop is `1 < 1` = false; back at the
top `i=1`, loop exits. `[2]` never generated.

**Missing bounds guard on the inner `while`** — `candidates[i] == candidates[i+1]` without
`i + 1 < length` throws on `[1,1,1]`.

Note these two **mask each other**: fixing the bound makes the missing guard *more* reachable, not
less. Fix both together.

**Landing on the last duplicate instead of past it** — if the skip loop runs before the recursion
or without the for-update, `index` still points at a copy and the duplicate is generated anyway.

### Two equivalent dedup idioms

```java
// forward — advance past the run after using a value
while (i + 1 < candidates.length && candidates[i] == candidates[i + 1]) i++;

// backward — skip if this value was just used at this level (the more common idiom)
if (i > index && candidates[i] == candidates[i - 1]) continue;
```

Same mechanism, opposite direction. The backward form needs the original `index` preserved — which
is why `int i = index` matters — and it groups all the guards at the top of the loop beside the
`break`.

---

## 7. Choosing between the two shapes

| | Pick / skip | For-loop |
|---|---|---|
| Tree | binary | n-ary |
| Dedup lives in | the skip branch | the loop advance |
| Pruning | explicit `return` on the base case | `break` kills the whole tail at once |
| Readability | closer to subsets intuition | more compact, standard idiom |

Both are correct and the same asymptotics. **Write the for-loop version in interviews** — it's the
idiom readers expect, and it generalises directly to LC 77, 90, 216.

---

## 8. Removing redundant guards

Once the sorted prune is in place, several conditions become unreachable:

- **`target < 0`** — unreachable. `target - candidates[i]` can't go negative if you never pick a
  value exceeding `target`.
- **A separate `candidates[index] <= target` guard on the pick** — always true after the early
  return.
- **`index == candidates.length`** (for-loop version) — already dead; the loop simply doesn't
  execute.

> **When a prune makes a base case unreachable, delete it.** Three checks doing one check's work is
> noise that hides the real structure.

In the pick/skip version this leaves two conditions; in the for-loop version, `target == 0` is the
**only** base case.

### Base case ordering — success before failure

```java
if (index == candidates.length || target < 0) return;   // 🚩 wrong order
if (target == 0) { record; return; }
```

Trace `candidates = [1], target = 1`: pick the 1, recurse with `index = 1, target = 0`. The first
check fires (`index == length`) and returns — the success check is never reached. Result `[]`
instead of `[[1]]`.

> **In backtracking, order base cases success-first.** Reaching the end of the array with target
> exactly 0 is a win, not a dead end. Success is a fact about the state; failure is a fact about
> the remaining options.

---

## 9. Sorting does two jobs

Worth separating, because "I sorted, so I don't need the set" is **wrong reasoning** — the
HashSet version sorted too and absolutely needed the set.

| Job | Still needed? |
|---|---|
| Makes duplicates **adjacent** → enables the skip | ✅ essential |
| **Canonicalises** output order → lets a set compare by value | ❌ irrelevant once dedup is structural |
| Enables the **`break`** prune | ✅ bonus |

**The actual argument for dropping the set:** at each recursion level, each *distinct* value is
used at most once as the head of the remaining combination. Two iterations at the same level
therefore start from different values, so their subtrees can never produce the same list. No
duplicate is ever **generated**, so there's nothing for a set to catch.

---

## 10. Complexity

### Time — O(n log n + 2ⁿ × k) → **O(2ⁿ × k)**

| Term | Source |
|---|---|
| `n log n` | `Arrays.sort` |
| `2ⁿ` | recursion tree — each element is in or out |
| `× k` | `new ArrayList<>(combination)` copies `k` elements, **once per valid combination** |

**The `k` multiplies, it doesn't add.** Writing `2ⁿ + k` is the common slip — the copy happens at
every successful leaf, so it scales with the leaf count.

Also: `ArrayList.add(e)` appends in amortised O(1), no shifting. The `k` comes from the **copy
constructor walking the source list**, nothing to do with insertion position.

`2ⁿ` dominates `n log n`, so the sort is normally dropped from the stated answer.

> **Nuance to have ready:** `2ⁿ` is a *loose upper* bound — the size of the full subset lattice. The
> real bound is tighter, since dedup avoids exploring the full lattice when duplicates exist and
> the `break` prunes further. You can't state a tighter bound without knowing the input
> distribution. Say `2ⁿ`, note it's an upper bound, mention pruning helps in practice.

**"Where does 2ⁿ come from?"** Each element is either in or out → 2ⁿ candidate subsets → that many
leaves in the worst case (all distinct, no pruning possible).

Contrast LC 39, where depth is `target / min(candidates)` rather than `n`, because the pick doesn't
advance the index.

### Space — auxiliary **O(n)**

| Component | Cost |
|---|---|
| `combination` list | at most `n` deep |
| recursion call stack | one frame per level |
| **total auxiliary** | **O(n)** |

State O(n), not O(k) — `k ≤ n` always. And don't forget the **call stack**; it's easy to count only
the explicit data structure.

Output space (usually excluded, but say so rather than dropping it silently): O(2ⁿ × k).

---

## 11. Trace to verify any rewrite

```
candidates = [10,1,2,7,6,1,5], target = 8
sorted     = [1,1,2,5,6,7,10]

expected: [1,1,6], [1,2,5], [1,7], [2,6]
```

Check three things specifically:

- **`[1,1,6]` appears** — both 1s used, so the skip must not block *nested* reuse of an equal value
- **`[1,7]` appears exactly once** — not once per 1
- **`[10]` is never explored** — the `break` fires

The first two together are the whole test: blocking too little duplicates `[1,7]`; blocking too much
loses `[1,1,6]`.

---

## 12. Pattern

> **Sort, then after declining a value skip every copy of it — duplicates are prevented by
> construction, not filtered afterwards.**

The transferable half: **in enumeration problems, a duplicate you never generate costs nothing; a
duplicate you filter has already cost you the whole subtree.** Reach for structural dedup, and
treat a `HashSet` over results as a sign you haven't found the real constraint yet.

---

## 13. Takeaways

1. **Reuse and duplicates are independent problems.** `index + 1` solves one; the skip solves the
   other.
2. **Filtering fixes the output; pruning fixes the work.** A set over results can't recover
   exponential time already spent.
3. **Nested reuse of an equal value is fine; sibling reuse at the same level is the duplicate.**
4. **Dedup lives in the loop advance or the skip branch** — never inside the recursive call.
5. **Anchor the skip comparison to a captured value**, and read forward. Comparing against a
   mutating index gives consecutive-pair comparisons and a negative index at 0.
6. **Base cases: success before failure.** `target == 0` first.
7. **Sorted array ⇒ `break`, not `continue`.** If one value overshoots, the whole tail does.
8. **Prune before the recursive call**, not via `target < 0` inside the callee.
9. **When a prune makes a base case unreachable, delete it.**
10. **Sorting enables dedup; it doesn't perform it.** Two different jobs.
11. **Pick/skip and the for-loop are the same tree** — know both, write the for-loop.
12. **`2ⁿ × k`, not `2ⁿ + k`.** The copy multiplies the leaf count.