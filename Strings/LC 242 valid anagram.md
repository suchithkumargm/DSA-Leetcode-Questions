# LC 242 — Valid Anagram

**Difficulty:** Easy · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450 | Apna College
**Pattern:** Frequency counting with a fixed-size array

---

## 1. Problem

Given two strings `s` and `t`, return `true` if `t` is an anagram of `s` — i.e. `t` uses exactly the
same characters with the same multiplicities, in any order.

```
s = "anagram", t = "nagaram"   →  true
s = "rat",     t = "car"       →  false
```

Constraints: `1 ≤ s.length, t.length ≤ 5·10⁴`, **lowercase English letters only**.

> The lowercase-only constraint is what licenses `int[26]`. The follow-up removes it (§6).

---

## 2. The idea

> ### 🔑 The fact
> **Two strings are anagrams iff their character counts match.** Order is irrelevant; only the
> multiset of characters matters.

So: count the characters of `s`, then walk `t` consuming from those counts. If `t` ever needs a
character that isn't available, it's not an anagram.

### Why the length check is load-bearing

```java
if (s.length() != t.length()) return false;
```

This isn't just an early exit. **It's what makes a single consuming pass sufficient.**

Without it, consider `s = "aab"`, `t = "ab"`. Walking `t` never finds a deficit — every character it
asks for is available — so you'd return `true`. To catch that you'd need a **third pass** verifying
every count returned to zero.

> **Equal lengths + no deficit while consuming ⇒ anagram.** Drop either half and the proof breaks.

---

## 3. Solution

```java
class Solution {
    public boolean isAnagram(String s, String t) {
        if (s.length() != t.length()) return false;

        int[] frequency = new int[26];

        for (int i = 0; i < s.length(); i++) {
            frequency[s.charAt(i) - 'a']++;
        }

        for (int i = 0; i < t.length(); i++) {
            int idx = t.charAt(i) - 'a';
            if (frequency[idx] == 0) return false;
            frequency[idx]--;
        }

        return true;
    }
}
```

**O(n) time · O(1) space.**

> ⚠️ **The space is O(1), not O(26) and not O(n).** The array is 26 slots whatever the input size —
> constants don't appear inside big-O. Say *"O(1), a fixed 26-element array"* rather than "O(26)".

### `== 0` vs `<= 0`

Counts can never go negative — you return *before* decrementing a zero. So `== 0` is exact and
states the invariant; `<= 0` is defensive and equally correct. Either is fine, but knowing the count
can't go negative is the point.

### One array, not two

Counting both strings into separate arrays and comparing them also works, and `Arrays.equals` makes
it short. The consume-from-one-array version does the same work with half the memory and exits early
on the first mismatch.

---

## 4. Alternative — sort both strings

```java
public boolean isAnagram(String s, String t) {
    if (s.length() != t.length()) return false;
    char[] a = s.toCharArray(), b = t.toCharArray();
    Arrays.sort(a);
    Arrays.sort(b);
    return Arrays.equals(a, b);
}
```

Two anagrams sort to the same sequence, so equality of the sorted forms is exactly the condition.

