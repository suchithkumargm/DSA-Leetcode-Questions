# LC 496 — Next Greater Element I

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE
**Pattern:** Monotonic stack — the canonical introduction

---

## 1. Problem

`nums1` is a **subset** of `nums2`, and both contain **distinct** integers. For each element of
`nums1`, find its **next greater element** in `nums2`: the first element to its right that is larger.
If there is none, return `-1`.

```
nums1 = [4,1,2], nums2 = [1,3,4,2]   →  [-1, 3, -1]
```

Constraints: `1 ≤ nums1.length ≤ nums2.length ≤ 1000`, values up to 10⁴, all distinct, `nums1 ⊆ nums2`.

**Follow-up:** *Could you find an O(nums1.length + nums2.length) solution?*

> The constraint `n ≤ 1000` is **permissive** — it exists so the O(n²) version passes. The follow-up
> is where the content is.

---

## 2. Two guarantees the solution leans on

**Distinct values** — this is what makes a `HashMap<Integer, Integer>` keyed by *value* legal. With
duplicates, "the next greater element of 5" isn't well defined when there are three 5s, and the whole
value-keyed approach collapses (you'd have to key by index and the problem would be shaped
differently).

**`nums1 ⊆ nums2`** — every query is guaranteed to be present. It's why `map.get(nums1[i])` can't
miss *as a key*. (It can still be absent from the map for a different reason — see §6.)

Say both out loud. They look incidental and they're load-bearing.

---

## 3. Brute force

For each element of `nums2`, scan right until something bigger appears.

```java
public int[] nextGreaterElement(int[] nums1, int[] nums2) {
    HashMap<Integer, Integer> map = new HashMap<>();

    for (int i = 0; i < nums2.length; i++) {
        int next = -1;
        for (int j = i + 1; j < nums2.length; j++) {
            if (nums2[j] > nums2[i]) { next = nums2[j]; break; }
        }
        map.put(nums2[i], next);
    }

    int[] ans = new int[nums1.length];
    for (int i = 0; i < nums1.length; i++) ans[i] = map.get(nums1[i]);
    return ans;
}
```

**O(n² + m) time · O(n) space.** Correct, and at `n ≤ 1000` it's fast in wall-clock terms (§8).

### Where the repeated work is

```
nums2 = [5, 4, 3, 2, 1, 6]
```

The scan from index 0 walks `4,3,2,1,6`. From index 1 it walks `3,2,1,6`. From index 2, `2,1,6`.
**The same suffix, re-walked every time.** Descending runs are the worst case, and they're exactly
where the information being thrown away is most useful.

---

## 4. ⚠️ Why a prefix/suffix-max array doesn't work

The instinct for "something about elements to my right" is a suffix aggregate. It fails, and the
reason is worth internalising:

```
nums2 = [5, 1, 6, 10]

suffix max from index 1  =  10
next greater of 1        =  6      ← the FIRST one larger, not the largest
```

**Prefix/suffix aggregates answer "what is the extreme over this range."** Next-greater is
*positional* and depends on the query value, so no precomputed aggregate gives it to you.

---

## 5. ⭐ The insight — which elements are discardable

Take `nums2 = [5, 4, 3, 2, 1, 6]`. You're standing at `5`, and `4, 3, 2, 1` lie between it and `6`.
Ask whether anything to the **left** of `5` could ever need one of them:

| a left-hand element `x` | what it hits first | can `4,3,2,1` be its answer? |
|---|---|---|
| `x < 5` | **5** — closer and bigger | no, the answer is `5` |
| `x > 5` | also bigger than 4,3,2,1 | no, none of them qualify |

Either way: **no.**

> ### 🔑 The fact
> **An element with a bigger element to its right can never be the next-greater for anything further
> left — the bigger, closer one shadows it.**

So at any moment you need to remember only the elements that are **still candidates**: those with
nothing bigger yet seen to their right.

