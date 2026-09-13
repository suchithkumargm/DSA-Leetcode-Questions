# LC 455 — Assign Cookies

**Difficulty:** Easy
**Pattern:** Greedy (sort + two pointers) · Bipartite matching in disguise
**Sheets:** NeetCode 150 (Greedy) | Love Babbar 450 | Apna College
**Status:** Solved — 99.31% runtime

---

## 1. Problem

You have `n` children and `m` cookies.

- `g[i]` = **greed factor** of child `i` — the minimum cookie size that will content them.
- `s[j]` = **size** of cookie `j`.

Rules:
- Each child gets **at most one** cookie.
- Cookie `j` contents child `i` **iff** `s[j] >= g[i]`.
- Each cookie is used **at most once**.

**Goal:** maximize the number of content children.

### Constraints (matter for the solution)

```
1 <= g.length <= 3 * 10^4
0 <= s.length <= 3 * 10^4        <-- s can be EMPTY
1 <= g[i], s[j] <= 2^31 - 1      <-- values near Integer.MAX_VALUE
```

Two things to notice immediately:
1. `s.length` can be `0` → answer is `0`. Your loop must not blow up.
2. Values go up to `Integer.MAX_VALUE` → **never** compare via subtraction (`s[j] - g[i] >= 0` overflows).

### Examples

```
g = [1,2,3], s = [1,1]     -> 1     (one cookie of size 1 contents child 0)
g = [1,2],   s = [1,2,3]   -> 2     (both children served, one cookie wasted)
g = [2,3,5], s = [1,2,4,6] -> 3
g = [7,8,9,10], s = [5,6,7,8] -> 2
```

---

## 2. Recognising what this actually is

Strip the story away and it is:

> Given two multisets `G` and `S`, find the **maximum matching** where each `g` is
> paired with a distinct `s` satisfying `s >= g`.

This is a bipartite matching problem. General bipartite matching is expensive (Hopcroft–Karp, `O(E√V)`). But this graph has a special structure:

> If cookie `s` can serve child `g`, it can serve **every child less greedy than `g`**.

That nesting/monotonic structure is exactly what makes a **greedy** solution valid and lets us skip all the matching machinery. Whenever you see "compatibility is defined by a single `>=` threshold," reach for sort + greedy, not matching.

---

## 3. The journey: brute → better → optimal

### Approach 1 — Brute force (try every assignment)

Recursively decide, for each child, which unused cookie to give them.

```java
// Conceptual only — do NOT submit this
int solve(int i, int[] g, int[] s, boolean[] used) {
    if (i == g.length) return 0;

    int best = solve(i + 1, g, s, used);          // skip this child entirely

    for (int j = 0; j < s.length; j++) {
        if (!used[j] && s[j] >= g[i]) {
            used[j] = true;
            best = Math.max(best, 1 + solve(i + 1, g, s, used));
            used[j] = false;                       // backtrack
        }
    }
    return best;
}
```

- **Time:** `O(m^n)` in the worst case — every child branches over every cookie. Even memoised on a bitmask of cookies it is `O(n * m * 2^m)`.
- **Space:** `O(n)` recursion + `O(m)` used array.
- **Verdict:** TLE past `m ≈ 20`. Useful only as a mental baseline / correctness oracle for small random tests.

**The insight it gives you:** the only real decision is *which* cookie to hand a child. If we can prove one specific choice is always safe, the entire search tree collapses to a single path.

---

### Approach 2 — Greedy, but with a linear search (the first correct idea)

The safe choice: **give a child the smallest cookie that still fits them.**

Intuition — spending a size-10 cookie on a greed-2 child is pure waste. The size-10 cookie is strictly more "powerful": anything a small cookie can do, a big one can do too. So never burn a big cookie when a small one suffices. Save the big ones for the children who genuinely need them.

```java
// Correct, but slow
public int findContentChildren(int[] g, int[] s) {
    boolean[] used = new boolean[s.length];
    int ans = 0;

    for (int i = 0; i < g.length; i++) {
        int bestIdx = -1;
        for (int j = 0; j < s.length; j++) {
            if (!used[j] && s[j] >= g[i]) {
                if (bestIdx == -1 || s[j] < s[bestIdx]) bestIdx = j;  // smallest fit
            }
        }
        if (bestIdx != -1) { used[bestIdx] = true; ans++; }
    }
    return ans;
}
```

- **Time:** `O(n * m)` → `3*10^4 * 3*10^4 = 9 * 10^8`. TLE.
- **Space:** `O(m)` for the `used` array.
- **Verdict:** correct algorithm, bad implementation. The greedy rule is already right; the bottleneck is *finding* the smallest fitting cookie, which we redo from scratch `n` times.

---