**O(n log n) time · O(n) space** (the two char arrays; Java can't sort a `String` in place).

Worse on both axes. Worth mentioning as the one-liner, then improving to counting — the comparison
*"sorting is O(n log n), counting is O(n) because the alphabet is bounded"* is the point being
tested.

---

## 5. Common pitfalls

**Omitting the length check.** The single-pass version then returns `true` for `s = "aab"`,
`t = "ab"` (§2). Either keep the guard or add a third pass checking all counts are zero.

**Counting both strings up and comparing, but forgetting the lengths still differ in multiplicity.**
Two arrays compared with `Arrays.equals` handles this correctly — but hand-rolled comparison loops
sometimes check only "every character in `t` appears in `s`", which is not the same claim.

**`O(26)` or `O(n)` for the space.** It's O(1).

**Assuming `s.equals(t)` is relevant.** Anagrams are usually *not* equal strings.

**Using `==` to compare the strings or their chars as objects.** `char` comparison with `==` is
fine; `String` comparison with `==` is reference equality.

**Hardcoding `- 'a'` when the input isn't guaranteed lowercase.** A capital or a space produces a
negative or out-of-range index → `ArrayIndexOutOfBoundsException`, not a wrong answer. See §6.

---

## 6. The follow-up — Unicode input

> *"What if the inputs contain Unicode characters?"*

`int[26]` no longer works: the alphabet isn't 26 symbols, and `- 'a'` is meaningless.

**Use a `HashMap<Character, Integer>`** — count up from `s`, consume with `t`, same two-pass
structure.

```java
Map<Character, Integer> freq = new HashMap<>();
for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);

for (char c : t.toCharArray()) {
    Integer count = freq.get(c);
    if (count == null || count == 0) return false;
    freq.put(c, count - 1);
}
return true;
```

**O(n) time · O(k) space**, k = distinct characters. No longer O(1), because the alphabet is
unbounded.

`merge(c, 1, Integer::sum)` returns the new count in one hash lookup instead of a `get`-then-`put`
pair.

> ⚠️ **The deeper caveat, worth raising unprompted:** a Java `char` is a UTF-16 code unit, not a
> code point. Characters outside the Basic Multilingual Plane (emoji, some CJK extensions) occupy
> **two** chars — a surrogate pair — so a `char`-keyed map splits them. For genuine Unicode
> correctness you'd iterate `s.codePoints()` and key on `int`.
>
> And "same characters" gets philosophically murky with combining marks and normalisation forms:
> `é` can be one code point or two. Naming that distinction is a strong signal; nobody expects you
> to implement it.

**A middle option:** for ASCII generally, `int[128]`. Still O(1) space, handles capitals,
punctuation and digits, and no hashing cost.

---

## 7. Dry run — `s = "anagram"`, `t = "nagaram"`

**Pass 1**, counting `s`:

| char | a | n | g | r | m |
|---|---|---|---|---|---|
| count | 3 | 1 | 1 | 1 | 1 |

**Pass 2**, consuming `t = "nagaram"`:

| i | char | count before | action |
|---|---|---|---|
| 0 | n | 1 | → 0 |
| 1 | a | 3 | → 2 |
| 2 | g | 1 | → 0 |
| 3 | a | 2 | → 1 |
| 4 | r | 1 | → 0 |
| 5 | a | 1 | → 0 |
| 6 | m | 1 | → 0 |

No deficit, all counts land at 0. Return **true** ✅

### The failure case — `s = "rat"`, `t = "car"`

Counts: `r:1, a:1, t:1`. Pass 2 starts with `'c'`, whose count is 0 → **return false** immediately.

### Why the length guard matters — `s = "aab"`, `t = "ab"`

Lengths differ, so the guard returns false up front. **Without it:** `'a'` → 2→1, `'b'` → 1→0, no
deficit, returns `true`. Wrong, and the leftover `a:1` is never inspected.

---

## 8. Edge cases

| case | expected | ✓ |
|---|---|---|
| different lengths | false | ✅ the guard |
| identical strings | true | ✅ a string is an anagram of itself |
| single character, same | true | ✅ |
| single character, different | false | ✅ |
| `t` has a character absent from `s` | false | ✅ count is 0 on first access |
| same characters, different multiplicities and equal lengths, e.g. `"aabb"` / `"abbb"` | false | ✅ `'b'` runs out |
| empty strings | true | ✅ (not in constraints, but both loops skip) |

---

## 9. Interview Q&A

**Q: Complexity?**
O(n) time, O(1) space — the frequency array is a fixed 26 slots regardless of input length.

**Q: Why is the space O(1) and not O(n)?**
The alphabet is bounded by the constraints. Space depends on the alphabet size, not the string
length.

**Q: Why the length check?**
It's what makes one consuming pass sufficient. Without it, `s` having *extra* characters goes
undetected, and you'd need a third pass verifying all counts hit zero.

**Q: Could you do it in one pass?**
Yes — increment for `s[i]` and decrement for `t[i]` in the same loop (lengths being equal), then
check all 26 counts are zero at the end. Same O(n), but it loses the early exit and still needs the
final verification pass, so it isn't obviously better.

**Q: What about Unicode?**
`HashMap<Character, Integer>`, O(k) space. And a `char` is a UTF-16 code unit, so genuine Unicode
needs code points. §6.

**Q: Could you sort instead?**
Yes, O(n log n) time and O(n) space. Counting is better because the alphabet is bounded.

**Q: Grouping many strings into anagram sets?**
LC 49. The sorted string, or a 26-length count signature, becomes the map key — the same two
characterisations of "anagram" used as a hash key instead of a comparison.