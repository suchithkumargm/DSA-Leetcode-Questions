# 15. 3Sum

**Difficulty:** Medium
**Topics:** Array, Two Pointers, Sorting, Hashing
**Link:** https://leetcode.com/problems/3sum/

---

## Problem

Given an integer array `nums`, return all **unique** triplets `[nums[i], nums[j], nums[k]]`
such that `i != j != k` and `nums[i] + nums[j] + nums[k] == 0`.

Two constraints do all the damage:

1. **All** triplets, not just one — so you can't return early.
2. **Unique** triplets — same values in a different index order count as one answer.

---

## First instinct — reuse the Two Sum hashmap (doesn't work)

The natural move is: fix one number, then run Two Sum on the rest looking for `-num`.

```java
HashMap<Integer, Integer> map = new HashMap<>();
for(int i=0; i<nums.length; i++) map.put(nums[i], i);   // value -> index
```

This falls apart for four separate reasons, all traceable to the two constraints above.

| # | Bug | Why |
|---|-----|-----|
| 1 | Map collapses duplicates | `put` overwrites, so `[-1,0,1,2,-1,-4]` keeps only the *last* `-1`. But 3Sum answers often need **two** copies of a value. |
| 2 | Guarding with `eleToFind != nums[i]` | Added to stop reusing one element — but it also blocks legitimately using two *different* elements holding the same value. `[-1,-1,2]` is missed entirely. |
| 3 | Outer element can be reused | The inner scan covers the whole array including the outer index. On `[1,-2,5]` it reports `1, 1, -2` using index 0 twice. |
| 4 | Only one pair per outer element | Returning on the first hit misses other triplets that share the same first number. |

**The lesson:** Two Sum guarantees exactly one answer and never cares about duplicates.
3Sum wants *all* answers and forbids duplicates. The hashmap idiom doesn't lift across
that gap without heavy patching.

---

## Approach 1 — Brute force (O(n^3))

Three nested loops with strictly increasing indices.

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Set<List<Integer>> answer = new HashSet<>();
        for(int i=0; i<nums.length; i++){
            for(int j=i+1; j<nums.length; j++){
                for(int k=j+1; k<nums.length; k++){
                    if(nums[i] + nums[j] + nums[k] == 0){
                        List<Integer> list = new ArrayList<>();
                        list.add(nums[i]);
                        list.add(nums[j]);
                        list.add(nums[k]);
                        Collections.sort(list);      // canonical order for dedupe
                        answer.add(list);
                    }
                }
            }
        }
        return new ArrayList<>(answer);
    }
}
```

### Key details

- **`j = i+1`, `k = j+1`** is what guarantees three *distinct indices*. No extra checks
  needed — indices are strictly increasing by construction. This is exactly what the
  hashmap version couldn't enforce.
- **The sort before `answer.add(list)` is mandatory.** `List.equals` is order-sensitive,
  so `[-1,0,1]` and `[0,1,-1]` hash differently and the `HashSet` would store both.
  Every copy of a triplet must be built in the same order for the set to dedupe.
- Better alternative: `Arrays.sort(nums)` once at the top. Then `i < j < k` means you're
  *already* reading ascending, so triplets are canonical for free — no per-triplet sort
  inside the innermost loop.

### Complexity

| | |
|---|---|
| Time | `O(n^3)` |
| Space | `O(k)` for the set, k = number of triplets |

**Verdict: TLE.** With `n = 3000`, n^3 is ~2.7 billion iterations.

---

## Approach 2 — Better: hashset window (O(n^2))

Fix `i`, walk `j` forward, and keep a set of everything strictly between them.

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Set<List<Integer>> st = new HashSet<>();
        int n = nums.length;

        for (int i = 0; i < n; i++) {
            Set<Integer> hashset = new HashSet<>();
            for (int j = i + 1; j < n; j++) {
                int third = -(nums[i] + nums[j]);
                if (hashset.contains(third)) {
                    List<Integer> temp = new ArrayList<>(List.of(nums[i], nums[j], third));
                    Collections.sort(temp);
                    st.add(temp);
                }
                hashset.add(nums[j]);      // insert AFTER the lookup
            }
        }
        return new ArrayList<>(st);
    }
}
```

### The one trick worth remembering

`hashset.add(nums[j])` happens **after** the lookup. That means at any moment the set
holds exactly the elements strictly *between* `i` and `j`:

```
    i           <-- hashset holds this window -->        j
  [ . ]  [ x ]  [ x ]  [ x ]  [ x ]  [ x ]  [ x ]  [ . ]
```

So the third element is guaranteed to sit between them, and **all three indices are
distinct by construction**. No index comparisons, no `k != i && k != j` checks.

This is the same principle as the two-pointer version: *restrict the search space so
distinctness falls out of the structure* rather than checking for it after the fact.

### Java gotcha

`List.of(...)` returns an **immutable** list — `Collections.sort` on it throws
`UnsupportedOperationException`. Wrap it: `new ArrayList<>(List.of(...))`.

### Complexity

| | |
|---|---|
| Time | `O(n^2)` |
| Space | `O(n)` inner set + `O(k)` outer set |

Passes, but carries `O(n)` extra space and pays hashing costs on every lookup and insert.

---