### Approach 3 — Sort + binary search / TreeMap multiset

The repeated work is "find min element `>= x` in the remaining cookies, then remove it." That is textbook `TreeMap`.

```java
public int findContentChildren(int[] g, int[] s) {
    TreeMap<Integer, Integer> pool = new TreeMap<>();
    for (int size : s) pool.merge(size, 1, Integer::sum);

    int ans = 0;
    for (int greed : g) {
        Map.Entry<Integer, Integer> e = pool.ceilingEntry(greed);  // smallest cookie >= greed
        if (e == null) continue;                                   // nobody can serve this child
        ans++;
        if (e.getValue() == 1) pool.remove(e.getKey());
        else pool.put(e.getKey(), e.getValue() - 1);
    }
    return ans;
}
```

- **Time:** `O(m log m)` to build + `O(n log m)` to query = **`O((n + m) log m)`**.
- **Space:** `O(m)` for the map, plus heavy boxing overhead (`Integer` objects, tree nodes).
- **Verdict:** passes, but ~5–10× slower than the two-pointer version in practice. Every `ceilingEntry` chases pointers through tree nodes; cache locality is terrible compared to a sorted `int[]`.

**Worth keeping in your head anyway** — this is the shape you need when the cookie pool is *dynamic* (cookies arriving/being removed mid-stream). Two pointers only work on a fixed, fully-known set.

---

### Approach 4 — Sort both + two pointers ✅ (optimal)

If we sort **both** arrays ascending, the "smallest fitting cookie" for the current child is simply **the next cookie we haven't discarded**. No search needed at all.

```java
class Solution {
    public int findContentChildren(int[] g, int[] s) {
        Arrays.sort(g);
        Arrays.sort(s);

        int i = 0, j = 0, ans = 0;

        while (i < g.length && j < s.length) {
            if (s[j] >= g[i]) {   // cookie j serves child i
                i++;
                j++;
                ans++;
            } else {              // cookie j is too small for EVERYONE left -> discard it
                j++;
            }
        }

        return ans;
    }
}
```

- **Time:** `O(n log n + m log m)` — fully dominated by the two sorts. The scan itself is `O(n + m)`.
- **Space:** `O(log n)` — see §6, it is *not* `O(1)`.

**Tighter equivalent** (fewer moving parts under interview pressure — `i` doubles as the answer since it only advances on a match):

```java
Arrays.sort(g);
Arrays.sort(s);
int i = 0;
for (int j = 0; j < s.length && i < g.length; j++) {
    if (s[j] >= g[i]) i++;
}
return i;
```

---

## 4. Why the two-pointer step is safe — the key invariant

Everything hinges on one line:

> **If `s[j] < g[i]`, cookie `j` is useless forever. Discard it (`j++` only).**

Why? Because `g` is sorted ascending, so `g[i] <= g[i+1] <= g[i+2] <= ...`. If cookie `j` is too small for the *least greedy remaining* child, it is too small for every remaining child. There is no future in which it gets used. Throw it away with zero regret.

Symmetrically:

> **If `s[j] >= g[i]`, take the match immediately.**

Cookie `j` is the smallest surviving cookie. Using it on child `i` costs us the least possible resource. Holding it back to "maybe use later" can never help — any later child it could serve can also be served by a bigger cookie we still hold.

Because each comparison permanently advances at least one pointer, the scan is `O(n + m)`.

### Formal justification — exchange argument

Claim: matching the smallest fitting cookie to the least greedy child is contained in some optimal solution.

Let `OPT` be an optimal assignment. Let child `i` be the least greedy child and `s[j]` the smallest cookie with `s[j] >= g[i]`.

1. **If `OPT` leaves child `i` unserved** but uses `s[j]` on some child `k`: since `g[k] >= g[i]`… careful — we need `s[j]` to still be valid. It is: `s[j] >= g[k] >= g[i]`. Reassign `s[j]` to child `i` and drop child `k`. Count unchanged → still optimal, and now matches our choice.
2. **If `OPT` serves child `i` with a different cookie `s[k]`** (necessarily `s[k] >= g[i]`, and `s[k] >= s[j]` since `s[j]` is the smallest fit): swap them. If `s[j]` was unused, the swap is free. If `s[j]` served some child `p`, then `g[p] <= s[j] <= s[k]`, so `s[k]` serves child `p` just fine. Count unchanged → still optimal.

Either way we transform `OPT` into an equally good solution that agrees with our greedy choice. Remove child `i` and cookie `s[j]`, recurse on the subproblem. Induction finishes it.

**The one-liner to say in an interview:** *"Swapping a bigger cookie for the smallest fitting one never breaks anyone else's assignment, because the bigger cookie can cover everything the smaller one could."*