And the structure follows by force: each new element **discards** the candidates it shadows, and
those are the **most recently added** ones — because they're the nearest. Add newest on top, remove
from the top. **That's a stack.**

---

## 6. Solution — monotonic stack

```java
class Solution {
    public int[] nextGreaterElement(int[] nums1, int[] nums2) {
        HashMap<Integer, Integer> map = new HashMap<>();
        Deque<Integer> stack = new ArrayDeque<>();

        for (int x : nums2) {
            while (!stack.isEmpty() && stack.peek() < x) {
                map.put(stack.pop(), x);        // the popped element's answer IS x
            }
            stack.push(x);                      // every element becomes a candidate
        }

        int[] ans = new int[nums1.length];
        for (int i = 0; i < nums1.length; i++) {
            ans[i] = map.getOrDefault(nums1[i], -1);   // never popped ⇒ no next greater
        }
        return ans;
    }
}
```

**O(n + m) time · O(n) space.**

### The three decisions

**Discard before pushing.** The popping condition is the whole algorithm; the push has no condition
at all. **Every element gets pushed, unconditionally, at the end of its own iteration** — because
every element is a candidate for whatever comes to its right, until something shadows it.

**Record on the pop.** The moment you pop an element, you've just discovered its answer: the element
that shadowed it. That's the only moment you know it, so that's where the `map.put` goes.

**The leftover stack is the `-1` case.** Whatever is still held when the loop ends never found
anything bigger. You can drain the stack writing `-1`s, or — simpler — let the lookup supply the
default. `getOrDefault` makes the second loop handle it and you never touch the stack again.

### Why it's called *monotonic*

The stack ends up holding elements in **decreasing order, bottom to top**. That isn't enforced — it's
a *consequence* of popping everything smaller before each push. Check it against any trace; it's the
property the pattern is named for, and it's what makes the top always the nearest surviving candidate.

### ⭐ Why one pass is O(n) despite the nested `while`

Count **total** pops across the entire outer loop, not per iteration:

> **Every element is pushed exactly once and popped at most once.** So the total number of pop
> operations over the whole run is at most `n`, however they're distributed. The outer loop is `n`
> iterations, the inner work sums to `≤ n`, total **O(n)**.

A single iteration can pop many elements — on `[4,3,2,9]` the `9` resolves three at once — but that
iteration is "spending" pops that earlier iterations didn't use.

**This is the same amortisation argument as LC 232**, and for the same underlying reason: *a popped
element is finished.* It has its answer and never returns to the stack. Work done is progress, not
rearrangement.

> Contrast **LC 225**, where no amortisation exists because each rotation is undone by the next push.
> **Before claiming amortised O(1), ask whether the expensive step is progress or just
> rearrangement.**

---

## 7. Right-to-left variant

Same pattern, bookkeeping inverted — worth knowing because many write-ups use it.

```java
for (int i = nums2.length - 1; i >= 0; i--) {
    while (!stack.isEmpty() && stack.peek() <= nums2[i]) stack.pop();   // discard, don't record
    map.put(nums2[i], stack.isEmpty() ? -1 : stack.peek());             // the answer is right there
    stack.push(nums2[i]);
}
```

| | left → right | right → left |
|---|---|---|
| the pop means | *this popped element's* answer is the current one | this popped element is useless, discard it |
| when you learn an answer | for **other** elements, as you pop them | for the **current** element, by peeking |
| `-1` handling | leftover stack, or `getOrDefault` | `stack.isEmpty()` at the moment of writing |

Left-to-right resolves *other* elements; right-to-left resolves *itself*. Both O(n). Left-to-right
generalises more naturally to problems where you need the index distance (LC 739, Daily Temperatures).

---

## 8. Why the O(n) version can measure *slower*

At `n ≤ 1000` the asymptotic win cannot show up, and there's a real constant-factor cost pulling the
other way:

