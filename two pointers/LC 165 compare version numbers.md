# LC 165 — Compare Version Numbers

**Difficulty:** Medium
**Topics:** Two Pointers, String Parsing
**Language:** Java
**Pattern:** Parse-and-compare with unequal-length inputs, where "missing" defaults to a value

---

## 1. Problem

Given two version strings like `"1.2"` and `"1.10"`, compare them.

- A version is revisions joined by dots: `"1.10.3"` → revisions `1`, `10`, `3`.
- Revisions are compared **as integers**, so leading zeros don't matter: `"01"` == `"1"`.
- If one version has fewer revisions, the missing ones are treated as **`0`**. So `"1.0"` == `"1"`.

Return `-1` if version1 < version2, `1` if version1 > version2, else `0`.

---

## 2. The One Insight

> **The unit of comparison is the revision, not the character.**

Everything else follows from this. Once you compare whole integers between the dots, three things become free:

| Tricky case | Why it just works |
|---|---|
| `"1.2"` vs `"1.10"` | `2 < 10` as integers. Char-by-char would say `'2' > '1'` — wrong. |
| `"01"` vs `"1"` | Both parse to the integer `1`. |
| `"1.0"` vs `"1"` | Exhausted string yields revision `0`. |

If your solution is comparing characters, it is structurally wrong, not just buggy.

---

## 3. The Journey — 4 Bugs in the First Attempt

### First attempt

```java
while(v1 < version1.length() && v2 < version2.length()){
    int v1Int = (v1 - '0');
    int v2Int = (v2 - '0');
    if(version1.charAt(v1) != '.' && v1Int < v2Int){
        return -1;
    } else if(version1.charAt(v1) != '.' && v1Int > v2Int){
        return 1;
    } else{
        while(version1.charAt(v1) != '.'){ v1++; }
        while(version2.charAt(v2) != '.'){ v2++; }
    }
}
return 0;
```

---

### Bug 1 — subtracting `'0'` from the index

```java
int v1Int = (v1 - '0');   // v1 is the INDEX, not the char
```

`v1` is `0` on the first pass, `'0'` is `48`, so `v1Int` is `-48`. It never touches the string at all.

**Cause:** the name `v1` *reads* like it holds a version value. It actually holds a position.
**Lesson:** name index variables `i` / `j`. Bad names cause real bugs, not just ugly code.

---

### Bug 2 — comparing characters instead of revisions

Even with bug 1 fixed, `"1.10"` vs `"1.2"` compares `'1'` against `'2'` and returns `-1`. Correct answer is `1`.

This is the structural bug. See §2.

---

### Bug 3 — the skip loops

```java
while(version1.charAt(v1) != '.'){ v1++; }
```

Two separate failures:

1. **Infinite loop.** The loop stops *on* the dot. Next outer iteration, `charAt(v1) == '.'`, so the `if`/`else if` both fail, control falls to `else`, and the same `while` immediately sees `'.'` and does nothing. Pointer never moves. Hangs forever.
2. **Index out of bounds.** The last revision has no trailing dot, so the pointer walks past the end of the string.

**Fix for both:** bound-check inside the loop condition, then explicitly step over the dot after.

---

### Bug 4 — `&&` in the outer loop

```java
while(v1 < version1.length() && v2 < version2.length())
```

Stops as soon as *either* string runs out. So `"1.0.1"` vs `"1"` exits after one revision and wrongly returns `0`.

**Fix:** `||`. Keep going while *either* string still has characters.

Note this is what forces the "missing revision = 0" rule to be handled — you're now deliberately entering iterations where one side is already exhausted.

---

## 4. Final Solution

```java
class Solution {
    public int compareVersion(String version1, String version2) {
        int v1 = 0;
        int v2 = 0;

        // || not && : keep going while EITHER string has characters left
        while (v1 < version1.length() || v2 < version2.length()) {

            // Build one revision as an integer.
            // If this pointer is already past the end, the loop body
            // never runs and the revision stays 0 — this is exactly
            // the "missing revision = 0" rule, for free.
            int integer1 = 0;
            while (v1 < version1.length() && version1.charAt(v1) != '.') {
                integer1 = (integer1 * 10) + (version1.charAt(v1) - '0');
                v1++;
            }

            int integer2 = 0;
            while (v2 < version2.length() && version2.charAt(v2) != '.') {
                integer2 = (integer2 * 10) + (version2.charAt(v2) - '0');
                v2++;
            }

            if (integer1 < integer2) {
                return -1;
            }
            if (integer1 > integer2) {
                return 1;
            }

            // Each pointer now sits on a '.' or past the end.
            // Incrementing past the end is safe — every read is guarded.
            v1++;
            v2++;
        }
        return 0;
    }
}
```

