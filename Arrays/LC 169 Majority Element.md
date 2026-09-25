# LC 169 — Majority Element

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450 | Apna College
**Pattern:** Boyer-Moore voting / cancellation

---

## 1. Problem

Given an array `nums` of size `n`, return the majority element — the one appearing **more than
⌊n/2⌋ times**.

**The problem guarantees a majority element always exists.** Remember this: three of the four
approaches below are wrong without it.

```
[3,2,3]          → 3
[2,2,1,1,1,2,2]  → 2
```

**Follow-up:** O(n) time and O(1) space.

---

## 2. Why "more than n/2" is the whole problem

Every approach exploits one fact:

> **An element occupying more than half the array cannot be outvoted, out-positioned, or out-counted
> by everything else combined — because everything else combined is strictly less than half.**

- Sorting exploits it **positionally** (it must cover the middle index).
- Boyer-Moore exploits it by **cancellation** (it can't be fully paired off).
- Bit counting exploits it **per bit** (each of its set bits is set in >n/2 numbers).

Remember only that sentence and all three are re-derivable.

---

## 3. Approach 1 — HashMap counting

Count everything, return whatever crosses the threshold. The 3-minute answer — state it, then
immediately say *"but I can do better on space."*

```java
class Solution {
    public int majorityElement(int[] nums) {
        HashMap<Integer, Integer> map = new HashMap<>();

        for (int num : nums) {
            map.put(num, map.getOrDefault(num, 0) + 1);
            if (map.get(num) > nums.length / 2) return num;   // early exit
        }

        return -1;   // unreachable given the problem's guarantee
    }
}
```

### ⚠️ Two things to get right

**Detect the winner during the counting pass.** A separate second loop is wasted work — the moment
any count crosses n/2 you have the answer. And if you *do* use two passes, iterate
**`map.entrySet()`**, not `nums`: iterating the array re-checks every duplicate, so on
`[2,2,2,2,2,1]` you check `2` five times.

**The final `return` is unreachable dead code.** It exists only because Java demands a return on
every path. Know that; an interviewer will ask why it's there.

### Java detail — one hash lookup instead of two

`map.put(...)` then `map.get(num)` hashes the key **twice**. `merge` returns the new value directly:

```java
int count = map.merge(num, 1, Integer::sum);
if (count > nums.length / 2) return num;
```

Same complexity, half the hash operations, and it reads better. Shows up constantly in
frequency-counting problems.

### Complexity

| | | |
|---|---|---|
| **Time** | O(n) | one pass; HashMap ops are **amortised** O(1) |
| **Space** | O(n) | worst case one entry per distinct element |

> **Say "amortised."** HashMap is O(1) *amortised*, not worst-case. Java 8+ converts long buckets to
> red-black trees, so the true worst case per operation is O(log n), not O(n). Free credibility
> point.

---

## 4. Approach 2 — Sorting

### The obvious version

Sort, then one pass counting runs, return the run longer than n/2. **O(n log n)**.

### The sharp version ⭐

> **After sorting, the majority element is at index `n/2`. Always.**

```java
Arrays.sort(nums);
return nums[nums.length / 2];
```

**Why:** the majority element occupies more than half the array, and after sorting its copies are
contiguous. A contiguous block longer than n/2 **cannot avoid** straddling the middle index — there
isn't room on either side of the midpoint to fit it entirely.

Sanity check, `n = 7`, needs ≥4 copies, middle index 3:
- pushed as far **left** as possible: indices `0,1,2,3` → covers 3 ✅
- pushed as far **right** as possible: indices `3,4,5,6` → covers 3 ✅

### Complexity

| | | |
|---|---|---|
| **Time** | O(n log n) | the sort |
| **Space** | **O(log n)** | ⚠️ Java's dual-pivot quicksort on primitives — recursion stack. **Not O(1).** Don't claim it is. |

Strictly worse than both O(n) approaches, but worth *offering* — it shows you see structure, not
just counters.

---

## 5. Approach 3 — Boyer-Moore Voting ⭐ the intended answer

```java
class Solution {
    public int majorityElement(int[] nums) {
        int candidate = nums[0];
        int count = 1;

        for (int i = 1; i < nums.length; i++) {
            if (nums[i] == candidate) {
                count++;
            } else {
                count--;
                if (count == 0) {          // only reachable after a decrement
                    candidate = nums[i];
                    count = 1;
                }
            }
        }
        return candidate;
    }
}
```

### Two design choices worth justifying

**The zero-check lives inside the `else`.** `count` can only reach 0 via a decrement, so testing
after an increment is wasted work. Give that reasoning out loud rather than discovering it by
testing.

**Name it `candidate`, not `majorityElement`.** Mid-loop the variable may hold something discarded
three iterations later — calling it "the majority element" is a claim it can't back up yet. The
honest name also reminds you a **verification pass** exists for the variants where the guarantee is
removed (§7).

### The standard seeding form

```java
int candidate = 0, count = 0;
for (int num : nums) {
    if (count == 0) candidate = num;
    count += (num == candidate) ? 1 : -1;
}
```

Both are correct. Learn this one too, because it's what every editorial shows, and because it
**generalises cleanly to LC 229** (majority > n/3), where you carry two candidates and two counters
and the zero-check placement matters.

### 🔑 Correctness — the pairing argument

> ⚠️ **"The count never goes below 0" proves nothing.** Every counter that resets at zero has that
> property. It says nothing about *which* element survives.

The real argument:

> Each decrement **pairs** one occurrence of the current candidate against one occurrence of some
> other element, and cancels **both**.
>
> The majority element has **more than n/2** copies, so everything else combined has **fewer than
> n/2**. Every cancellation destroys at most one copy of the majority element, and there are
> strictly fewer non-majority elements available to do the destroying than there are copies to
> destroy.
>
> Therefore the majority element cannot be fully cancelled. Something survives — and only the
> majority element can be what survives.

**Test whether you actually have it:** the proof must explain why the answer is *the majority element
specifically*, not merely why the algorithm terminates or why a counter stays non-negative. If your
sentence would be equally true of a non-majority input, it isn't a proof.

### What it depends on

**The guarantee that a majority element exists.** Remove it and Boyer-Moore returns *a* number with
no meaning — `[1,2,3]` yields `3`, a majority of nothing. See §7.

**Time O(n) · Space O(1).** This is the follow-up's answer.

---

## 6. Approach 4 — Bit counting

### Idea

Build the answer **one bit at a time** instead of finding the element as a whole.

Pick any bit position `i`. If the majority element has bit `i` set, then more than n/2 of the numbers
have bit `i` set — because the majority element *alone* accounts for more than n/2 of them. If its
bit `i` is clear, fewer than n/2 numbers can have it set.

> **Bit `i` of the answer is 1 ⟺ more than n/2 array elements have bit `i` set.**

```java
public int majorityElement(int[] nums) {
    int result = 0;
    for (int i = 0; i < 32; i++) {
        int bitCount = 0;
        for (int num : nums) {
            if ((num >> i & 1) == 1) bitCount++;
        }
        if (bitCount > nums.length / 2) result |= (1 << i);
    }
    return result;
}
```

⚠️ **The sign bit (i = 31).** `1 << 31` is `Integer.MIN_VALUE`, and `result |= (1 << 31)` correctly
sets it — two's complement works out. Be deliberate: use `>>` or `>>>` consistently and test with a
negative input. LC 169's constraints allow negatives.

**Time O(32n) = O(n) · Space O(1).** ~32× slower than Boyer-Moore in practice.

**Why learn it if it's strictly worse?** Because the *technique* — solving bitwise for the answer
rather than searching for it — is the core idea in **LC 137 (Single Number II)** and **LC 260 (Single
Number III)**, where it isn't optional.

---

## 7. Follow-ups you will actually be asked

### "What if a majority element is NOT guaranteed?"

Boyer-Moore's candidate becomes meaningless. Add a **verification pass**:

```java
int count = 0;
for (int num : nums) if (num == candidate) count++;
return count > nums.length / 2 ? candidate : -1;
```

Still O(n) time, O(1) space — two passes instead of one. **The most common follow-up on this
problem.** Have it ready.

Which approaches survive without the guarantee:

| approach | survives? |
|---|---|
| HashMap | ✅ the threshold check is explicit |
| Sorting → `nums[n/2]` | ❌ needs verification |
| Boyer-Moore | ❌ needs verification |
| Bit counting | ❌ needs verification |

### "Find all elements appearing more than ⌊n/3⌋ times." → LC 229

Generalised Boyer-Moore: at most **two** such elements can exist (three would need >n, a
contradiction). Carry two candidates and two counters, then **verify both** — verification is
mandatory here because n/3-majority elements aren't guaranteed to exist.

The `count = 0` seeding form (§5) is the one that generalises. Do this problem next.

### "What about a stream, one pass, can't store the array?"

Boyer-Moore works — O(1) space, single pass, which is why it exists. You can't run the verification
pass, so you need the existence guarantee.

---

## 8. Approach comparison

| # | Approach | Time | Space | Notes |
|---|---|---|---|---|
| 1 | HashMap counting | O(n) | **O(n)** | 3-minute answer; state it, then improve it |
| 2 | Sort + count runs | O(n log n) | O(log n) | strictly dominated |
| 2b | Sort → `nums[n/2]` | O(n log n) | O(log n) | one line; good "I see structure" answer |
| 4 | Bit counting | O(32n) | **O(1)** | worse constant; technique matters for LC 137/260 |
| **3** | **Boyer-Moore** | **O(n)** | **O(1)** | ⭐ **the answer** — single pass, two ints |

There's also randomised sampling (pick a random index, verify, repeat — O(n) expected). Mention it
only if asked for exotica.

---

## 9. Edge cases

| case | expected | Boyer-Moore |
|---|---|---|
| `[1]` | 1 | ✅ loop never runs, returns `nums[0]` |
| `[2,2]` | 2 | ✅ |
| all identical | that element | ✅ count only ever increments |
| majority at the very end, `[1,2,3,3,3,3,3]` | 3 | ✅ candidate flips until 3 takes over |
| majority split, `[3,1,3,2,3]` | 3 | ✅ cancellations don't exhaust it |
| negative numbers | works | ✅ (only the *bitwise* approach needs sign-bit care) |
| `n` even, exactly n/2 copies | **not a majority** | needs `>`, never `>=` |

> ⚠️ The threshold is **strictly greater than** ⌊n/2⌋. For `n = 4`, three copies qualify and two do
> not.

---

## 10. Interview Q&A

**Q: Prove Boyer-Moore is correct.**
The pairing argument, §5. Each decrement cancels one candidate against one non-candidate; there
aren't enough non-majority elements to cancel a strict majority.

**Q: What if there's no majority element?**
The candidate is meaningless — add a verification pass counting its actual occurrences.

**Q: Why does `nums[n/2]` work after sorting?**
A contiguous block longer than half the array must straddle the middle index.

**Q: Can you do it without the existence guarantee *and* in one pass?**
No — you fundamentally need to see the whole array before you can confirm a count exceeds n/2.

**Q: Why is the sort-based solution not O(1) space?**
Java sorts primitives with dual-pivot quicksort, which recurses — O(log n) stack.

---

## 11. Pattern

> **Pair each occurrence of the candidate against one non-candidate and cancel both — an element
> with more than n/2 copies cannot be fully cancelled, so whatever survives is the answer.**

This sentence reconstructs the code. *"Increment if same, decrement if not, reset at zero"* does not
— it's the code, restated.

---

## 12. Takeaways

1. **"More than n/2" is a structural guarantee, not just a counting fact.** Ask what a threshold
   *forces* before reaching for a map.
2. **Boyer-Moore's proof is the pairing argument.** "The count never goes negative" proves nothing.
3. **Boyer-Moore needs the existence guarantee.** Without it, add a verification pass.
4. **Name it `candidate`, not `majorityElement`.**
5. **`map.merge(k, 1, Integer::sum)`** returns the new count — one hash lookup instead of two.
6. **Iterate the map, not the array**, when looking for a winner among counts.
7. **Sorting in Java is O(log n) space, not O(1).** Don't overclaim.
8. **Solving bitwise for the answer** is a general technique → LC 137, 260.
9. **Say "amortised" about HashMap.** Java 8+ worst case per op is O(log n), not O(n).
10. **Survey before typing:** brute force → what resource to reduce → all approaches with costs →
    pick one and justify it. Ninety seconds, and it's scored.

---

## 13. Related problems

| # | Problem | Shared idea |
|---|---|---|
| **229** | Majority Element II (> n/3) | ⭐ generalised Boyer-Moore, two candidates, **verification mandatory** |
| 1150 | Check If a Number Is Majority Element | binary search on a sorted array for the same threshold |
| 137 | Single Number II | solve bitwise for the answer (§6's technique, where it's required) |
| 260 | Single Number III | bit manipulation + partitioning |
| 136 | Single Number | XOR cancellation — same "pair off and see what survives" instinct |
| LCCI 17.10 | Majority Element | same problem, existence **not** guaranteed → verification pass |