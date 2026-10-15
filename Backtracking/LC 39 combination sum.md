# LC 39 — Combination Sum

**Sheets:** NeetCode 150 | Striver A2Z (Recursion → Subsequences) | Love Babbar 450 | Apna College
**Pattern:** Backtracking — pick / not-pick on an *unbounded* candidate
**Difficulty:** Medium

---

## 1. Problem

Given `candidates[]` of **distinct** integers and a `target`, return **all unique combinations**
that sum to `target`.

- The **same number may be reused unlimited times**.
- Two combinations differ if the *multiset* of chosen numbers differs.
- Order doesn't matter, within a combination or between them.

**Constraints — they drive the complexity:**
```
1 <= candidates.length <= 30
2 <= candidates[i] <= 40
1 <= target <= 40
```

> The tiny `target <= 40` is the whole reason this is solvable. It tells you the recursion depth is
> bounded by **target**, not by n.

---

## 2. The key observation — read this before writing code

**Unlimited reuse changes the shape of the recursion tree.**

| Problem type | On "pick", index moves to | Depth bounded by |
|---|---|---|
| Subsets / Subsequences | `index + 1` | `n` |
| **Combination Sum (this)** | **`index` — stays!** | **`target / min(cand)`** |
| Combination Sum II (LC 40) | `index + 1` | `n` |

> ### 🔑 The fact
> **Every candidate is either in or out — and because reuse is allowed, choosing "in" doesn't
> advance the index; only "out" does.**

That single clause is what separates this from every subsets problem, and it's the knob you change
to get LC 40.

---

## 3. Approaches

### 3.1 Why loops can't do it

The first instinct — nested loops over pairs, triples — fails because **the combination length
isn't fixed**. `target = 40, candidates = [2]` needs a combination of length 20; any fixed nesting
depth `d` only finds combinations of exactly length `d`.

> **Unknown or variable depth ⇒ recursion is mandatory.** That's the signal this is a backtracking
> problem, not a loop problem.

### 3.2 Avoiding permutation duplicates — structurally

Generate every multiset and filter, and you risk emitting both `[2,3]` and `[3,2]`.

**The fix is structural, not a dedup pass:** by only ever moving the index *forward*, every
combination is generated in non-decreasing index order. `[3,2]` can never be produced, because once
you've moved past index 0 you can't go back.

**No HashSet needed. Dedup by construction, not by filtering.**

### 3.3 Pick / not-pick recursion — the solution

```java
class Solution {
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        List<List<Integer>> ans = new ArrayList<>();
        findCombination(0, target, candidates, new ArrayList<>(), ans);
        return ans;
    }

    private void findCombination(int index, int target, int[] candidates,
                                 List<Integer> combination, List<List<Integer>> ans) {
        if (target == 0) {
            ans.add(new ArrayList<>(combination));   // COPY, not the live reference
            return;
        }
        if (index == candidates.length || target < 0) return;

        // 1. PICK current element -> index STAYS (reuse allowed)
        int curr = candidates[index];
        combination.add(curr);
        findCombination(index, target - curr, candidates, combination, ans);
        combination.remove(combination.size() - 1);          // undo

        // 2. SKIP current element -> index MOVES ON, forever
        findCombination(index + 1, target, candidates, combination, ans);
    }
}
```

**Both calls are unconditional and sequential.** Not a branch — see Bug 1.

> **Two-line summary:** at every element, either take it (and stay, so you can take it again) or
> skip it (and never look at it again). Record when target hits exactly 0.

---

## 4. The bugs that actually cost time

### 🚩 Bug 1 — `if/else` instead of two calls

```java
if (candidates[index] <= target) {          // 🚩
    combination.add(candidates[index]);
    findCombination(index, target - candidates[index], ...);
    combination.remove(combination.size() - 1);
} else {
    findCombination(index + 1, target, ...);
}
```

With an `else`, each index is **either** taken **or** skipped — never both. The recursion walks a
single path down the tree and produces at most one combination.

**Pick and skip are not alternatives. They are two independent subtrees**, and both must be
explored at every node. The `else` has to go; the two calls run one after the other.

This is the bug that blocks people on this problem, and it's easy to miss because the code looks
reasonable and even returns *a* valid combination sometimes.

### 🚩 Bug 2 — `ans.add(combination)` stores a live reference

```java
ans.add(combination);                    // WRONG
ans.add(new ArrayList<>(combination));   // RIGHT
```

Exactly **one** `ArrayList` is created and passed down the whole recursion. Adding it directly puts
N pointers to the *same* object into `ans`; as the recursion unwinds, the backtracking `remove()`
calls empty it — so `ans` ends up as a list of identical, usually empty, lists.

