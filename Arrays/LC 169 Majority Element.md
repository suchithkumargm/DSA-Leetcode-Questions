# LC 169 — Majority Element (Easy)

> **Problem:** Given an array `nums` of size `n`, return the majority element — the element that
> appears **more than ⌊n/2⌋ times**.
> **The problem guarantees the majority element always exists.** (Remember this — two of the four
> approaches below are wrong without it.)
> Follow-up: solve it in **O(n) time and O(1) space**.
>
> Example: `[3,2,3]` → `3`
> Example: `[2,2,1,1,1,2,2]` → `2`

**Session record — Sep 20**
| | |
|---|---|
| HashMap version | solved cold, **3 min**, unaided |
| Boyer-Moore | named and coded **unaided**; correctness *proof* was wrong |
| Didn't know | the bitwise-count approach |
| Habit flagged | reached for the keyboard before surveying the approach space |

---

## 0. Why "more than n/2" is the whole problem

Every approach below is a different way of exploiting one fact:

> **An element occupying more than half the array cannot be "outvoted", "out-positioned", or
> "out-counted" by everything else combined — because everything else combined is strictly
> less than half.**

- Sorting exploits it **positionally** (it must cover the middle index).
- Boyer-Moore exploits it by **cancellation** (it can't be fully paired off).
- Bit counting exploits it **per bit** (each of its set bits is set in >n/2 numbers).

If you remember only that sentence, you can re-derive all three. That's the difference between
remembering a pattern and remembering code.

---

## 1. Approach 1 — HashMap counting

### Intuition

Count everything, return whatever crosses the threshold. No cleverness required — this is the
answer you should produce in under 3 minutes and then immediately say *"but I can do better on
space."*

### First version (what I wrote) — with two flaws

```java
class Solution {
    public int majorityElement(int[] nums) {
        HashMap<Integer, Integer> map = new HashMap<>();

        for (int num : nums) {
            map.put(num, map.getOrDefault(num, 0) + 1);
        }

        for (int num : nums) {                      // 🚩 flaw 2
            if (map.get(num) > nums.length / 2) return num;
        }

        return nums[0];                             // 🚩 flaw 1 — unreachable
    }
}
```

**🚩 Flaw 1 — unreachable dead code.** The problem *guarantees* a majority element, so the second
loop always returns. `return nums[0]` can never execute. It's there only because Java demands a
return on every path. Know that it's dead; an interviewer will ask why it's there.

**🚩 Flaw 2 — redundant second pass over `nums`.** Two separate problems with it:
1. The winner can be detected **during** the counting pass — the moment any count crosses n/2,
   you have the answer and can return early.
2. Even if you keep two passes, the second should iterate **`map.entrySet()`**, not `nums`.
   Iterating `nums` re-checks every duplicate: for `[2,2,2,2,2,1]` you check `2` five times.
   The map has at most n entries and usually far fewer.

### Final version

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

### Java detail — one lookup instead of two

`map.put(...)` followed by `map.get(num)` hashes the key **twice**. `merge` returns the new value
directly:

```java
int count = map.merge(num, 1, Integer::sum);
if (count > nums.length / 2) return num;
```

Same complexity, one hash operation instead of two, and it reads better. Worth having in the
toolkit — it shows up constantly in frequency-counting problems.

### Complexity

| | | why |
|---|---|---|
| **Time** | `O(n)` | one pass; HashMap ops are **amortised** O(1) — say "amortised" out loud, it's not a guarantee |
| **Space** | `O(n)` | worst case every element distinct… except it can't be, given the guarantee. Still O(n) in the general bound. |

> **Say "amortised."** HashMap is O(1) *amortised*, not worst-case — a pathological hash
> distribution degrades it. Java 8+ converts long buckets to red-black trees, so the true worst
> case per operation is O(log n), not O(n). Knowing that is a free credibility point.

---

## 2. Approach 2 — Sorting

### The obvious version

Sort, then one pass counting runs, return the run longer than n/2. **O(n log n)** time.

### The sharp version ⭐

> **After sorting, the majority element is at index `n/2`. Always.**

No counting pass at all:

```java
Arrays.sort(nums);
return nums[nums.length / 2];
```

**Why:** the majority element occupies more than half the array, and after sorting its copies are
contiguous. A contiguous block longer than n/2 **cannot avoid** straddling the middle index —
there isn't enough room on either side of the midpoint to fit it entirely.

Sanity check, `n = 7`, needs ≥4 copies, middle index 3:
- pushed as far **left** as possible: indices `0,1,2,3` → covers index 3 ✅
- pushed as far **right** as possible: indices `3,4,5,6` → covers index 3 ✅

### Complexity

| | | |
|---|---|---|
| **Time** | `O(n log n)` | dominated by the sort |
| **Space** | `O(log n)` | Java's dual-pivot quicksort on primitives — recursion stack. **Not O(1).** Don't claim it is. |

Strictly worse than both O(n) approaches, but it's a good thing to *offer* — it shows you see
structure, not just counters.

---

## 3. Approach 3 — Boyer-Moore Voting ⭐ (the intended answer)

### Code

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

### Two design choices, both defensible

**The zero-check lives inside the `else`.** `count` can only reach 0 via a decrement, so testing
it after an increment is wasted work. Correct reasoning, and the kind of justification to give
out loud rather than discover by testing.

**Naming: `candidate`, not `majorityEle`.** Mid-loop the variable may be holding something that
gets discarded three iterations later — calling it "the majority element" is a claim the variable
can't back up yet. `candidate` is honest, and the name itself reminds you that a **verification
pass** exists for the variants where the guarantee is removed (§5).

### The standard seeding form

Mine starts `candidate = nums[0], count = 1, i = 1`. The editorial form is:

```java
int candidate = 0, count = 0;
for (int num : nums) {
    if (count == 0) candidate = num;
    count += (num == candidate) ? 1 : -1;
}
```

Both correct. Learn the `count = 0` form too, because:
- it's what you'll see in every editorial, and
- it **generalises cleanly to LC 229** (majority > n/3), where you carry two candidates and two
  counters and the zero-check placement matters.

### 🔴 Correctness — I got this wrong

**What I said:** *"the count won't go below 0."*

That's **not a proof of anything.** Every counter that resets at zero has that property. It says
nothing about *which* element survives. I'd have failed this follow-up in an interview.

**The correct argument — pairing:**

> Each decrement **pairs** one occurrence of the current candidate against one occurrence of some
> other element, and cancels **both**.
>
> The majority element has **more than n/2** copies, so everything else combined has **fewer than
> n/2** copies. Every cancellation destroys at most one copy of the majority element, and there
> are strictly fewer non-majority elements available to do the destroying than there are copies
> to destroy.
>
> Therefore the majority element cannot be fully cancelled. Something survives — and only the
> majority element can be what survives.

**Test whether you've actually got it:** the proof must explain why the answer is *the majority
element specifically*, not merely why the algorithm terminates or why a counter stays
non-negative. If your sentence would be equally true of a non-majority input, it isn't a proof.

### ⚠️ What the proof depends on

**The guarantee that a majority element exists.** Remove it and Boyer-Moore returns *a* number
with no meaning at all — `[1,2,3]` yields `3`, which is not a majority of anything. See §5.

### Complexity

| | |
|---|---|
| **Time** | `O(n)` — single pass |
| **Space** | `O(1)` — two ints |

This is the answer the follow-up wants.

---

## 4. Approach 4 — Bit counting (didn't know this one)

### Intuition

Build the answer **one bit at a time** instead of finding the element as a whole.

Pick any bit position `i`. If the majority element has bit `i` set, then more than n/2 of the
numbers in the array have bit `i` set — because the majority element *alone* accounts for more
than n/2 of them. And if its bit `i` is clear, then fewer than n/2 numbers can have it set.

> So: **bit `i` of the answer is 1 ⟺ more than n/2 array elements have bit `i` set.**

Count all 32 positions, reassemble.

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

### The trap ⚠️

The **sign bit (i = 31)**. `1 << 31` is `Integer.MIN_VALUE`, and `result |= (1 << 31)` correctly
sets it — two's complement means this works out. But be deliberate about it: use `>>` or `>>>`
consistently and test with a negative-value input. LC 169's constraints allow negatives.

### Complexity

| | |
|---|---|
| **Time** | `O(32n)` = `O(n)` — the constant is the point, it's ~32× slower than Boyer-Moore in practice |
| **Space** | `O(1)` |

**Why learn it if it's strictly worse?** Because the *technique* — solving bitwise for the answer
rather than searching for the answer — is the core idea in **LC 137 (Single Number II)** and
**LC 260 (Single Number III)**, where it isn't optional.

---

## 5. Follow-ups you will actually be asked

### "What if a majority element is NOT guaranteed?"

Boyer-Moore's candidate becomes meaningless. Add a **verification pass**:

```java
int count = 0;
for (int num : nums) if (num == candidate) count++;
return count > nums.length / 2 ? candidate : -1;
```

Still O(n) time, O(1) space — two passes instead of one. **This is the most common follow-up on
this problem.** Have it ready.

Note which approaches survive the removal of the guarantee:
| approach | survives without the guarantee? |
|---|---|
| HashMap | ✅ threshold check is explicit |
| Sorting → `nums[n/2]` | ❌ needs a verification pass |
| Boyer-Moore | ❌ needs a verification pass |
| Bit counting | ❌ needs a verification pass |

### "Now find all elements appearing more than ⌊n/3⌋ times." → **LC 229**

Generalised Boyer-Moore: at most **two** such elements can exist (three would need >n, a
contradiction). Carry two candidates and two counters, then **verify both** at the end —
verification is mandatory here, not optional, because n/3-majority elements aren't guaranteed
to exist.

The `count = 0` seeding form (§3) is the one that generalises. Do this problem next.

### "What about a stream, one pass, can't store the array?"

Boyer-Moore works — it's O(1) space and single-pass. That's exactly why it exists. You can't run
the verification pass, though, so you need the existence guarantee.

---

## 6. Approach comparison

| # | Approach | Time | Space | Notes |
|---|---|---|---|---|
| 1 | HashMap counting | O(n) | **O(n)** | 3-minute answer; state it, then improve it |
| 2 | Sort + count runs | O(n log n) | O(log n) | strictly dominated |
| 2b | Sort → `nums[n/2]` | O(n log n) | O(log n) | one line; good "I see structure" answer |
| 4 | Bit counting | O(32n) | **O(1)** | worse constant; technique matters for LC 137/260 |
| **3** | **Boyer-Moore** | **O(n)** | **O(1)** | ⭐ **the answer** — single pass, two ints |

There's also randomised sampling (pick a random index, verify, repeat — O(n) expected). Mention
it only if asked for exotica.

---

## 7. Edge cases checklist

| case | expected | Boyer-Moore |
|---|---|---|
| `[1]` | 1 | ✅ loop never runs, returns `nums[0]` |
| `[2,2]` | 2 | ✅ |
| all elements identical | that element | ✅ count only ever increments |
| majority at the very end, e.g. `[1,2,3,3,3,3,3]` | 3 | ✅ candidate flips until 3 takes over |
| majority split, e.g. `[3,1,3,2,3]` | 3 | ✅ cancellations don't exhaust it |
| negative numbers | works | ✅ (⚠️ only the *bitwise* approach needs sign-bit care) |
| `n` even, exactly n/2 copies | **not a majority** | needs `>`, never `>=` — off-by-one trap |

> ⚠️ The threshold is **strictly greater than** ⌊n/2⌋. `>` not `>=`. For `n = 4`, three copies
> qualify and two do not.

---

## 8. 🔴 Habit flagged this session

**Solved in 3 minutes, then went straight to coding the next approach without surveying first.**

Asked twice for *"every approach, with time and space, and the best achievable space,"* I
answered neither time — I sent improved code instead. Only after being pushed did four approaches,
a proof, and the O(1) solution come out.

> **The gap between a 3-minute answer and a complete one is entirely in the surveying step —
> and in an interview nobody pushes you. You have to do it unprompted.**

**Fix — before typing anything, say out loud:**
1. Brute force, and its cost.
2. What resource the follow-up is asking me to reduce — time, space, or passes.
3. Every approach I can see, with time *and* space for each.
4. Which one I'm writing, and why not the others.

Steps 1–4 cost ninety seconds. The interviewer is scoring them.

**Related failure from earlier sessions:** this is the same shape as the "pick later to optimise"
note left open on LC 25 for nine days — stopping at the first thing that passes. See LC 25 §7–8.

### What actually improved this session ✅

- Stated complexity **unprompted**, and said "amortised" without being asked.
- **Justified** the `else`-placement of the zero-check by reasoning, not by testing it.
- Zero submit-to-verify cycles.
- Pushed back when I mislabelled the ledger entry — that's the right instinct. The ledger is
  only useful if it's accurate.

---

## 9. One-line pattern

> **Pair each occurrence of the candidate against one non-candidate and cancel both — an element
> with more than n/2 copies cannot be fully cancelled, so whatever survives is the answer.**

Note this sentence reconstructs the code. *"Increment if same, decrement if not, reset at zero"*
does not — it's the code, restated. **Memorise the reason, not the steps.**

---

## 10. Takeaways for future problems

1. **"More than n/2" is a structural guarantee, not just a counting fact.** It forces positional,
   cancellation, and per-bit consequences. Ask what a threshold *forces* before reaching for a map.
2. **Boyer-Moore's proof is the pairing argument.** "The count never goes negative" proves nothing.
   A proof must explain why *this specific element* survives.
3. **Boyer-Moore needs the existence guarantee.** Without it, add a verification pass. Expect the
   follow-up.
4. **Name it `candidate`, not `majorityEle`.** The name should reflect what the variable can
   actually claim mid-loop.
5. **`map.merge(k, 1, Integer::sum)`** returns the new count — one hash lookup instead of two.
6. **Iterate the map, not the array,** when looking for a winner among counts.
7. **Sorting in Java is O(log n) space, not O(1).** Recursion stack. Don't overclaim.
8. **Solving bitwise for the answer** is a general technique → LC 137, 260.
9. **Say "amortised" about HashMap.** Java 8+ worst case per op is O(log n), not O(n) (treeified
   buckets).
10. **Survey before typing.** Brute force → what resource to reduce → all approaches with costs →
    pick one and justify it. Ninety seconds, and it's scored.

---

## 11. Related problems (same muscles)

| # | Problem | Shared idea |
|---|---|---|
| **229** | Majority Element II (> n/3) | ⭐ generalised Boyer-Moore, two candidates, **verification mandatory** — do this next |
| 1150 | Check If a Number Is Majority Element | binary search on a sorted array for the same threshold |
| 137 | Single Number II | solve bitwise for the answer (§4's technique, where it's required) |
| 260 | Single Number III | bit manipulation + partitioning |
| 136 | Single Number | XOR cancellation — same "pair off and see what survives" instinct |
| 面试 17.10 | Majority Element (LCCI) | same problem, existence **not** guaranteed → verification pass |

---

*Sep 20 — solved. HashMap in 3 min cold; Boyer-Moore named and coded unaided. Correctness proof
was wrong (fixed above). Bitwise approach was new. **Next: LC 229**, which is this problem with the
verification pass made mandatory.*