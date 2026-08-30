# 18. 4Sum

**Difficulty:** Medium
**Topics:** Array, Two Pointers, Sorting, Hashing
**Link:** https://leetcode.com/problems/4sum/

---

## Problem

Given an integer array `nums` and an integer `target`, return all **unique** quadruplets
`[nums[a], nums[b], nums[c], nums[d]]` with distinct indices such that they sum to `target`.

Structurally identical to 3Sum with one more element. The new difficulty is **integer
overflow** — 4Sum's constraints allow values up to `10^9`, wide enough that four of them
blow past `int` range.

---

## Approach 1 — Brute force (O(n^4))

Four nested loops with strictly increasing indices.

```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        Set<List<Integer>> answer = new HashSet<>();
        for(int i=0; i<nums.length; i++){
            for(int j=i+1; j<nums.length; j++){
                for(int k=j+1; k<nums.length; k++){
                    for(int l=k+1; l<nums.length; l++){
                        long sum = (long) nums[i] + nums[j] + nums[k] + nums[l];
                        if(sum == target){
                            List<Integer> list = new ArrayList<>();
                            list.add(nums[i]);
                            list.add(nums[j]);
                            list.add(nums[k]);
                            list.add(nums[l]);
                            Collections.sort(list);      // canonical order for dedupe
                            answer.add(list);
                        }
                    }
                }
            }
        }
        return new ArrayList<>(answer);
    }
}
```

- `j=i+1`, `k=j+1`, `l=k+1` guarantees four distinct indices by construction.
- `Collections.sort(list)` is what makes the `HashSet` dedupe work — `List.equals` is
  order-sensitive, so every copy of a quadruplet must be built in the same order.
- The `(long)` cast is mandatory. See the overflow section below.

| | |
|---|---|
| Time | `O(n^4)` |
| Auxiliary space | `O(1)` (excluding the set/output) |

**Verdict: TLE.**

---

## Approach 2 — Better: hashset window (O(n^3))

Fix `i` and `j`, walk `k` forward, keep a set of everything strictly between `j` and `k`.

```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        Set<List<Integer>> answer = new HashSet<>();
        for(int i=0; i<nums.length; i++){
            for(int j=i+1; j<nums.length; j++){
                HashSet<Long> set = new HashSet<>();          // one level ABOVE the k loop
                for(int k=j+1; k<nums.length; k++){
                    long fourth = (long) target - nums[i] - nums[j] - nums[k];
                    if(set.contains(fourth)){
                        List<Integer> list = new ArrayList<>();
                        list.add(nums[i]);
                        list.add(nums[j]);
                        list.add(nums[k]);
                        list.add((int) fourth);
                        Collections.sort(list);
                        answer.add(list);
                    }
                    set.add((long) nums[k]);                  // insert AFTER the lookup
                }
            }
        }
        return new ArrayList<>(answer);
    }
}
```

### Where the set must live

The set has to be declared **one level above the loop that scans**. Here `k` scans, so the
set belongs in the `j` loop.

Declaring it inside the `k` loop is a silent killer — a fresh empty set every iteration
means `contains` is always false, `add` writes to something discarded a line later, and
the method returns an empty list with no error.

### Insert after lookup

`set.add(nums[k])` runs after the `contains` check, so at any moment the set holds exactly
the elements strictly between `j` and `k`:

```
    i     j        <-- set holds this window -->        k
  [ . ] [ . ]  [ x ]  [ x ]  [ x ]  [ x ]  [ x ]  [ . ]
```

The fourth element is guaranteed to sit in that window, so all four indices are distinct
**by construction** — no index comparisons needed.

### `Set<Long>`, not `Set<Integer>`

`fourth` must be a `long` to avoid overflow. But `Set<Integer>.contains(someLong)` is
**always false** — `Integer.equals(Long)` returns false regardless of value, and it
compiles fine because `contains` takes `Object`. Make the set `Set<Long>` and box the
inserts with `(long) nums[k]`.

The `(int) fourth` cast on the way out is safe: it only runs when `fourth` matched
something already in the set, and everything in the set came from `nums`.

| | |
|---|---|
| Time | `O(n^3)` |
| Auxiliary space | `O(n)` |

### Why space is O(n), not O(n^2)

The set is *created* `n^2` times (once per `(i,j)` pair), but only **one is alive at any
moment** — reassigning `set` makes the previous one unreachable and it gets collected.

> Space complexity measures **peak live footprint**, not cumulative allocation.

Passes, but slow in practice: every set operation boxes a primitive and hashes an object,
plus a `Collections.sort` and a list hash on every hit.

---

## Approach 3 — Optimal: sort + two pointers (O(n^3), O(1) space)