| | brute force | monotonic stack |
|---|---|---|
| inner comparison | `nums2[j] > nums2[i]` — raw `int` array reads | `stack.peek() < x` — **unboxes an `Integer`** |
| allocation | none | an `Integer` box per push, plus the deque's array |

`Deque<Integer>` and `HashMap<Integer,Integer>` box every value. 10⁶ cache-friendly `int` comparisons
genuinely can beat a few thousand pointer-chasing boxed operations.

**If the constant matters**, use a plain `int[]` as the stack:

```java
int[] stack = new int[nums2.length];
int sp = 0;                      // sp = next free slot
// push:   stack[sp++] = x;
// peek:   stack[sp - 1]
// pop:    stack[--sp]
// empty:  sp == 0
```

No boxing, one allocation, and the size bound is known in advance.

> **The crossover is the point.** At `n = 10⁵`, O(n²) is 10¹⁰ operations — minutes — while the stack
> version is still microseconds. The input size here simply can't demonstrate it. Ship the O(n) one;
> it's what the follow-up asks for.

---

## 9. Common pitfalls

### 🚩 `if` instead of `while`

```java
if (!stack.isEmpty() && stack.peek() < x) { … }    // 🚩 resolves at most one candidate
```

On `[4, 3, 2, 9]` the `9` must resolve **three** held elements in a single iteration. An `if` records
only the top one and leaves `3` and `4` on the stack forever — they'd silently come back as `-1`.

### 🚩 Forgetting the leftover stack → NullPointerException

```java
ans[i] = map.get(nums1[i]);      // 🚩 null for anything never popped
```

Trace `nums2 = [1,2,3,4]`: 1, 2 and 3 all get shadowed and recorded, but **4 is still on the stack**
when the loop ends. `map.get(4)` returns `null`, and unboxing `null` to `int` throws.

Note the failure mode: `nums1 ⊆ nums2` guarantees the key *exists in nums2*, not that it reached the
*map*. Those are different claims. `getOrDefault(…, -1)` closes it.

### 🚩 Pushing conditionally

The push is unconditional. Any guard on it drops candidates and produces spurious `-1`s.

### 🚩 Recording on the push rather than the pop

You don't know an element's answer when you push it — that's the entire reason it goes on the stack.
The answer is discovered at the moment something shadows it, i.e. at the pop.

### 🚩 Reaching for a prefix/suffix-max array

§4 — the largest to the right is not the first larger to the right.

### 🚩 Assuming the map keys are safe with duplicates

Value-keyed lookup is only legal because the problem guarantees distinct values (§2).

---

## 10. Dry runs

### `nums2 = [1, 3, 4, 2]`, `nums1 = [4, 1, 2]`

Stack written **top → bottom**.

| x | pops (and what gets recorded) | stack after push | map |
|---|---|---|---|
| 1 | — (empty) | `[1]` | `{}` |
| 3 | pop 1 → `map[1] = 3` | `[3]` | `{1:3}` |
| 4 | pop 3 → `map[3] = 4` | `[4]` | `{1:3, 3:4}` |
| 2 | — (`4 < 2` is false) | `[2, 4]` | `{1:3, 3:4}` |

Leftover stack: `2` and `4` — neither ever shadowed, so both default to `-1`.

Lookups: `4 → -1`, `1 → 3`, `2 → -1` ⇒ **`[-1, 3, -1]`** ✅

Note the final stack `[2, 4]` top-to-bottom is **increasing upward** = decreasing bottom-to-top.
That's the monotonic invariant.

### `nums2 = [5, 4, 3, 2, 1, 6]` — the multi-pop case

| x | stack before | pops | map additions |
|---|---|---|---|
| 5 | `[]` | — | |
| 4 | `[5]` | — | |
| 3 | `[4,5]` | — | |
| 2 | `[3,4,5]` | — | |
| 1 | `[2,3,4,5]` | — | |
| 6 | `[1,2,3,4,5]` | **all five** | `1:6, 2:6, 3:6, 4:6, 5:6` |

