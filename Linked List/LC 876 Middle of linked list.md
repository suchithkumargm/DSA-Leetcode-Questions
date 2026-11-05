# LC 876 — Middle of the Linked List

**Difficulty:** Easy · **Pattern:** Two-speed pointers — position by ratio, not by counting

---

## 1. Problem

Given the head of a singly linked list, return the **middle node**. If there are two middle nodes,
return the **second** one.

```
[1,2,3,4,5]    →  node 3           (odd → unambiguous)
[1,2,3,4,5,6]  →  node 4           (even → the SECOND middle, not 3)
```

Constraints: `1 ≤ n ≤ 100`. The list is never empty.

> ⚠️ **"The second one" is the whole specification.** It's one character of difference in the loop
> condition (§4), and several related problems want the *first* middle instead.

---

## 2. The idea

The obvious approach is two passes: count the nodes, then walk `n/2` steps. That works and is O(n).

The better one needs only one pass:

> ### 🔑 The fact
> **If one pointer moves twice as fast as another, then when the fast one reaches the end, the slow
> one is exactly halfway.** The ratio does the division — no counting, no length.

```
[1, 2, 3, 4, 5]

start:   slow=1  fast=1
step 1:  slow=2  fast=3
step 2:  slow=3  fast=5      ← fast.next is null, stop. slow is the middle ✅
```

This is the same two-speed pointer shape as Floyd's cycle detection (LC 141/142) with the meeting
check removed — there you race them to *collide*, here you race them to *position* one of them.

---

## 3. Solution

```java
class Solution {
    public ListNode middleNode(ListNode head) {
        ListNode slow = head;
        ListNode fast = head;

        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
        }
        return slow;
    }
}
```

**O(n) time, single pass · O(1) space.**

### Why the condition needs both checks

`fast` takes **two** hops per iteration, so both links must be confirmed to exist:

- `fast != null` — handles **even** lengths, where `fast` lands exactly past the end
- `fast.next != null` — handles **odd** lengths, where `fast` lands *on* the last node

⚠️ **Order matters.** `&&` short-circuits, so `fast != null` must come first. Reversed, you
dereference null.

`slow` needs no check — it can never pass `fast`.

---

## 4. ⚠️ First middle vs second middle — one condition apart

The two variants, which differ only in where the loop stops:

```java
while (fast != null && fast.next != null)              // → SECOND middle (this problem)
while (fast.next != null && fast.next.next != null)    // → FIRST middle
```

On `[1,2,3,4]`:

| condition | stops when | returns |
|---|---|---|
| `fast != null && fast.next != null` | `fast` runs past the end | **3** (second middle) |
| `fast.next != null && fast.next.next != null` | `fast` is on the last node | **2** (first middle) |

On an **odd** length both return the same node — the ambiguity only exists for even lengths, which
is why it's easy to write the wrong one and not notice.

> ### Why this matters beyond LC 876
> Several problems split a list at the middle and need the **first** middle so the two halves divide
> correctly:
>
> - **LC 234 (Palindrome Linked List)** — reverse the second half and compare. The wrong middle
    >   splits unevenly and the comparison fails.
> - **LC 143 (Reorder List)** — same split-and-reverse setup.
> - **LC 148 (Sort List)** — merge sort needs both halves non-empty, or the recursion never
    >   terminates on a 2-node list.
>
> **Know which one you've written**, and check it on a 4-element list before building anything on
> top of it.

---

## 5. Common pitfalls

**Only checking `fast != null`.** On an odd-length list `fast` lands on the last node, and
`fast.next.next` throws.

**Only checking `fast.next != null`.** On an even-length list `fast` becomes null, and the condition
itself throws on the next evaluation. (This is also the `do-while`-with-bottom-guards trap: a
condition checked after the dereference protects the *next* iteration, never the first.)

**Returning `fast`.** `fast` is at or past the end; `slow` holds the answer.

**Two passes when one suffices.** Counting then walking is correct but misses the point of the
problem.

**Writing the first-middle variant by accident** (§4) — invisible on odd lengths.

**Starting `fast` at `head.next`.** Shifts the result by one and reintroduces the null-guard
problem, same as in LC 141.

---

## 6. Dry runs

### Odd — `[1,2,3,4,5]`

| iter | slow | fast | condition next time |
|---|---|---|---|
| start | 1 | 1 | ok |
| 1 | 2 | 3 | `fast=3` ok, `fast.next=4` ok |
| 2 | 3 | 5 | `fast=5` ok, **`fast.next=null`** → exit |

Return **3** ✅

### Even — `[1,2,3,4,5,6]`

| iter | slow | fast | condition next time |
|---|---|---|---|
| start | 1 | 1 | ok |
| 1 | 2 | 3 | ok |
| 2 | 3 | 5 | ok |
| 3 | 4 | **null** | `fast != null` → exit |

Return **4** ✅ — the second middle, as specified.

Note the two exits are different: odd lengths stop on `fast.next == null`, even lengths on
`fast == null`. That's why both checks are needed.

---

## 7. Edge cases

| case | expected | ✓ |
|---|---|---|
| single node | that node | ✅ `fast.next` is null → loop never runs |
| two nodes | the **second** | ✅ one iteration, `fast` becomes null |
| `head == null` | not in constraints | the loop handles it — returns null without a guard |
| very long list | correct | ✅ one pass |

---

## 8. Interview Q&A

**Q: Why does this find the middle?**
`fast` covers twice the ground. When it has consumed the whole list, `slow` has consumed half.

**Q: Why both conditions?**
`fast` hops twice. `fast != null` catches even lengths; `fast.next != null` catches odd. Order
matters because `&&` short-circuits.

**Q: How would you return the *first* middle instead?**
Stop one step earlier: `while (fast.next != null && fast.next.next != null)`.

**Q: Could you just count and walk?**
Yes, O(n) either way — but two passes. The two-pointer version is one pass and generalises to the
whole family (cycle detection, nth-from-end, splitting a list).

**Q: What if you needed the node at 1/3 of the list?**
Same idea with a 3× pointer. The ratio sets the fraction.

**Q: How does this relate to LC 141?**
Identical loop with the meeting check removed. There you race two pointers to collide; here you race
them to position one of them.

---

## 9. Pattern

> **A pointer moving at k× speed partitions the list by 1/k — when the fast one finishes, the slow
> one is at the fraction you wanted.**

The transferable half: **two pointers at a fixed speed ratio compute a position without knowing the
length.** The same family covers cycle detection (LC 141/142), nth-from-end (LC 19, a fixed *gap*
rather than a ratio), and every split-the-list-in-half setup.

---

## 10. Takeaways

1. **Speed ratio replaces counting.** No length needed, one pass.
2. **Both null checks, `fast` first** — `fast` takes two hops and `&&` short-circuits.
3. **The two exits differ:** odd lengths stop on `fast.next == null`, even on `fast == null`.
4. **`while (fast != null && fast.next != null)` → second middle;
   `while (fast.next != null && fast.next.next != null)` → first middle.** Check which you need.
5. **Test on a 4-element list** — the variants are indistinguishable on odd lengths.
6. **Return `slow`, not `fast`.**
7. **Same loop as Floyd's**, minus the meeting check.