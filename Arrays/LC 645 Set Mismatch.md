# LC 645 — Set Mismatch

**Difficulty:** Easy (deceptively — the O(1) space version is medium-tier)
**Pattern:** Index-as-hash / value-bounded-by-n
**Date solved:** Sep 2026

---

## Problem

You have the set `{1, 2, ..., n}`. Due to an error, **one number got duplicated** — it replaced another number, which is now **missing**.

Given the corrupted array `nums`, return `[duplicate, missing]`.

```
Input:  [1, 2, 2, 4]
Output: [2, 3]        // 2 appears twice, 3 is gone

Input:  [1, 1]
Output: [1, 2]
```

**Constraints that matter:**
- `2 <= n <= 10^4`
- `1 <= nums[i] <= n`   ← **this is the whole problem.** Values are bounded by the array length.

---

## The key observation (remember this, not the code)

> Every value is in `[1, n]`, and the array has length `n`.
> So **every value is a valid index** (after `-1` shift).

That single fact is what unlocks every non-trivial solution here. The array can act as its own hash table. Whenever you see "values in range `[1, n]`" in a constraint, your brain should immediately go: *index-as-hash*.

---

## Approach 1 — HashSet (first attempt)

Walk the array, put things in a set. If you're about to insert something already there, that's your duplicate. Then walk `1..n` and whichever value isn't in the set is the missing one.

```java
class Solution {
    public int[] findErrorNums(int[] nums) {
        int[] ans = new int[2];
        Set<Integer> set = new HashSet<>();

        for (int num : nums) {
            if (set.contains(num)) {
                ans[0] = num;          // duplicate
            }
            set.add(num);
        }

        for (int i = 1; i <= nums.length; i++) {
            if (!set.contains(i)) {
                ans[1] = i;            // missing
                break;
            }
        }

        return ans;
    }
}
```

**Time:** O(n) — two passes, each set op is O(1) average
**Space:** O(n) — the set
**Result:** beats ~38%

### Why it's slow despite being O(n)

Big-O hides the constant factor, and here the constant is large:

| Cost | What's happening |
|---|---|
| **Boxing** | `Set<Integer>` can't store `int`. Every `num` gets wrapped in an `Integer` object. Values > 127 escape the Integer cache, so that's a fresh heap allocation per element. |
| **Hashing** | `hashCode()` + bucket index computation on every `contains` and `add`. |
| **Pointer chasing** | HashMap nodes are scattered on the heap. Each lookup is a cache miss. |
| **Resizing** | The set grows and rehashes as it fills. |

None of this changes the complexity class. All of it changes the wall-clock time.

---

## Approach 2 — Count / marker array ✅ (my chosen solution)

Same shape as the set version, but the "set" is a plain `int[]`.

```java
class Solution {
    public int[] findErrorNums(int[] nums) {
        int[] ans = new int[2];
        int[] set = new int[nums.length + 1];   // index 0 unused

        for (int num : nums) {
            if (set[num] == num) {   // already written → seen before
                ans[0] = num;
            }
            set[num] = num;
        }

        for (int i = 1; i <= nums.length; i++) {
            if (set[i] != i) {       // never written → missing
                ans[1] = i;
                break;
            }
        }

        return ans;
    }
}
```

**Time:** O(n)
**Space:** O(n)
**Result:** beats ~94%

### Why it's fast

- Raw `int` — no boxing, no allocation.
- `set[num]` is a direct memory offset. No hash function.
- The array is one contiguous block → cache friendly, hardware prefetcher loves it.

Same asymptotic complexity, ~3x faster in practice. **Lesson: when your keys are small bounded integers, an array *is* the optimal hash table.**

### The subtle correctness detail

Java initialises `int[]` to all zeros. Valid values are `1..n`, so **0 can never be a legitimate value**. That means `set[num] == num` is true *only* if I wrote it there myself.

