# LC 14 — Longest Common Prefix

**Difficulty:** Easy · **Sheets:** NeetCode | Striver SDE | Love Babbar 450 | Apna College
**Pattern:** Vertical scanning — compare one column across all strings

---

## 1. Problem

Write a function to find the longest common prefix string among an array of strings. If there is no
common prefix, return `""`.

```
["flower","flow","flight"]   →  "fl"
["dog","racecar","car"]      →  ""
```

Constraints: `1 ≤ strs.length ≤ 200`, `0 ≤ strs[i].length ≤ 200`, lowercase English letters only.

> Note `strs[i].length` can be **0**. An empty string in the array forces the answer to `""`.

---

## 2. The idea

> ### 🔑 The fact
> **The common prefix can be at most as long as the shortest string** — so scan column by column and
> stop at the first column where any string disagrees (or runs out).

Two directions to scan, same complexity:

| | How | Exits early when |
|---|---|---|
| **Vertical** ⭐ | fix a column index, compare that character across all strings, then move to the next column | the prefix is **short** — you stop after a few columns regardless of how long the strings are |
| **Horizontal** | take `strs[0]` as the candidate, shrink it against each string in turn | the prefix is **long** — you keep the candidate through most of the array |

Vertical is the usual choice: on `["dog","racecar","car"]` it exits after one comparison, where
horizontal shrinks a 3-character candidate to nothing one string at a time.

---

## 3. Solution — vertical scanning

```java
class Solution {
    public String longestCommonPrefix(String[] strs) {
        int minLen = Integer.MAX_VALUE;
        for (String s : strs) minLen = Math.min(minLen, s.length());

        int index = 0;
        while (index < minLen) {
            char currChar = strs[0].charAt(index);

            for (int i = 1; i < strs.length; i++) {
                if (strs[i].charAt(index) != currChar) {
                    return strs[0].substring(0, index);
                }
            }
            index++;
        }
        return strs[0].substring(0, index);
    }
}
```

**O(n × m) time**, n = number of strings, m = length of the answer (bounded by the shortest string).
**O(1) auxiliary** beyond the returned string.

### Why `strs[0].substring(0, index)`

The answer is **always a prefix of `strs[0]`** — every character that survived came from it. So
there's no need for a `StringBuilder` accumulating characters one at a time; one `substring` call
at the end does it.

Both exit points end up as the same expression, which is a hint you could `break` out of the outer
loop and return once at the bottom. That makes `index` visibly the only thing the answer depends on.

### The two bound strategies

**Pre-compute the minimum length** (above) — one extra O(n) pass, and it makes the invariant
explicit: *the answer cannot exceed the shortest string.*

**Or fold the bound into the comparison:**

```java
if (index >= strs[i].length() || strs[i].charAt(index) != currChar) {
    return strs[0].substring(0, index);
}
```

No pre-pass; `&&`/`||` short-circuiting handles it, and "the string ended" becomes just another way
of failing to match. Slightly tighter, slightly less obvious about *why* the bound exists. Either is
fine — but **one of them is mandatory** (§5).

---

## 4. The sorting approach

Sorting lets you compare only **two** strings instead of all of them.

```java
public String longestCommonPrefix(String[] strs) {
    Arrays.sort(strs);
    String first = strs[0], last = strs[strs.length - 1];

    int i = 0;
    while (i < first.length() && i < last.length() && first.charAt(i) == last.charAt(i)) i++;
    return first.substring(0, i);
}
```

**Why first-and-last suffice:** after lexicographic sorting, the two extremes are the most
dissimilar pair. Anything between them agrees with both on at least their shared prefix, so no other
string can shorten the answer.

**Complexity: O(n · m · log n)** — comparison sort, and each string comparison is itself up to O(m).
**Worse than plain vertical scanning**, which needs no sort at all. Worth knowing as a neat
observation; not worth shipping.

> ⚠️ It also **mutates the input array**. Mention that unprompted if you offer it.

---

## 5. ⚠️ The bounds trap — and a subtle reason the naive version survives

```java
while (index < strs[0].length()) {                 // 🚩 bounds only strs[0]
    char currChar = strs[0].charAt(index);
    for (int i = 1; i < strs.length; i++) {
        if (strs[i].charAt(index) != currChar) ... // 🚩 no bound on strs[i]
```

The loop assumes `strs[0]` is the **shortest** string. Nothing guarantees that — `["ab","a"]` has a
longer first element, and `"a".charAt(1)` throws.

### But with `Arrays.sort` first, it doesn't crash

Not luck, and worth being able to prove:

> If `strs[i]` is **shorter** than `strs[0]` and sorts **after** it, then `strs[0]` must differ from
> `strs[i]` at some index **within** `strs[i]`'s length — otherwise `strs[i]` would be a prefix of
> `strs[0]` and would sort **before** it.
>
> So every index that would overrun is an index you've already returned before reaching.