```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        Arrays.sort(nums);
        List<List<Integer>> answer = new ArrayList<>();

        for(int i=0; i<nums.length; i++){
            if(i != 0 && nums[i] == nums[i-1]){
                continue;
            }
            for(int j=i+1; j<nums.length; j++){
                if(j != i + 1 && nums[j] == nums[j-1]){
                    continue;
                }

                int k = j + 1, l = nums.length - 1;

                while(k < l){
                    long sum = (long) nums[i] + nums[j] + nums[k] + nums[l];

                    if(sum == target){
                        answer.add(new ArrayList<>(
                            List.of(nums[i], nums[j], nums[k], nums[l])));
                        k++;
                        l--;
                        while(k < l && nums[k] == nums[k-1]){
                            k++;
                        }
                        while(k < l && nums[l] == nums[l+1]){
                            l--;
                        }
                    } else if(sum < target){
                        k++;                     // need a bigger number
                    } else{
                        l--;                     // need a smaller number
                    }
                }
            }
        }

        return answer;
    }
}
```

### The `j != i+1` skip

This is the one genuinely new idea over 3Sum. The two outer skips look similar but ask
different questions:

```java
if(i != 0     && nums[i] == nums[i-1]) continue;   // first i overall?
if(j != i + 1 && nums[j] == nums[j-1]) continue;   // first j for THIS i?
```

`j` restarts at `i+1` for every `i`, so the guard has to be `j != i+1`, not `j != 0`.
Using `j != 0` would skip the legitimate first `j` of each new `i` whenever it happens to
equal the element before it — silently losing quadruplets.

### Pointer logic

Sorting makes "bigger values are to the right" a fact you can steer by:

```
sum < target  ->  too small  ->  k++     (move toward larger values)
sum > target  ->  too large  ->  l--     (move toward smaller values)
sum == target ->  record, then move BOTH
```

Moving only one pointer after a hit is pointless — the other still holds the same value,
and no other partner can hit the target with it.

`k` and `l` reset for every `(i,j)` pair. That's correct, not a bug: when `i` or `j`
changes, the remaining target changes, so it's a new subproblem over the full range to the
right of `j`. A large value that was too big before may be exactly right now.

### No `HashSet` needed

The four duplicate guards (two outer `continue`s, two inner `while`s) make it impossible
to produce the same quadruplet twice. That's their entire job. Using a plain `ArrayList`
drops the boxing and list-hashing cost and makes auxiliary space genuinely `O(1)`.

### Skip form

This version advances first, then skips:

```java
k++; l--;
while(k < l && nums[k] == nums[k-1]) k++;
while(k < l && nums[l] == nums[l+1]) l--;
```

The alternative — skip to the last duplicate, then advance — is equivalent. Both work;
just be consistent. Bounds guard (`k < l`) must come **before** the array access in the
`&&`, or the edges go out of range.

| | |
|---|---|
| Time | `O(n^3)` — sort is `O(n log n)`, dominated |
| Auxiliary space | `O(1)` excluding output |

---

## Integer overflow — the 4Sum-specific trap

`nums[i]` can be up to `10^9`. Four of them reach `4 * 10^9`; `int` caps at ~`2.1 * 10^9`.
The sum wraps to a negative number and the comparison silently produces wrong answers.

**Casting the result does not help — you must cast an operand.**

```java
long sum = nums[i] + nums[j] + nums[k] + nums[l];          // WRONG
// entire sum computed in int, overflows, THEN widens

long sum = (long) nums[i] + nums[j] + nums[k] + nums[l];   // CORRECT
// first operand is long, so the whole chain promotes
```

Same trap in the "better" solution:

```java
long fourth = target - (nums[i] + nums[j] + nums[k]);      // WRONG — inner sum is int
long fourth = (long) target - nums[i] - nums[j] - nums[k]; // CORRECT
```

One cast on the leading operand is enough; Java promotes everything downstream.

---

## Comparison

| Approach | Time | Auxiliary space | Notes |
|---|---|---|---|
| Brute force | `O(n^4)` | `O(1)` | TLE |
| Hashset window | `O(n^3)` | `O(n)` | Passes; heavy boxing/hashing constants |
| Sort + two pointers | `O(n^3)` | `O(1)` | Optimal |

---

## Mistakes made along the way

1. **No overflow handling** in the first brute force. The `int` sum wraps at `10^9`-scale
   inputs.
2. **Set declared inside the `k` loop** in the better solution — recreated empty every
   iteration, so `contains` never fired and the method returned nothing. Fails silently.
3. **`Set<Integer>.contains(long)`** compiles but is always false. Needed `Set<Long>`.
4. **Casting the result instead of an operand** — `long fourth = target - (int sum)`
   overflows before the widening happens.
5. **Kept the `HashSet` in the optimal version**, which makes the duplicate skips
   redundant work and breaks the `O(1)` space claim.

---

## Takeaways

- **4Sum = 3Sum + one loop.** The skeleton transfers directly; only the `j != i+1` skip
  and the overflow handling are new.
- **Cast an operand, not the result.** `(long) a + b`, never `long x = a + b`.
- **Boxed generics don't cross-compare.** `Integer.equals(Long)` is false even for equal
  values, and it compiles without warning because `contains` takes `Object`.
- **A helper collection belongs one level above the loop that scans it.**
- **Space complexity is peak live footprint**, not how many times something was allocated.