# LC 39 — Combination Sum

**Sheets:** NeetCode 150 | Striver A2Z (Recursion → Subsequences) | Love Babbar 450 | Apna College
**Pattern:** Backtracking — pick / not-pick on an *unbounded* candidate
**Difficulty:** Medium
**Status:** Solved (beats ~95%)

---

## 1. Problem

Given `candidates[]` of **distinct** integers and a `target`, return **all unique combinations** that sum to `target`.

- The **same number may be reused unlimited times**.
- Two combinations are different if the *multiset* of chosen numbers differs.
- Order within a combination doesn't matter; order of combinations doesn't matter.

**Constraints (important — they drive the complexity):**
```
1 <= candidates.length <= 30
2 <= candidates[i] <= 40
1 <= target <= 40
```

> The tiny `target <= 40` is the whole reason this is solvable. Remember this — it tells you the recursion depth is bounded by target, not by n.

---

## 2. The key observation (do this before writing code)

**Unlimited reuse changes the shape of the recursion tree.**

| Problem type | On "pick", index moves to | Depth bounded by |
|---|---|---|
| Subsets / Subsequences | `index + 1` | `n` |
| **Combination Sum (this)** | **`index` (stays!)** | **`target / min(cand)`** |
| Combination Sum II (LC 40) | `index + 1` | `n` |

Because we can reuse an element, the "pick" branch does **not** advance the index. That single line is what makes this problem different from every subsets problem — and it's also exactly where the TLE came from.

---

## 3. Journey of approaches

### 3.1 Naive / "brute force that doesn't even work" — nested loops

First instinct is often: loop over pairs, triples, etc.

```java
for (int i = 0; i < n; i++)
    for (int j = i; j < n; j++)
        if (candidates[i] + candidates[j] == target) ...
```

**Why it fails:** the combination length is **not fixed**. `target = 40, candidates = [2]` needs a combination of length 20. You cannot write 20 nested loops. Any fixed nesting depth `d` only finds combinations of exactly length `d`.

**Takeaway:** unknown/variable depth → recursion is mandatory. This is the signal that the problem is a backtracking problem, not a loop problem.

---

### 3.2 Brute force — generate every multiset, then filter

Conceptually: enumerate every possible multiset of candidates with sum ≤ target, then keep the ones summing to exactly target.

- Correct, but does no pruning at all.
- Wastes work exploring branches after the sum already blew past target.
- Also risks generating `[2,3]` and `[3,2]` as separate answers if you're careless about ordering.

**Two problems to solve:**
1. How to avoid permutation duplicates (`[2,3]` vs `[3,2]`).
2. How to stop early when the sum overshoots.

**The fix for (1) is structural, not a dedup step:** by only ever moving the index *forward*, you force every combination to be generated in non-decreasing index order. `[3,2]` can never be produced, because once you've moved past index 0 you can't go back. **No HashSet needed.** This is the crucial insight — dedup by construction, not by filtering.

---

### 3.3 Optimal — pick / not-pick recursion (final accepted solution)

```java
class Solution {
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        List<List<Integer>> ans = new ArrayList<>();
        findCombination(0, target, candidates, new ArrayList<>(), ans);
        return ans;
    }

    private void findCombination(int index, int target, int[] candidates,
                                 List<Integer> combination, List<List<Integer>> ans) {
        // base conditions
        if (index == candidates.length || target < 0) {
            return;
        }

        if (target == 0) {
            ans.add(new ArrayList<>(combination));   // COPY, not the live reference
            return;
        }

        // pick current element  -> index STAYS (reuse allowed)
        int currEle = candidates[index];
        combination.add(currEle);
        findCombination(index, target - currEle, candidates, combination, ans);
        combination.remove(combination.size() - 1); // undo (backtrack)

        // don't pick current element -> index MOVES ON, forever
        findCombination(index + 1, target, candidates, combination, ans);
    }
}
```

**Two-line summary of the whole algorithm:**
> At every element, either take it (and stay, so you can take it again) or skip it (and never look at it again). Record when target hits exactly 0.

---

## 4. Bugs hit along the way — the real lesson

These are the ones that actually cost time. Every backtracking bug I hit from here will probably be one of these.

### Bug 1 — Branches swapped → infinite recursion → **TLE**

```java
// WRONG
findCombination(index + 1, target - currEle, ...);  // pick
findCombination(index,     target,           ...);  // skip  <-- INFINITE
```

The skip branch had the **same index and same target** as the caller. Nothing changes between calls → no progress → stack grows forever.

**Diagnostic habit:** for every recursive call, ask *"what strictly decreases here?"* If nothing does, it never terminates.
- Pick branch: `target` decreases (since `candidates[i] >= 2 > 0`). ✅
- Skip branch: `index` increases. ✅

Also note the intuition for *which* branch stays: **reuse happens on pick**, so pick is the one that must not advance.

### Bug 2 — Hardcoded removal value

