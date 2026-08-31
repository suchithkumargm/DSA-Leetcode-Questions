# LC 2 — Add Two Numbers

**Topic:** Linked List · Simulation · Carry propagation
**Difficulty:** Medium
**Pattern:** Digit-wise simulation with carry (same family as "Add Binary", "Plus One", "Multiply Strings")

---

## 1. Problem

Two non-empty linked lists, each node holds a single digit `0–9`.
Digits are stored in **reverse order** — the head is the *least significant* digit.
Add the two numbers, return the sum as a linked list in the same reverse order.

```
l1 = [2,4,3]   ->  342
l2 = [5,6,4]   ->  465
                   ----
                   807
answer = [7,0,8]
```

**Constraints that matter:**
- `1 <= list length <= 100`  → up to a **100-digit number**
- No leading zeros in the number itself, except the number 0 (i.e. `[0,0,1]` is valid — that's 100)

---

## 2. Intuition

The single most important observation is hidden in the words *"reverse order"*.

```
       3   4   2        <- normal reading of 342
list:  2 -> 4 -> 3      <- head is the ONES place
```

That is **exactly** the order you add numbers by hand on paper:
start at the rightmost column, add, write the digit, carry to the left.

```
    3 4 2
  + 4 6 5
  -------
  start here ^   move left, carrying
```

The list is already handed to you in the order you need to consume it.
So the answer to *"do I ever need the full number in a variable?"* is **no**.
You never need the number. You only ever need three things at a time:
`digit1 + digit2 + carry`.

> **Recall trigger:** reverse-order storage isn't an obstacle, it's a gift. It matches the direction addition naturally flows.

---

## 3. First approach (what I tried) — and why it failed

### The idea
1. Walk each list, build the actual number in a `long`
2. Add the two numbers
3. Convert the sum back into a linked list

```java
// FAILED APPROACH — kept for the lesson
private long extractNumFromLL(ListNode head){
    long num = 0;
    boolean hasLeadingZero = (head.val == 0);
    while(head != null){
        num = (num * 10) + head.val;
        head = head.next;
    }
    if(hasLeadingZero) return reverse(num) * 10;
    return reverse(num);
}
```

### Failure 1 — leading zeros are destroyed (correctness)

`num = num * 10 + val` silently swallows any zero that appears before the first
non-zero digit of the traversal. But in this problem those front-of-traversal
zeros are the **low-order digits of the real number**. They carry information.

Dry run on `l1 = [0,0,1]`, which represents **100**:

| step | node val | `num` after `num*10 + val` |
|------|----------|----------------------------|
| 1    | 0        | 0                          |
| 2    | 0        | 0                          |
| 3    | 1        | 1                          |

`num = 1`. Reverse → `1`. The `hasLeadingZero` patch multiplies by 10 → `10`.
**Expected 100. Got 10.**

The `* 10` hack assumes there is exactly *one* leading zero. Two zeros, three
zeros — it breaks. And no amount of patching fixes it, because:

> Once the information is gone from `num`, reversing afterwards cannot bring it back.

That's the real lesson. The bug isn't the missing `* 10`; it's that the
*representation* (`long`) can't hold what the list holds.

### Failure 2 — overflow (fatal, unfixable by patching)

Constraints allow **100 nodes** → a 100-digit number.

```
long  max  = 9,223,372,036,854,775,807   -> 19 digits
needed     = up to 100 digits
```

Even if every zero bug were fixed, the approach cannot pass. `BigInteger` would
technically compile and pass, but it sidesteps the entire point of the question —
the interviewer is testing pointer manipulation and carry handling, not library knowledge.

### The meta-lesson

> When a problem gives you data in a structure, ask what that structure is *for*
> before converting it to something more familiar. Converting LL → number → LL
> is doing work in both directions to arrive back where you started.

---

## 4. Correct approach — one-pass carry simulation

Walk both lists together. Maintain a single `carry` variable.

At each step:
```
currSum = (l1.val if exists) + (l2.val if exists) + carry
digit   = currSum % 10       -> append to result
carry   = currSum / 10       -> feeds the next step
```

Max possible `currSum` = `9 + 9 + 1 = 19`, so **carry is always 0 or 1**. Integer
division does the job; no special cases needed.

### Two dedicated edge cases

1. **Unequal lengths** — `[9,9,9] + [1]`. Treat the exhausted list as contributing `0`.
2. **Leftover carry after both lists end** — `[5] + [5]` → `[0,1]`, `[9,9] + [1]` → `[0,0,1]`.
   The result can be **one node longer** than the longer input.

Both are folded into one loop condition:

```java
while (temp1 != null || temp2 != null || carry != 0)
```

The `carry != 0` clause is what makes the loop run one extra time to emit the
final carry digit. Without it you need a trailing `if (carry != 0) {...}` block.

### Dummy node

Building a list front-to-back means the first node is a special case (nothing to
attach it to yet). A **dummy head** removes that branch entirely: append blindly
in the loop, return `dummy.next` at the end.

```
dummy(-1) -> [7] -> [0] -> [8]
   |         ^
  head    return head.next
```

---

## 5. Final solution

```java
class Solution {
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        ListNode head = new ListNode(-1);   // dummy
        ListNode temp = head;
        int carry = 0;
        ListNode temp1 = l1;
        ListNode temp2 = l2;

        while (temp1 != null || temp2 != null || carry != 0) {
            int currSum = carry;

            if (temp1 != null) {
                currSum += temp1.val;
                temp1 = temp1.next;
            }
            if (temp2 != null) {
                currSum += temp2.val;
                temp2 = temp2.next;
            }

            temp.next = new ListNode(currSum % 10);
            carry = currSum / 10;
            temp = temp.next;
        }

        return head.next;
    }
}
```

### Alternative: three-loop form (also correct)

Equally valid, just more verbose — merge loop, then drain `l1`, then drain `l2`,
then a trailing `if (carry != 0)`. The single-loop version is preferred because
the leftover carry is impossible to forget: it lives in the loop condition
rather than in a block you might omit.

---

## 6. Dry run — `l1 = [2,4,3]`, `l2 = [5,6,4]`

| iter | t1.val | t2.val | carry in | currSum | node appended | carry out |
|------|--------|--------|----------|---------|---------------|-----------|
| 1    | 2      | 5      | 0        | 7       | 7             | 0         |
| 2    | 4      | 6      | 0        | 10      | 0             | 1         |
| 3    | 3      | 4      | 1        | 8       | 8             | 0         |
| exit | null   | null   | 0        | —       | —             | —         |

Result: `[7,0,8]` ✓  (342 + 465 = 807)

### Dry run — the carry edge case, `l1 = [9,9]`, `l2 = [1]`

| iter | t1.val | t2.val | carry in | currSum | node appended | carry out |
|------|--------|--------|----------|---------|---------------|-----------|
| 1    | 9      | 1      | 0        | 10      | 0             | 1         |
| 2    | 9      | —      | 1        | 10      | 0             | 1         |
| 3    | —      | —      | 1        | 1       | 1             | 0         |

Result: `[0,0,1]` ✓  (99 + 1 = 100)
Note iteration 3 runs **only** because of the `carry != 0` clause.

---

## 7. Complexity

| | |
|---|---|
| **Time** | `O(max(m, n))` — single pass, each node visited once |
| **Space** | `O(max(m, n))` for the output list; `O(1)` auxiliary (just `carry` + a few pointers) |

If the interviewer asks about space, be precise: the output list isn't counted as
extra space in most conventions, so the honest answer is **O(1) auxiliary space**.

---

## 8. Interview talking points

- **Why not convert to a number?** Overflow at 100 digits, plus reverse-order
  storage makes the conversion lossy for trailing zeros. And it's unnecessary work.
- **How big can the carry get?** Never more than 1, since `9 + 9 + 1 = 19`.
- **Can the answer be longer than both inputs?** Yes, by exactly one node, when a
  final carry survives.
- **What if the digits were stored in normal order (LC 445)?** The carry now flows
  opposite to traversal direction. Options: reverse both lists, use two stacks, or
  recursion. Stacks is the usual expected answer when reversal is disallowed.
- **In-place variant?** You could write results into `l1`'s nodes instead of
  allocating, if mutation is permitted.

---

## 9. Pattern to carry forward

> **Merging two sequences with leftover state:**
> `while (a != null || b != null || state != 0)` with null-guarded reads inside.

This single condition simultaneously handles: unequal lengths, exhausted inputs,
and residual state. Shows up in:

- LC 2 — Add Two Numbers *(carry)*
- LC 67 — Add Binary *(carry, strings)*
- LC 21 — Merge Two Sorted Lists *(no state, but same null-guard shape)*
- LC 415 — Add Strings *(carry, no lists)*
- LC 445 — Add Two Numbers II *(carry, but reversed direction)*

## 10. Mistakes to not repeat

1. Don't reach for numeric conversion when the input is a digit sequence — check
   the length constraint first.
2. Patching a symptom (`* 10` for one leading zero) instead of questioning the
   representation is a sign the approach itself is wrong.
3. When building a list front-to-back, reach for the dummy node immediately.
4. Always test `[5] + [5]` and `[9,9] + [1]` on any carry problem before submitting.