---

## 5. Dry runs

### Dry run A — hits both branches

```
g = [5, 2, 3]   ->  sorted: [2, 3, 5]
s = [4, 1, 6, 2] -> sorted: [1, 2, 4, 6]
```

| Step | i | j | g[i] | s[j] | `s[j] >= g[i]`? | Action | ans |
|------|---|---|------|------|------------------|--------|-----|
| 1 | 0 | 0 | 2 | 1 | no  | discard cookie → `j++` | 0 |
| 2 | 0 | 1 | 2 | 2 | yes | match → `i++, j++` | 1 |
| 3 | 1 | 2 | 3 | 4 | yes | match → `i++, j++` | 2 |
| 4 | 2 | 3 | 5 | 6 | yes | match → `i++, j++` | 3 |
| 5 | 3 | 4 | — | — | — | `i == g.length` → exit | 3 |

**Return 3.** Note step 1: the size-1 cookie is dropped permanently and never reconsidered.

---

### Dry run B — cookies run out first

```
g = [10, 9, 8, 7] -> sorted: [7, 8, 9, 10]
s = [5, 6, 7, 8]  -> sorted: [5, 6, 7, 8]
```

| Step | i | j | g[i] | s[j] | `s[j] >= g[i]`? | Action | ans |
|------|---|---|------|------|------------------|--------|-----|
| 1 | 0 | 0 | 7 | 5 | no  | `j++` | 0 |
| 2 | 0 | 1 | 7 | 6 | no  | `j++` | 0 |
| 3 | 0 | 2 | 7 | 7 | yes | `i++, j++` | 1 |
| 4 | 1 | 3 | 8 | 8 | yes | `i++, j++` | 2 |
| 5 | 2 | 4 | 9 | — | — | `j == s.length` → exit | 2 |

**Return 2.** Children 9 and 10 go hungry — correct, nothing left is big enough.

---

### Dry run C — the counterexample for the classic bug

Suppose the `else` branch wrongly did `i++` instead of `j++`.

```
g = [5, 6], s = [3, 7]   (both already sorted)
```

| Step | i | j | g[i] | s[j] | Wrong code (`i++`) | Correct code (`j++`) |
|------|---|---|------|------|--------------------|----------------------|
| 1 | 0 | 0 | 5 | 3 | `3 < 5` → `i++` → i=1 | `3 < 5` → `j++` → j=1 |
| 2 | — | — | 6 / 5 | 3 / 7 | `3 < 6` → `i++` → i=2, **exit, ans=0** | `7 >= 5` → match, **ans=1** |

Wrong answer `0`, correct answer `1`. The wrong version throws away a *child* who was perfectly servable by a later cookie. **Children are the thing we're maximising — never discard one while cookies remain.** Only cookies get discarded.

---

## 6. Complexity — with the intuition

### Time: `O(n log n + m log m)`

| Phase | Cost | Why |
|-------|------|-----|
| `Arrays.sort(g)` | `O(n log n)` | dual-pivot quicksort on primitives |
| `Arrays.sort(s)` | `O(m log m)` | same |
| two-pointer scan | `O(n + m)` | each iteration advances `j` by 1; `j` bounded by `m`, `i` by `n` |

The scan is linear, so sorting dominates. Commonly written `O(n log n)` treating `n ≈ m`, but state the precise form if the interviewer is picky.

**Can we beat `n log n`?** Only if values are bounded — then counting sort gives `O(n + m + V)`. Here `V` is up to `2^31`, so no. Comparison-based sorting is the floor.

### Space: `O(log n)` — **not** `O(1)`

This is the detail that catches people out, and interviewers do ask.

- `Arrays.sort(int[])` → **dual-pivot quicksort**, which recurses. Stack depth is `O(log n)` expected. So auxiliary space is `O(log n)`, not `O(1)`.
- `Arrays.sort(Integer[])` → **TimSort** (stable, required for objects), which allocates a merge buffer → `O(n)` space.

**Safe interview phrasing:** *"`O(1)` extra beyond the sort — the sort itself costs `O(log n)` for the quicksort recursion stack."*

> **Recurring pitfall, already in your notes:** boxing matters. `int[]` vs `Integer[]` changes both the algorithm used and the space complexity. Same trap as `List.remove(int)` vs `remove(Object)` — the type silently picks a different implementation.

---

## 7. Common bugs & gotchas