Sorting therefore makes the missing bound safe. **But the safety is non-obvious**, depends on an
argument three steps removed from the code, and evaporates the instant you drop the sort — which is
exactly what you should do, since the sort costs a `log n` factor for nothing.

> **Make the bound explicit.** Either pre-compute the minimum length or check it inline. Correct by
> construction beats correct by a lemma about lexicographic ordering.

---

## 6. Other common pitfalls

- **`StringBuilder` accumulating characters** — correct, but the answer is a prefix of `strs[0]`, so
  `substring` is one allocation instead of a growing buffer.
- **Not handling an empty string in the array.** `minLen` becomes 0, the loop never runs, and
  `substring(0, 0)` returns `""` ✅ — but the inline-check version also handles it, via
  `index >= strs[i].length()` on the first iteration. Confirm whichever you wrote.
- **Returning `null` instead of `""`** when there's no common prefix.
- **Comparing with `==` on `String`** anywhere — reference equality. (Not an issue when comparing
  `char`s with `==`, which is what this solution does.)
- **Assuming the array is non-empty** without checking the constraints. Here `strs.length ≥ 1`, so
  `strs[0]` is safe; if it weren't, that's a guard.
- **Forgetting the sort mutates the caller's array** if you take the §4 route.

---

## 7. Dry runs

### `["flower","flow","flight"]`

`minLen = 4` (from `"flow"`).

| index | `currChar` from `strs[0]` | `"flow"` | `"flight"` | result |
|---|---|---|---|---|
| 0 | `f` | `f` ✓ | `f` ✓ | advance |
| 1 | `l` | `l` ✓ | `l` ✓ | advance |
| 2 | `o` | `o` ✓ | **`i`** ✗ | return `substring(0, 2)` = **`"fl"`** |

### `["dog","racecar","car"]`

`minLen = 3`.

| index | `currChar` | `"racecar"` | result |
|---|---|---|---|
| 0 | `d` | **`r`** ✗ | return `substring(0, 0)` = **`""`** |

One comparison. This is where vertical scanning earns its place.

### `["ab","a"]` — the bounds case

`minLen = 1`, so the loop runs for `index = 0` only: `'a'` vs `'a'` ✓, then `index = 1` fails the
bound. Return `substring(0, 1)` = `"a"` ✅

Without the `minLen` bound this is where `"a".charAt(1)` would throw.

---

## 8. Edge cases

| case | expected | ✓ |
|---|---|---|
| single string | that string | ✅ inner loop never runs, `index` reaches `minLen` |
| one string is empty | `""` | ✅ `minLen = 0` |
| all strings identical | the whole string | ✅ |
| no common prefix at all | `""` | ✅ returns on `index = 0` |
| one string is a prefix of all others | that string | ✅ the `minLen` bound stops there |
| `strs[0]` longer than another string | correct | ✅ — the case the naive bound crashes on |

---

## 9. Interview Q&A

**Q: Complexity?**
O(n × m), n strings and m the length of the answer. Every character of the answer is compared
against every string once. O(1) auxiliary.

**Q: Vertical or horizontal scanning?**
Same bound. Vertical exits earlier when the prefix is short, which is the common case — and short
prefixes are where the early exit matters most.

**Q: Why does sorting let you compare only two strings?**
Lexicographic order puts the most dissimilar pair at the two extremes; everything between agrees
with both on at least their shared prefix.

**Q: Is sorting better?**
No — O(n·m·log n) versus O(n·m), plus it mutates the input. It's a nice observation, not an
optimisation.

**Q: Could you do better than O(n × m)?**
Not meaningfully — you must read every character of the answer in every string to confirm it. A trie
built from all the strings also gives O(total characters) and is worth mentioning if the prefix
queries were **repeated**; for a single query it's strictly more work.

**Q: What if you needed the longest common *suffix*?**
Same algorithm scanning from the end of each string, or reverse everything first.

**Q: What about the longest common *substring* rather than prefix?**
A different problem entirely — that's DP (or suffix structures), O(n·m) for two strings and much
harder for many.

---

## 10. Pattern

> **The answer can be at most as long as the shortest input, so fix a column and compare it across
> everything — the first disagreement is the boundary.**

The transferable half: **when an answer must satisfy every element of a collection, bound it by the
weakest element first, then verify position by position.** The minimum length isn't an
implementation detail here — it's the search space.

---

## 11. Takeaways

1. **Bound by the shortest string.** Either pre-compute `minLen` or check `index >= strs[i].length()`
   inline — one of the two is mandatory.
2. **Bounding only `strs[0]` is wrong**, and sorting *accidentally* rescues it via a non-obvious
   lemma. Don't rely on that; make the bound explicit.
3. **The answer is always a prefix of `strs[0]`** — return `substring`, skip the `StringBuilder`.
4. **Vertical scanning exits earlier on short prefixes**, which is the common case.
5. **Sorting costs a `log n` factor and mutates the input** for no asymptotic gain.
6. **Identical expressions at two return points** suggest restructuring to a single exit.
7. **An empty string in the array forces `""`** — check your version handles it.