One iteration performs five pops — and it's exactly the iteration the brute force re-walked the whole
suffix for, five separate times. The five pops are paid for by the five push-only iterations before
them. **Total pops across the run: 5. Total pushes: 6. Linear.**

---

## 11. Edge cases

| case | result |
|---|---|
| `nums2` strictly increasing `[1,2,3]` | every element resolves immediately; stack holds only the last |
| `nums2` strictly decreasing `[3,2,1]` | nothing ever pops; all `-1`; stack holds everything ✅ the `getOrDefault` case |
| single element | pushed, never popped, `-1` |
| `nums1 == nums2` | fine — every key is in the map or defaults |
| the maximum of `nums2` | always `-1`; nothing can shadow it |
| `nums1` length 1 | fine |

---

## 12. Interview Q&A

**Q: Complexity?**
O(n + m) time, O(n) space. The first loop is linear because every element is pushed once and popped
at most once, so total pops across the whole run are bounded by `n`.

**Q: Why doesn't the nested `while` make it quadratic?**
Because the inner work is bounded globally, not per-iteration. An iteration that pops five elements
is spending pops that five earlier iterations didn't use.

**Q: Why is a stack the right structure?**
An element is shadowed by the first bigger element to its right, and the candidates you rule out are
always the most recently seen ones. Newest in, newest out — LIFO.

**Q: Why is the stack decreasing?**
Consequence, not constraint: you pop everything smaller before each push, so nothing smaller can sit
below something bigger.

**Q: What if `nums2` had duplicates?**
A value-keyed map breaks. You'd key by index and return answers positionally.

**Q: What if you needed the *distance* to the next greater element rather than its value?**
Push **indices** instead of values; on each pop, `currentIndex − poppedIndex` is the distance. That's
LC 739 (Daily Temperatures) — same loop, different payload.

**Q: What if the array were circular?**
LC 503 — iterate `2n` times using `i % n`, and only push during the first pass. The second pass just
resolves leftovers.

**Q: Next *smaller* element instead?**
Flip the comparison to `>`. The stack becomes increasing. Every variant of this pattern is a
comparison flip plus a direction choice.

**Q: Why not a suffix-max array?**
It gives the largest element to the right, not the first larger one. §4.

---

## 13. Pattern

> **An element with a bigger element to its right can never be the answer for anything further left —
> so keep only the unshadowed candidates, newest on top, and discard them as each new element
> arrives.**

The transferable half: **when a scan repeatedly re-walks the same suffix, ask what the scan *learned*
that it's throwing away.** Here the answer is "these elements are permanently ruled out", and the
structure that holds "still-relevant, newest first, discard from the newest end" is a stack.

And the complexity half, which generalises past this problem:

> **Work amortises only when it's progress.** A popped element here is finished forever — same as LC
> 232's transferred element, and unlike LC 225's rotation, which the next push undoes.

---

## 14. Takeaways

1. **The constraint `n ≤ 1000` is permissive**, not restrictive — it lets the brute force pass. The
   follow-up is the problem.
2. **Suffix-max ≠ next-greater.** Aggregates answer "extreme over a range"; this is positional.
3. **The shadowing rule is the whole insight:** something bigger *and closer* makes everything behind
   it unreachable.
4. **Pop conditionally, push unconditionally.** All the logic lives in the discard.
5. **Record on the pop** — that's the only moment an element's answer is known.
6. **`while`, never `if`.** One element can resolve many.
7. **The leftover stack is the `-1` case**, and `map.get` on it returns `null` → NPE. Use
   `getOrDefault`.
8. **Monotonicity is a consequence, not an invariant you maintain.**
9. **O(n) because each element is pushed once and popped at most once** — a global bound, not a
   per-iteration one.
10. **Boxing is a real constant factor.** `int[]` with a manual pointer removes it when it matters.
11. **Distinct values are what make a value-keyed map legal.** Check that guarantee before relying
    on it.