**Result:** Beats 100%.

---

## 5. Why the Guards Are Load-Bearing

This is the subtle part worth recalling. Two things depend on the `v1 < version1.length() &&` guard:

1. **`&&` short-circuits left to right.** The bounds check *must* come before `charAt`. Swap the order and it throws.
2. **An exhausted pointer produces `0` with no special-casing.** The inner loop simply doesn't execute, `integer2` keeps its initialised value of `0`, and `"1.0"` vs `"1"` compares `0` against `0`. There is no `if (exhausted) rev = 0;` anywhere — the structure produces it.

Because of guard #1, the unconditional `v1++` at the bottom is also safe. `v1` may become `length() + 1`, but nothing ever reads at that index.

---

## 6. Digit Accumulation

```java
integer1 = (integer1 * 10) + (version1.charAt(v1) - '0');
```

Standard string→int idiom. Shift left one decimal place, add the new digit.

Parsing `"123"`:

| char | before | ×10 | + digit | after |
|---|---|---|---|---|
| `'1'` | 0 | 0 | +1 | 1 |
| `'2'` | 1 | 10 | +2 | 12 |
| `'3'` | 12 | 120 | +3 | 123 |

Leading zeros vanish automatically: `"007"` → `0` → `0` → `7`.

`charAt(i) - '0'` works because ASCII digits are contiguous: `'0'`=48 … `'9'`=57.

---

## 7. Dry Runs

### 7a. `"1.2"` vs `"1.10"` → expect `-1`

| Step | v1 | v2 | integer1 | integer2 | Action |
|---|---|---|---|---|---|
| enter loop | 0 | 0 | — | — | `0<3 \|\| 0<4` → true |
| parse | 1 | 1 | 1 | 1 | both stop at `'.'` |
| compare | 1 | 1 | 1 | 1 | equal → skip dots |
| | 2 | 2 | | | |
| enter loop | 2 | 2 | — | — | `2<3` → true |
| parse v1 | 3 | | 2 | | `'2'`, hits end |
| parse v2 | | 4 | | 10 | `'1'`→1, `'0'`→10 |
| compare | 3 | 4 | 2 | 10 | `2 < 10` → **return -1** ✓ |

Char-by-char would have compared `'2'` vs `'1'` and returned `1`. This is the case that proves §2.

---

### 7b. `"1.0.1"` vs `"1"` → expect `1`  *(unequal lengths)*

`version1.length() = 5`, `version2.length() = 1`

| Step | v1 | v2 | integer1 | integer2 | Note |
|---|---|---|---|---|---|
| enter | 0 | 0 | — | — | true |
| parse | 1 | 1 | 1 | 1 | equal |
| skip dots | 2 | 2 | | | v2 is now past its end |
| enter | 2 | 2 | — | — | `2<5` true, `2<1` false → `\|\|` saves us |
| parse v1 | 3 | | 0 | | reads `'0'`, stops at `'.'` |
| parse v2 | | 2 | | **0** | guard fails immediately, loop body never runs |
| compare | | | 0 | 0 | equal |
| skip | 4 | 3 | | | |
| enter | 4 | 3 | — | — | `4<5` true |
| parse v1 | 5 | | 1 | | reads `'1'`, hits end |
| parse v2 | | 3 | | **0** | still exhausted |
| compare | | | 1 | 0 | `1 > 0` → **return 1** ✓ |

Two things to notice: bug 4's `||` fix is what let us reach iteration 2, and `integer2 = 0` appeared with zero special-case code.

---

### 7c. `"1.0"` vs `"1"` → expect `0`  *(trailing zeros)*