```
   Live reference (WRONG)              Snapshot (RIGHT)
   ans ──┐                             ans ──> [copy of [2,2,3]]
         ├──> combination  <-- mutated        ──> [copy of [7]]
   ans ──┘     by backtracking          combination is free to mutate
```

**Rule: copy on record, undo on return.** Both halves are mandatory.

(Same distinction as a per-level list in BFS: a **reused** buffer must be copied; a **fresh** object
per iteration doesn't need to be.)

### 🚩 Bug 3 — recursing after recording a solution

```java
if (target == 0) {
    ans.add(new ArrayList<>(combination));
    findCombination(index, target - candidates[index], ...);   // 🚩
    return;
}
```

A complete combination has been found. There is nothing further to explore down that path — record
and return. The extra call explores a subtree with a negative target, which does nothing but burn
frames (or worse, if the base cases don't catch it).

### 🚩 Bug 4 — branches swapped → infinite recursion → TLE

```java
findCombination(index + 1, target - curr, ...);  // "pick" advancing the index
findCombination(index,     target,        ...);  // "skip" standing still  ← INFINITE
```

The skip branch has the **same index and same target** as its caller. Nothing changes between
calls, so there's no progress.

> **Diagnostic habit:** for every recursive call, ask *"what strictly changes here?"* If nothing
> does, it never terminates.
> - Pick: `target` decreases (candidates ≥ 2 > 0) ✅
> - Skip: `index` increases ✅

And the intuition for *which* branch stays put: **reuse happens on pick**, so pick is the one that
must not advance.

### 🚩 Bug 5 — the `List.remove` overload trap

```java
combination.remove(Integer.valueOf(5));        // WRONG — and why 5?
combination.remove(combination.size() - 1);    // RIGHT
```

| Call | Overload | Meaning | Cost |
|---|---|---|---|
| `list.remove(3)` | `remove(int index)` | remove element **at position 3** | O(n) shift |
| `list.remove(Integer.valueOf(3))` | `remove(Object o)` | remove the **first element equal to 3** | O(n) scan + shift |

For a `List<Integer>` this is a classic trap. In backtracking you **always** want by-last-index:
by-value removes the *first* match rather than the one you just added, which is wrong whenever the
combination contains duplicates.

> **Java 21+:** `combination.removeLast()` reads better and is what `SequencedCollection` added.
> Older Java needs `remove(combination.size() - 1)`.

### Non-bug worth understanding — base-case ordering

```java
if (index == candidates.length || target < 0) return;
if (target == 0) { record; return; }
```

Checking `index == length` *before* `target == 0` looks dangerous. Could a valid answer be
discarded?

**No, and here's why:** `target` only decreases on the **pick** branch, where `index` is unchanged
and was valid (you just read `candidates[index]`). The **skip** branch passes `target` through
unchanged, and it's only reachable if the parent's target was ≠ 0 — otherwise the parent recorded
and returned. The initial call has `target ≥ 1`. So a call with `target == 0` always has a valid
index.

**But that's correct-by-accident reasoning that depends on the exact branch structure.** Put
`target == 0` first anyway — it's free, and it makes correctness obvious instead of requiring a
proof.

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
│    │    │         └─S─ (2, 1, ...) ... (4, ...) ✗ index == length
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

Two things visible in the tree: **every node has both a P and an S child** (Bug 1's absence), and
**`[3,2,2]` never appears** — the forward-only index killed it structurally.

---

## 6. Complexity

### How to compute it for an enumeration problem

> **Tree nodes × work per node.** No cache flattens the tree here, so you bound the tree directly.
>
> This is a *different recipe* from memoised DP, where it's *states × work per state*.

### Time: O(2^(target/min) × k)

where `k` is the average length of a recorded combination.

**Why not `O(2^n)`:** `2^n` is the subsets answer, where each of `n` elements gets one binary
decision and the depth is `n`. Here the **pick branch doesn't advance the index**, so depth isn't
bounded by `n`:

```
candidates = [2], target = 40
→ pick 2, pick 2, ... 20 levels deep, from a single candidate (n = 1)
```

Depth is bounded by `target / min(candidates)`. The commonly quoted bound is
**O(N^(target/min + 1))**.

**The `× k`:** `new ArrayList<>(combination)` at each recorded leaf copies up to `k` elements. Not
free, and interviewers ask.

### Space — state both separately

| Component | Complexity | Why |
|---|---|---|
| **Auxiliary** | **O(n + target/min)** | recursion depth = at most `n` skips **plus** `target/min` picks; the `combination` list is the same bound, so it adds no term |
| **Output** (`ans`) | O(#combinations × k) | the required return value — conventionally excluded, but **say so** rather than silently dropping it |

> Not just `O(n)`: a pick doesn't advance the index, so a path can be far deeper than the candidate
> count. Not just `O(target/min)` either: skips contribute up to `n` more frames.

### The interview-scoring point

> This is exponential and **no polynomial algorithm exists** — because the *output itself* can be
> exponentially large. You cannot enumerate `2^t` combinations in less than `2^t` time. The bound
> is inherent to the problem, not a weakness in the approach.

Saying that shows you understand output-sensitive lower bounds. Strong signal.

---

## 7. Further optimisations

### 7.1 Prune before recursing

The base version makes the call and lets the child return immediately on `target < 0`. Cheap to
avoid:

```java
if (curr <= target) {
    combination.add(curr);
    findCombination(index, target - curr, candidates, combination, ans);
    combination.remove(combination.size() - 1);
}
findCombination(index + 1, target, candidates, combination, ans);
```

Saves a full level of wasted frames. Same asymptotics, real constant-factor win.

Note the guard and the `target < 0` base case are **alternatives**, not both needed. Know which
you chose: the guard prunes one level earlier; the base case lets the call happen and bail
immediately.

### 7.2 Sort + `break` — the for-loop formulation

```java
private void solve(int start, int target, int[] candidates,
                   List<Integer> comb, List<List<Integer>> ans) {
    if (target == 0) { ans.add(new ArrayList<>(comb)); return; }

    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > target) break;    // sorted ⇒ everything after is too big
        comb.add(candidates[i]);
        solve(i, target - candidates[i], candidates, comb, ans);   // i, not i+1 → reuse
        comb.remove(comb.size() - 1);
    }
}
// call Arrays.sort(candidates) first
```

| | Pick / not-pick | For-loop |
|---|---|---|
| Tree | binary | n-ary |
| Mental model | closer to subsets, easier to reason about | more compact, prunes better |
| Pruning | needs an explicit guard | `break` kills the whole tail at once |
| Extends to LC 40 dedup | awkward | **natural** — `i > start && cand[i] == cand[i-1]` |

**Know pick/not-pick for intuition; write the for-loop version in interviews.** It generalises
cleanly to LC 40, 77, 90, 216.

---

## 8. Reusable template

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

**The one knob: `i` vs `i + 1`.**

---

## 9. Pre-submit checklist for any backtracking problem

- [ ] Are pick and skip **both** explored — two calls, no `else`?
- [ ] Does every recursive call make something **strictly** change? (else infinite recursion)
- [ ] Am I **copying** on record — `new ArrayList<>(path)`?
- [ ] Am I **undoing** by last index, never `remove(Integer.valueOf(x))`?
- [ ] Is the success check (`target == 0`) **before** the failure checks?
- [ ] Do I return immediately after recording, with no further recursion?
- [ ] Is `i` vs `i + 1` correct for the reuse semantics?
- [ ] Are duplicates prevented **structurally** (forward-only index), not by a HashSet?
- [ ] Can I state time as `2^depth × copy-cost`, and split space into auxiliary vs output?

---

## 10. Pattern

> **Every candidate is either in or out — and because reuse is allowed, choosing "in" doesn't
> advance the index; only "out" does.**

The transferable half: **enumerate by making one binary decision per element, and prevent
permutation duplicates by only ever moving the index forward.** Whether the pick advances the index
is the single parameter that distinguishes this whole family of problems.

---

## 11. Takeaways

1. **Variable-depth search ⇒ recursion.** Fixed loop nesting can only find fixed-length answers.
2. **Pick and skip are two subtrees, not two branches of an `if`.** Both calls, unconditional.
3. **Reuse ⇒ the pick doesn't advance the index.** That one line defines the problem.
4. **Forward-only index kills permutation duplicates structurally** — no HashSet.
5. **Copy on record, undo on return.** Both halves, every time.
6. **Record and return** — never recurse after finding a solution.
7. **Every call must change something strictly**, or it doesn't terminate.
8. **`remove(size() - 1)`, never `remove(Integer.valueOf(x))`.**
9. **Put the success check first** so correctness doesn't depend on a subtle argument.
10. **Enumeration complexity = tree nodes × work per node**, not states × work per state.
11. **Depth here is `target/min`, not `n`** — the pick branch doesn't consume a candidate.
12. **Exponential is the lower bound** when the output is exponential. Say so.