> ⚠️ This trick breaks if 0 is in the value domain. Then you'd need `boolean[] seen` or `int[] count` instead. Using the value as its own "seen" flag only works when 0 is outside the domain.

A `count[]` variant is equally valid and slightly more general:

```java
int[] count = new int[n + 1];
for (int num : nums) count[num]++;
for (int i = 1; i <= n; i++) {
    if (count[i] == 2) dup = i;
    if (count[i] == 0) missing = i;
}
```

---

## Approach 3 — Sign marking, O(1) extra space

*(Understand it; don't have to default to it. This is the "can you do better than O(n) space?" answer.)*

**Idea:** stop allocating a second array — use `nums` itself as the marker array. Since every value maps to a valid index, visit index `abs(nums[i]) - 1` and **negate** whatever sits there. Negation is a marker that doesn't destroy information, because `Math.abs()` recovers the original.

- Arrive at a slot that's **already negative** → you've been sent there twice → that's the **duplicate**.
- After the pass, the one slot still **positive** was never visited → its index + 1 is the **missing** number.

```java
class Solution {
    public int[] findErrorNums(int[] nums) {
        int dup = -1, missing = -1;

        for (int i = 0; i < nums.length; i++) {
            int idx = Math.abs(nums[i]) - 1;   // abs: nums[i] may already be flipped
            if (nums[idx] < 0) {
                dup = Math.abs(nums[i]);
            } else {
                nums[idx] = -nums[idx];
            }
        }

        for (int i = 0; i < nums.length; i++) {
            if (nums[i] > 0) {
                missing = i + 1;
                break;
            }
        }

        return new int[]{dup, missing};
    }
}
```

**Time:** O(n)
**Space:** O(1) extra (output not counted)

**The one line that trips everyone up:** `Math.abs(nums[i])`. By the time you reach later elements, earlier iterations have already flipped some of them. Reading `nums[i]` raw would give a negative index → `ArrayIndexOutOfBoundsException`.

**Caveats to say out loud in an interview:**
1. It **mutates the input**. Ask if that's acceptable. (Restore with one pass of `nums[i] = Math.abs(nums[i])` if needed.)
2. Only works because values are in `[1, n]`. Zero has no sign, so a 0 in the data breaks it.

---

## Approach 4 — Math (sum + sum of squares)

Let `d` = duplicate, `m` = missing.

```
S  = sum(nums)  − n(n+1)/2        = d − m
Sq = sum(nums²) − n(n+1)(2n+1)/6  = d² − m² = (d+m)(d−m)
```

So `d + m = Sq / S`, and combined with `d − m = S`:

```
d = (S + Sq/S) / 2
m = d − S
```

**Time:** O(n) · **Space:** O(1) · **Doesn't mutate the input**

⚠️ `sum of squares` for n = 10⁴ is ~3.3×10¹¹ → **overflows `int`. Use `long`.** This is the reason this approach is rarely worth it: clever, but overflow-fragile and harder to explain than Approach 3.

---

## Dry run

Input: `nums = [1, 2, 2, 4]`, `n = 4`. Expected output: `[2, 3]`.

### Approach 2 (my solution)

`set` starts as `[0, 0, 0, 0, 0]` (indices 0–4).

| step | num | `set[num]` before | `set[num] == num`? | action | `set` after |
|---|---|---|---|---|---|
| 1 | 1 | 0 | 0 == 1 → no | write | `[0,1,0,0,0]` |
| 2 | 2 | 0 | 0 == 2 → no | write | `[0,1,2,0,0]` |
| 3 | 2 | 2 | 2 == 2 → **yes** | `ans[0] = 2` | `[0,1,2,0,0]` |
| 4 | 4 | 0 | 0 == 4 → no | write | `[0,1,2,0,4]` |

Second loop over `i = 1..4`:

| i | `set[i]` | `set[i] != i`? | action |
|---|---|---|---|
| 1 | 1 | no | continue |
| 2 | 2 | no | continue |
| 3 | **0** | **yes** | `ans[1] = 3`, break |

→ `[2, 3]` ✅

### Approach 3 (sign marking)

| i | `nums[i]` | `abs` | `idx = abs−1` | `nums[idx]` | action | array after |
|---|---|---|---|---|---|---|
| 0 | 1 | 1 | 0 | 1 (+) | flip | `[-1, 2, 2, 4]` |
| 1 | 2 | 2 | 1 | 2 (+) | flip | `[-1,-2, 2, 4]` |
| 2 | 2 | 2 | 1 | −2 (−) | **already negative → `dup = 2`** | `[-1,-2, 2, 4]` |
| 3 | 4 | 4 | 3 | 4 (+) | flip | `[-1,-2, 2,-4]` |

Scan for the positive: index 2 holds `2` (positive) → `missing = 2 + 1 = 3`.

→ `[2, 3]` ✅

Note how index 2 is the *only* one never targeted — because the value 3 (which would have pointed there) is exactly what went missing.

---

## Comparison

| Approach | Time | Space | Mutates input? | Beats | Notes |
|---|---|---|---|---|---|
| 1. HashSet | O(n) | O(n) | no | ~38% | Boxing + hashing overhead |
| 2. **Count array** | O(n) | O(n) | no | **~94%** | **My pick — clean, fast, obviously correct** |
| 3. Sign marking | O(n) | O(1) | **yes** | ~100% | The "can you do better?" answer |
| 4. Sum + sum² | O(n) | O(1) | no | ~100% | Needs `long`; overflow-fragile |
| — Sorting | O(n log n) | O(1) | yes | — | Strictly worse; mentioned only for completeness |

**Interview strategy:** lead with Approach 2 (fast to write, no bugs), state that it's O(n) space, then say *"if you want O(1) extra space and I'm allowed to mutate the input, there's a sign-marking trick."* Write it only if they ask. Getting a correct answer down fast beats fumbling a clever one.

---

## The transferable pattern 🔑

**When values are constrained to `[1, n]` and the array has length `n`, the array can be its own hash table.**

Two flavours of marker:
- **Separate array** (`int[] count` / `boolean[] seen`) — O(n) space, simple, safe.
- **Sign flip in place** — O(1) space, mutates input, only valid when 0 ∉ domain.

Family of problems using exactly this:

| Problem | Ask | Marker used |
|---|---|---|
| **LC 645** Set Mismatch | dup + missing | sign flip |
| **LC 448** Find All Numbers Disappeared | all missing | sign flip |
| **LC 442** Find All Duplicates | all duplicates | sign flip |
| **LC 41** First Missing Positive | first absent positive | **cyclic sort** (values unbounded → sign trick fails) |
| **LC 268** Missing Number | one missing | XOR or sum formula |
| **LC 287** Find the Duplicate | dup, no mutation allowed | **Floyd's cycle detection** |

> LC 41 and LC 287 are the two that *look* like this family but need a different tool. 41 because values aren't bounded, 287 because mutation is forbidden. Worth doing both next.

---

## Takeaways

1. **O(n) with a HashSet ≠ O(n) with an int array.** Same complexity class, very different constant. Boxing + hashing + cache misses are real costs. When keys are small bounded ints, reach for the array.
2. **Read the constraints as hints.** `1 <= nums[i] <= n` isn't boilerplate — it's the setup for index-as-hash.
3. **Using a value as its own "seen" flag** works only when the default (0) is outside the value domain. Know why, so you know when it breaks.
4. **Sign as a marker** is lossless — `abs()` recovers the value. That's why you can store two bits of information (value + visited) in one slot.
5. **Don't over-read LeetCode percentiles** on small problems. 38 → 94 was a genuine improvement, but the number bounces a few points run to run.
6. **In an interview, name the trade-off.** "O(1) space but destroys the input" is the honest framing, and saying it earns more than silently mutating.