| Step | v1 | v2 | integer1 | integer2 | Note |
|---|---|---|---|---|---|
| enter | 0 | 0 | — | — | true |
| parse | 1 | 1 | 1 | 1 | equal |
| skip | 2 | 2 | | | |
| enter | 2 | 2 | — | — | `2<3` true |
| parse v1 | 3 | | 0 | | reads `'0'` |
| parse v2 | | 2 | | 0 | exhausted → 0 |
| compare | | | 0 | 0 | equal |
| skip | 4 | 3 | | | v1 = 4 > length 3, harmless |
| loop test | 4 | 3 | | | `4<3` false, `3<1` false → exit |
| | | | | | **return 0** ✓ |

---

### 7d. `"1.01"` vs `"1.001"` → expect `0`  *(leading zeros)*

Second iteration: `"01"` accumulates `0 → 0*10+1 = 1`. `"001"` accumulates `0 → 0 → 1`. Both `1`. Equal → `0` ✓

No stripping logic needed — integer parsing handles it.

---

## 8. Complexity

### Time — **O(n + m)**, or **O(max(n, m))**

The nesting is misleading. Nested loops do *not* automatically mean multiplication.

The argument: **`v1` only ever increments and never resets.** Across the entire execution — inner loops and outer loop combined — `v1` can advance at most `version1.length()` times before the loop conditions all go false. Same for `v2`. Total work is bounded by the sum of the two string lengths.

This is **amortised analysis**, and it's the same argument used for:
- Two-pointer / sliding window problems
- Merging two sorted arrays

> **How to say it in an interview:** "Each pointer moves forward monotonically and never resets, so total pointer movement is bounded by n + m regardless of loop nesting."

### Space — **O(1)**

Four `int`s: `v1`, `v2`, `integer1`, `integer2`. No allocation, no growth with input.

---

## 9. The `split()` Alternative

```java
String[] a = version1.split("\\.");   // "\\." — split takes a REGEX, bare "." matches anything
String[] b = version2.split("\\.");
for (int i = 0; i < Math.max(a.length, b.length); i++) {
    int x = i < a.length ? Integer.parseInt(a[i]) : 0;
    int y = i < b.length ? Integer.parseInt(b[i]) : 0;
    if (x != y) return x < y ? -1 : 1;
}
return 0;
```

| | Two-pointer | `split()` |
|---|---|---|
| Time | O(n + m) | O(n + m) |
| **Space** | **O(1)** | **O(n + m)** — two arrays + every substring |
| Lines | ~25 | ~9 |
| Regex gotcha | none | `"\\."` — a bare `"."` matches every char and returns an empty array |

`split()` is fine for a first pass and shows you know the library. The two-pointer version is the one to have in your fingers — O(1) space is the reason it exists.

---

## 10. Assumptions to Name Out Loud

```java
integer1 = (integer1 * 10) + digit;   // can this overflow?
```

Yes — in general. It's safe here **only because the constraints promise every revision fits in a 32-bit int**. Without that guarantee you'd need `long`, or compare digit-strings after stripping leading zeros.

Also worth knowing: `Integer.compare(a, b)` would shorten the comparison, but its contract only guarantees a *negative / zero / positive* return, not exactly `-1 / 0 / 1`. OpenJDK does return `-1/0/1` so it passes LeetCode, but the explicit ternary doesn't rely on an implementation detail.

> Interviewers reward naming the assumption you're leaning on, even when the code is correct.

---

## 11. Recall Checklist

Before rereading the code, try to answer these:

1. Why `||` and not `&&` in the outer loop condition?
2. Where does "missing revision = 0" get implemented? *(Trick question — nowhere explicitly.)*
3. Why must the bounds check come **before** `charAt` in the `&&`?
4. Why is the unconditional `v1++` at the bottom safe even when `v1` is already at `length()`?
5. Why is the time complexity O(n + m) and not O(n × m)?
6. What breaks if you compare characters instead of parsed integers? Give the test case.
7. Why `split("\\.")` and not `split(".")`?

---

## 12. Transferable Pattern

**Two pointers over two sequences of unequal length, where an exhausted sequence contributes a default value.**

Same shape appears in:
- **LC 2 / LC 445 — Add Two Numbers:** loop while `l1 != null || l2 != null`, exhausted list contributes digit `0`.
- **LC 88 — Merge Sorted Array:** one array runs out first, drain the other.
- **LC 21 — Merge Two Sorted Lists:** same `||` structure.
- **LC 415 — Add Strings:** identical `||` + default-`0` idiom.

The reusable move: **let the guarded loop naturally produce the default instead of writing a special case.** Fewer branches, fewer bugs, and the correctness argument gets shorter.