```java
combination.remove(Integer.valueOf(5));   // WRONG — why 5?!
combination.remove(combination.size() - 1); // RIGHT
```

**Java gotcha worth memorising — `List` has two `remove` overloads:**

| Call | Overload resolved | Meaning | Cost |
|---|---|---|---|
| `list.remove(3)` | `remove(int index)` | remove element **at position 3** | O(n) shift |
| `list.remove(Integer.valueOf(3))` | `remove(Object o)` | remove **first element equal to 3** | O(n) scan + shift |

For a `List<Integer>` this is a classic trap. In backtracking you **always** want by-index-of-last:
- by-value removes the *first* match, not the one you just added → wrong element gone when duplicates exist
- by-value is an O(n) linear scan; by-last-index is effectively O(1)

### Bug 3 — `ans.add(combination)` stores a live reference

```java
ans.add(combination);                    // WRONG
ans.add(new ArrayList<>(combination));   // RIGHT
```

There is exactly **one** `ArrayList` object created (in `combinationSum`) and it's passed down the entire recursion. Adding it directly puts N pointers to the *same* object into `ans`. As the recursion unwinds, the backtracking `remove()` calls empty that object — so `ans` ends up as a list of identical (usually empty) lists.

**Rule:** *copy on record, undo on return.* Both halves are mandatory.

```
   Live reference (WRONG)              Snapshot (RIGHT)
   ans ──┐                             ans ──> [copy of [2,2,3]]
         ├──> combination  <-- mutated        ──> [copy of [7]]
   ans ──┘     by backtracking          combination is free to mutate
```

### Non-bug worth knowing — base-case ordering

```java
if (index == candidates.length || target < 0) return;
if (target == 0) { record; return; }
```

Checking `index == length` *before* `target == 0` looks dangerous — could we return without recording a valid answer?

**Answer: no, it's safe here.** Reasoning:
- `target` only decreases on the **pick** branch, where `index` is unchanged (and was valid, since we just read `candidates[index]`).
- The **skip** branch passes `target` through unchanged. But we can only *reach* the skip branch if the parent's target was ≠ 0 (otherwise it recorded and returned).
- The initial call has `target >= 1` per constraints.
- ⇒ A call with `target == 0` always has a valid `index`. No answer is ever lost.

**But it's fragile logic that depends on the branch structure.** Put `target == 0` first anyway — it's free and makes correctness obvious to a reader (and to an interviewer).

---

## 5. Dry run — `candidates = [2,3,6,7]`, `target = 7`

`(idx, target, combination)` — `P` = pick, `S` = skip

```
(0, 7, [])
├─P─ (0, 5, [2])
│    ├─P─ (0, 3, [2,2])
│    │    ├─P─ (0, 1, [2,2,2])
│    │    │    ├─P─ target = -1            ✗ prune
│    │    │    └─S─ (1, 1, [2,2,2])
│    │    │         ├─P─ target = -2       ✗
│    │    │         └─S─ (2, 1, ...) ... (3, 1, ...) ... (4, ...) ✗ index == length
│    │    └─S─ (1, 3, [2,2])
│    │         ├─P─ (1, 0, [2,2,3])        ✅ RECORD [2,2,3]
│    │         └─S─ (2, 3, [2,2]) ...      ✗ 6 and 7 both overshoot
│    └─S─ (1, 5, [2])
│         ├─P─ (1, 2, [2,3])
│         │    ├─P─ target = -1            ✗
│         │    └─S─ (2, 2, [2,3]) ...      ✗
│         └─S─ (2, 5, [2]) ...             ✗
└─S─ (1, 7, [])
     ├─P─ (1, 4, [3])
     │    ├─P─ (1, 1, [3,3]) ...           ✗
     │    └─S─ (2, 4, [3]) ...             ✗
     └─S─ (2, 7, [])
          ├─P─ (2, 1, [6]) ...             ✗
          └─S─ (3, 7, [])
               ├─P─ (3, 0, [7])            ✅ RECORD [7]
               └─S─ (4, ...)               ✗ index == length
```

**Output:** `[[2,2,3], [7]]` ✅

Notice: `[3,2,2]` never appears. The index-only-moves-forward rule killed it structurally.

---

## 6. Complexity — with the intuition, not just the formula

### Time: **O(2^t × k)**
where `t = target`, `k = average length of a recorded combination`

**Why NOT `O(2^n)` — this is the part I got wrong first:**

`2^n` is the answer for **subsets**, where each of `n` elements gets one binary decision, so the tree depth is `n`.

Here the pick branch **doesn't advance the index**, so depth is *not* bounded by `n`:
```
candidates = [2], target = 40
→ pick 2, pick 2, pick 2, ... 20 times deep
→ from a single candidate (n = 1)!
```

Depth is bounded by `target / min(candidates[i])`. So the binary tree has `~2^t` nodes, not `2^n`.