## Approach 3 — Optimal: sort + two pointers (O(n^2), O(1) space)

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {

        Arrays.sort(nums);

        List<List<Integer>> ans = new ArrayList<>();

        for(int left=0; left<nums.length-2; left++){
            if(left > 0 && nums[left] == nums[left-1]){
                continue;                             // skip duplicate first element
            }

            int mid = left + 1, right = nums.length - 1;

            while(mid < right){
                int sum = nums[left] + nums[mid] + nums[right];

                if(sum == 0){
                    ans.add(new ArrayList<>(List.of(nums[left], nums[mid], nums[right])));
                    while(mid < right && nums[mid] == nums[mid+1]){
                        mid++;
                    }
                    while(mid < right && nums[right] == nums[right-1]){
                        right--;
                    }
                    mid++;
                    right--;
                } else if(sum < 0){
                    mid++;                            // need a bigger number
                } else{
                    right--;                          // need a smaller number
                }
            }
        }

        return ans;
    }
}
```

### Why sorting unlocks everything

Once sorted, "bigger numbers are to the right" becomes a fact you can steer by:

```
sum < 0  ->  too small  ->  mid++     (move toward larger values)
sum > 0  ->  too large  ->  right--   (move toward smaller values)
sum == 0 ->  record, then move BOTH
```

Moving only one pointer after a hit is pointless — the other still holds the same value,
and no other partner can produce zero with it. Both must move.

Sorting also puts equal values adjacent, which makes duplicate-skipping an `O(1)`
neighbour comparison instead of a set lookup.

### `left` is a loop variable; `mid` and `right` are pointers

This is the distinction that causes the most confusion. `right` **resets to `n-1` on
every iteration of `left`**, and that is correct — not a bug.

When `left` advances, the target changes, so it's a completely new subproblem: find two
numbers in `nums[left+1 .. n-1]` summing to `-nums[left]`. A large value that was too big
for the previous target may be exactly right for this one.

```
nums = [-4, -1, -1,  0,  1,  2]        (sorted)

left = 0  (-4), target 4
        L    M--------------->R        finds nothing (max pair 1+2 = 3)

left = 1  (-1), target 1
             L    M---------->R        finds [-1, -1, 2]  <-- needs index 5 back!
```

If `right` carried over from the previous pass, index 5 would be gone and `[-1,-1,2]`
would be missed. Loop variables advance monotonically; pointers are recreated fresh for
each scan.

No asymptotic cost — each inner scan is `O(n)` regardless of where `right` starts.

### Duplicate handling — two skips, different directions

**Outer skip** compares **backwards**:
```java
if(left > 0 && nums[left] == nums[left-1]) continue;
```
This means "I already processed this value as a starting element." Take the *first*
occurrence, skip the rest.

Comparing **forwards** (`nums[left] == nums[left+1]`) is wrong twice over: it reads out
of bounds at the last index, and on `[-1,-1,0,1,2]` it skips `left=0` — the only position
that can find `[-1,-1,2]`.

**Inner skips** run only after a hit, and must be bounds-guarded (`mid < right`) *before*
the array access. They park the pointers on the last duplicate; the following `mid++` /
`right--` steps past it.

Note there's no skip in the `sum < 0` / `sum > 0` branches — it'd be redundant. Skipping
only matters after recording an answer.

### Complexity

| | |
|---|---|
| Time | `O(n^2)` — sort is `O(n log n)`, dominated |
| Space | `O(1)` extra, ignoring the output list |

---

## Comparison

| Approach | Time | Space | Notes |
|---|---|---|---|
| Brute force | `O(n^3)` | `O(k)` set | TLE at n=3000 |
| Hashset window | `O(n^2)` | `O(n)` + set | Passes; hashing overhead |
| Sort + two pointers | `O(n^2)` | `O(1)` | Optimal; the expected interview answer |

---

## Mistakes made along the way

1. **Forgot `Arrays.sort(nums)`** before two-pointering. Every pointer decision is a
   guess without it. Also forced an ugly `nums.length == 3` special case that vanished
   once the sort was added.
2. **Three pointers moving in one loop.** Had `left`, `mid`, `right` all advancing inside
   a single `while`. That can't enumerate all triplets — `left` never got past index 0 on
   `[-4,-1,-1,0,1,2]`. The fix is an outer `for` over `left`, with a nested `while` that
   fully explores each one.
3. **Dropped the inner `while` entirely** when restructuring. Computed one sum per `left`,
   took one branch, then the `for` reassigned the pointers and threw the work away.
4. **`sum` computed outside the loop.** It has to recompute every time a pointer moves.
5. **Duplicate skip comparing forwards instead of backwards** — crashes at the last index
   and silently loses valid triplets.
6. **Unguarded skip loops** — `nums[left+1]` and `nums[right-1]` go out of bounds at the
   array edges. Bounds check must come *before* the comparison in the `&&`.

---

## Takeaways

- **Restrict the search space instead of validating after the fact.** Both the `O(n^2)`
  solutions get index-distinctness for free from their structure. The hashmap version had
  to check for it and got it subtly wrong.
- **Sorting is often the enabling move**, not the goal. Here it buys three things at once:
  directional pointer logic, adjacent duplicates, and canonical triplet ordering.
- **Loop variable vs pointer** is a real distinction. Loop variables march forward once;
  pointers are born and die within a single scan.
- `List.of()` is immutable — wrap it in `new ArrayList<>()` before sorting or mutating.

---

## Follow-ups

- **LC 18 (4Sum)** — this exact pattern with one more loop wrapped around it. Watch for
  integer overflow on the sum; use `long`.
- **LC 16 (3Sum Closest)** — same skeleton, track the minimum `|sum|` instead of `== 0`.
- **LC 259 (3Sum Smaller)** — same skeleton, but on `sum < target` you add `right - mid`
  to the count in one shot instead of stepping.
- **LC 167 (Two Sum II)** — the two-pointer core in isolation, on a pre-sorted array.

The `Arrays.sort` + two pointers idiom recurs across all of these. Worth having in muscle
memory.