| # | Bug | Why it breaks | Fix |
|---|-----|---------------|-----|
| 1 | `i++` in the `else` branch | Discards a servable child (Dry run C) | Only `j++` — cookies get discarded, never children |
| 2 | Forgetting to sort one array | Invariant "too small for `g[i]` ⟹ too small for all" collapses | Sort **both** |
| 3 | Sorting descending without rewriting the logic | Comparison direction flips | If descending, match greediest child to biggest cookie and the `else` discards the *child* |
| 4 | `s[j] - g[i] >= 0` instead of `s[j] >= g[i]` | Values up to `2^31 - 1` → **integer overflow** → wrong sign | Always direct comparison. (Same family as preferring `Integer.compare` over subtraction in comparators.) |
| 5 | `s.length == 0` not handled | Constraint explicitly allows it | The `while` guard `j < s.length` handles it; a `for (int j...)` loop does too. Don't index `s[0]` blindly |
| 6 | Returning `j` instead of `ans` | `j` counts cookies *examined*, not matches | Return `ans` (or `i`, which only moves on a match) |
| 7 | Using `Integer[]` + `Arrays.sort` "to use a comparator" | TimSort + boxing → `O(n)` space, ~3× slower | Keep `int[]`; you don't need a comparator for natural ascending order |

---

## 8. Follow-ups an interviewer might throw at you

1. **"What if a child can be contented by multiple cookies summing to their greed?"**
   Completely different problem — that's bin packing / subset sum, NP-hard. Greedy no longer works.

2. **"What if you must also minimise total wasted cookie size?"**
   The same greedy still wins. Smallest-fitting-cookie minimises waste at every step, so it's simultaneously optimal for both objectives.

3. **"Cookies arrive as a stream and can be withdrawn."**
   Two pointers require the full set up front. Switch to the `TreeMap` multiset from Approach 3 — `O(log m)` per insert/delete/query.

4. **"Prove your greedy is correct."**
   Exchange argument from §4. Have it ready — for Easy greedy problems the *proof* is what's actually being tested, not the code.

5. **"Arrays too large to fit in memory."**
   External merge sort both files, then stream the two sorted files with the same two-pointer logic. The scan phase needs only `O(1)` memory.

---

## 9. Related problems

### Literally the same problem

- **LC 2410 — Maximum Matching of Players With Trainers.** Identical: `players[i] <= trainers[j]`. Same code, renamed variables. Free solve.

### Same pattern: sort + greedy matching

- **LC 870 — Advantage Shuffle** (Medium) — sort both, greedily beat the opponent with the smallest card that wins; otherwise sacrifice your weakest. Direct escalation of this idea.
- **LC 881 — Boats to Save People** (Medium) — sorted two pointers, but from **opposite ends**. Good contrast: here both pointers move forward; there they converge.
- **LC 1710 — Maximum Units on a Truck** ✅ (done) — sort by value density, take greedily.
- **LC 1200 — Minimum Absolute Difference** ✅ (done) — sorting makes the answer adjacent.

### Greedy interval family (already covered)

- **LC 435 — Non-overlapping Intervals** ✅
- **LC 646 — Maximum Length of Pair Chain** ✅
- **LC 56 — Merge Intervals** ✅

### Suggested next steps in Greedy

- **LC 134 — Gas Station** (Medium) — greedy where the safe choice is genuinely non-obvious; needs its own proof.
- **LC 621 — Task Scheduler** (Medium) — greedy + counting, formula-based insight.
- **LC 45 / 55 — Jump Game II / Jump Game** — greedy reachability, very common in interviews.

---

## 10. Recall card (read this first on revision)

> **Problem:** max children contented, `s[j] >= g[i]`, one cookie each.
> **Pattern:** sort both ascending, two pointers moving forward.
> **Safe choice:** smallest cookie that fits the least greedy child.
> **Invariant:** cookie too small for `g[i]` ⟹ too small for everyone left ⟹ discard it.
> **Pointer rule:** match → `i++, j++`. No match → `j++` **only**. Never `i++` in the `else`.
> **Answer:** `i` (children served), since `i` advances only on a match.
> **Time:** `O(n log n + m log m)` · **Space:** `O(log n)` (quicksort stack — not `O(1)`).
> **Proof:** exchange argument — swapping in a bigger cookie never breaks anyone else's assignment.
> **Trap:** `s[j] - g[i] >= 0` overflows (values up to `2^31 - 1`). Use `>=` directly.

---

## 11. Final solution

```java
class Solution {
    public int findContentChildren(int[] g, int[] s) {
        Arrays.sort(g);
        Arrays.sort(s);

        int i = 0;  // pointer over children (ascending greed) — also the answer
        int j = 0;  // pointer over cookies  (ascending size)

        while (i < g.length && j < s.length) {
            if (s[j] >= g[i]) i++;  // this cookie contents child i
            j++;                    // the cookie is consumed either way
        }

        return i;
    }
}
```

`Runtime: beats 99.31%` · The `j++` outside the `if` merges both branches — the cookie is used up whether it was a match or a discard.