**Where the `× k` comes from:** the `new ArrayList<>(combination)` at each recorded leaf copies up to `k` elements. That copy is not free and interviewers do ask about it.

### Space: **split it into two — always state both separately**

| Component | Complexity | Why |
|---|---|---|
| **Auxiliary** (the one that matters) | **O(t)** | recursion stack depth + the `combination` list; worst case `target / min(cand)` |
| **Output** (`ans`) | O(#combinations × k) | required return value, conventionally excluded — but *say so out loud* rather than silently dropping it |

### The interview-scoring point

> This is exponential and **no polynomial algorithm exists** — because the *output itself* can be exponentially large. You cannot enumerate `2^t` combinations in less than `2^t` time. The bound is inherent to the problem, not a weakness in the approach.

Saying this shows you understand output-sensitive lower bounds. It's a genuinely strong signal.

---

## 7. Further optimisations (beyond the accepted solution)

### 7.1 Prune *before* recursing, not after

Current code makes the call, then the child returns immediately on `target < 0`. Cheap to avoid:

```java
if (currEle <= target) {
    combination.add(currEle);
    findCombination(index, target - currEle, candidates, combination, ans);
    combination.remove(combination.size() - 1);
}
findCombination(index + 1, target, candidates, combination, ans);
```

Saves one full level of wasted stack frames. Same asymptotic complexity, real constant-factor win.

### 7.2 Sort + `break` (the for-loop formulation)

```java
private void solve(int start, int target, int[] candidates,
                   List<Integer> comb, List<List<Integer>> ans) {
    if (target == 0) { ans.add(new ArrayList<>(comb)); return; }

    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > target) break;   // sorted ⇒ everything after also too big
        comb.add(candidates[i]);
        solve(i, target - candidates[i], candidates, comb, ans);  // i, not i+1 → reuse
        comb.remove(comb.size() - 1);
    }
}
// call with Arrays.sort(candidates) first
```

**Pick/not-pick vs for-loop — which to use?**

| | Pick / not-pick | For-loop |
|---|---|---|
| Tree | binary | n-ary |
| Mental model | easier to reason about, closer to subsets | more compact, prunes better |
| Pruning | needs explicit guard | `break` kills whole tail at once |
| Extends to LC 40 dedup | awkward | **natural** (`i > start && cand[i]==cand[i-1]`) |

**Recommendation:** know pick/not-pick for intuition, but **write the for-loop version in interviews** — it generalises cleanly to the whole family (LC 40, 216, 77, 90).

---

## 8. Reusable template — commit this to muscle memory

```java
void backtrack(int start, State state, List<Integer> path, List<List<Integer>> ans) {
    if (isSolution(state)) {
        ans.add(new ArrayList<>(path));   // ← COPY
        return;
    }
    if (isDeadEnd(state)) return;

    for (int i = start; i < n; i++) {
        if (shouldPrune(i, state)) break;       // sorted-input pruning
        if (isDuplicate(i, start)) continue;    // LC 40 / 90 dedup

        path.add(choice(i));                    // choose
        backtrack(reuseAllowed ? i : i + 1,     // ← THE knob that defines the problem
                  advance(state, i), path, ans);
        path.remove(path.size() - 1);           // ← UNDO
    }
}
```

**The one knob that defines each problem: `i` vs `i + 1`.**

---

## 9. Pre-submit checklist for any backtracking problem

- [ ] Does every recursive call make something **strictly** decrease/increase? (else infinite recursion)
- [ ] Am I **copying** on record — `new ArrayList<>(path)`?
- [ ] Am I **undoing** by last index — `remove(path.size() - 1)`, never `remove(Integer.valueOf(x))`?
- [ ] Is the success check (`target == 0`) **before** the failure checks?
- [ ] Is `i` vs `i + 1` correct for reuse semantics?
- [ ] Are duplicate combinations prevented **structurally** (forward-only index), not by a HashSet?
- [ ] Can I state time as `2^depth × copy-cost`, and split space into auxiliary vs output?

---

## 10. Next problems — same skeleton, one twist each

| # | Problem | The twist | Sheets |
|---|---|---|---|
| **40** | Combination Sum II | each element used **once**, input has **duplicates** → sort, then `i > start && cand[i] == cand[i-1] → continue`. The fix is *outside* the recursion structure. | NC, SDE, AC, LB |
| **216** | Combination Sum III | fixed size `k`, candidates fixed as 1–9 | NC, SDE |
| **77** | Combinations | pure "choose k from n" | SDE |
| **78** | Subsets | no target — record at **every** node, not just leaves | NC, SDE, AC, LB |
| **90** | Subsets II | duplicates + no target → same dedup trick as 40 | NC, SDE, LB |
| **131** | Palindrome Partitioning | choice is a *substring*, not an element | NC, SDE |

**Order:** 40 → 78 → 90 → 216 → 131. Once 40 clicks, the rest go fast — the skeleton is identical.