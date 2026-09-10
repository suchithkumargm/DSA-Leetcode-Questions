# LC 25 — Reverse Nodes in k-Group (Hard)

> **Problem:** Given a linked list, reverse the nodes k at a time and return the modified list.
> Nodes in a leftover group of size `< k` stay **as they are**.
> Follow-up: solve it in **O(1) extra space**.
>
> Example: `1→2→3→4→5`, `k=2` → `2→1→4→3→5`
> Example: `1→2→3→4→5`, `k=3` → `3→2→1→4→5`

---

## 0. Prerequisite — the reverse-a-linked-list bug

Before k-group, the sub-routine has to be right. This is the bug I hit:

```java
while (curr != null) {
    next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = curr.next;   // ❌ BUG
}
```

### Why it fails

`curr.next` was **overwritten** on line 3. By the time line 5 runs, `curr.next == prev`, not the original next node.

Dry run on `1→2→3`:

| step | next | curr.next = prev | prev | curr = curr.next |
|---|---|---|---|---|
| start | – | – | null | 1 |
| iter 1 | 2 | `1.next = null` | 1 | `curr = 1.next = null` ← dead |

Loop exits after one iteration. Returns just `1`.

### Fix

```java
curr = next;   // ✅ that's why we saved `next` in the first place
```

**Lesson to remember:** the whole point of the `next` temp variable is that `curr.next` is destroyed mid-iteration. If you're not reading from `next`, the temp is pointless.

### Correct version

```java
private ListNode reverseLL(ListNode head, int k) {
    ListNode prev = null;
    ListNode curr = head;
    ListNode next = null;      // null reads better than head — it's meaningless until assigned

    while (k > 0) {
        next = curr.next;      // save
        curr.next = prev;      // flip
        prev = curr;           // advance prev
        curr = next;           // advance curr
        k--;
    }
    return prev;               // prev = new head
}
```

**Invariant to memorise:** after the loop, `prev` = new head of the reversed portion, `curr` = the first node *past* the reversed portion, and the original `head` node = the new **tail**.

That third fact is the one everybody forgets, and it's the key to the whole optimisation later.

---

## 1. Approach 1 — Brute force: split into groups, collect, then stitch

### Intuition

Break the problem into two independent halves that are each easy to reason about:

1. **Split & reverse:** chop the list into chunks of `k`, reverse each chunk in isolation. Each reversed chunk is a clean, null-terminated mini-list.
2. **Stitch:** walk the collected chunks in order and re-link tail-of-previous → head-of-current.

Storing the chunk heads in an `ArrayList` means I never have to hold complicated pointers across iterations — the list does the bookkeeping.

### Code (final working version)

```java
class Solution {
    public ListNode reverseKGroup(ListNode head, int k) {
        int size = getSize(head);
        List<ListNode> list = new ArrayList<>();

        // ---- PHASE 1: split + reverse ----
        while (size >= k) {
            ListNode temp = head;
            for (int i = 0; i < k; i++) temp = temp.next;  // temp = start of next group

            list.add(reverseLL(head, k));
            head = temp;
            size -= k;
        }
        if (size > 0) list.add(head);   // leftover chunk, unreversed

        // ---- PHASE 2: stitch ----
        ListNode prev = null;           // tail of the previously-attached group
        for (ListNode node : list) {
            ListNode tail = node;
            while (tail.next != null) tail = tail.next;   // walk to this group's tail

            if (prev == null) head = node;   // first group → becomes overall head
            else prev.next = node;           // link forward

            prev = tail;
        }
        return head;
    }

    private int getSize(ListNode head) {
        int size = 0;
        while (head != null) { size++; head = head.next; }
        return size;
    }

    private ListNode reverseLL(ListNode head, int k) {
        ListNode prev = null, curr = head, next = null;
        while (k > 0) {
            next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
            k--;
        }
        return prev;
    }
}
```

### The stitching bug I hit first

My first stitch loop was:

```java
temp.next = prev;   // ❌ backwards
prev = temp;
```

This links `tail(i+1).next = tail(i)` — **pointing backwards**, and it also nukes the forward pointer of every group's tail.

Dry run of the broken version on `[1,2,3,4,5], k=2` → groups `[2→1]`, `[4→3]`, `[5]`:

| i | curr (head) | temp (tail) | did | result |
|---|---|---|---|---|
| 0 | 2 | 1 | `1.next = null` | `2→1` |
| 1 | 4 | 3 | `3.next = 1` | `4→3→1` — 2 is now orphaned |
| 2 | 5 | 5 | `5.next = 3` | `5→3→1` |

Returned head was still `2`, whose `.next` was set to null at i=0 and never touched again → output `[2]`.

**Two lessons:**
1. Write the assignment in the direction data flows: `previousTail.next = currentHead`. Say it out loud before typing it.
2. Do the **link** step *before* computing the current group's tail, or you'll clobber the pointer you need.

### Complexity

| | | why |
|---|---|---|
| **Time** | `O(n)` | but with a large constant — **3 full passes**: `getSize`, the reversals, and the tail-walks in the stitch loop. Plus the k-step lookahead per group ≈ a 4th partial pass. |
| **Space** | `O(n/k)` | the `ArrayList` holds one pointer per group. For `k=1` that's `O(n)` — fails the follow-up. |

LC result: **beats ~34%**. (Runtime percentiles on this problem are mostly JIT noise — the *structural* waste above is the real issue.)

---

## 2. Optimisation A — remove `getSize()` (the lookahead trick)

### Intuition

`getSize` is a whole pass just to answer a **local** question: "are there at least k nodes ahead of me right now?" Answer it locally instead by walking ahead and seeing if you fall off the end.

### The subtle trap ⚠️

The naive version walks `k` steps and checks for null:

```java
ListNode temp = head;
for (int i = 0; i < k && temp != null; i++) temp = temp.next;
if (temp == null) break;   // ❌ WRONG
```

`temp == null` is **ambiguous**. It means *either*:
- fewer than `k` nodes remain (should break), **or**
- exactly `k` remain and we legitimately landed one-past-the-end (should reverse!).

This silently drops the last full group.

### The fix — walk `k-1` steps, land **on** the kth node

```java
ListNode kth = head;
for (int i = 0; i < k - 1 && kth != null; i++) kth = kth.next;
if (kth == null) break;            // unambiguous: fewer than k nodes left

ListNode nextGroup = kth.next;     // may be null — that's fine, just means last group
```

Now `kth == null` has exactly one meaning. And `nextGroup` is read *after* confirming the group is full, so a null there just means "this was the last group" — the next iteration starts from null, the for-loop body never runs, and `break` fires immediately.

### Version A code

```java
public ListNode reverseKGroup(ListNode head, int k) {
    List<ListNode> list = new ArrayList<>();

    while (true) {
        ListNode kth = head;
        for (int i = 0; i < k - 1 && kth != null; i++) kth = kth.next;
        if (kth == null) break;

        ListNode nextGroup = kth.next;
        list.add(reverseLL(head, k));
        head = nextGroup;
    }
    if (head != null) list.add(head);

    ListNode prev = null;
    for (ListNode node : list) {
        ListNode tail = node;
        while (tail.next != null) tail = tail.next;
        if (prev == null) head = node;
        else prev.next = node;
        prev = tail;
    }
    return head;
}
```

Empty list check: `head == null` → `kth` starts null → for body never runs → break → `if (head != null)` skips → empty list → returns null. ✅

### Note on `while (true)`

It's fine **here**, and the rule generalises:

| verdict | condition |
|---|---|
| ✅ acceptable | break is at the **top** of the body, and there is exactly **one** break |
| 🚩 smell | multiple scattered breaks, or a break buried behind several levels of nesting |

The alternative (condition in the header) forces you to duplicate the lookahead call — once before the loop, once at the bottom — which is its own bug source. Java has no do-while-with-condition-in-the-middle, so you pick your poison.

---

## 3. Optimisation B — remove the `ArrayList` (O(1) space)

This is the bigger win, and the one worth internalising.

### Intuition

Why did I need the list at all? Because I produced all the groups first, *then* connected them. If I connect each group **the moment I create it**, there's nothing to store.

To do that I need three facts at the end of each reversal — and all three are already sitting in my hands, free:

| I need | Where it comes from | Cost |
|---|---|---|
| new head of the group | `prev` after the reverse loop | free |
| new **tail** of the group | the node I passed *in* — the original `groupHead` | free (just save it first) |
| start of the next group | `curr` after the reverse loop | free |

The old code paid for #2 with a `while (tail.next != null)` walk and #3 with a k-step lookahead. **Both were already available.**

### Two tricks that make it clean

**Trick 1 — the dummy node.**
`ListNode dummy = new ListNode(0, head)` gives a fake node in front of the list. Now `prevTail.next` is *always* the right place to attach the next group, including for the very first group. The `if (prev == null) head = node;` special case disappears entirely, and `dummy.next` tracks the real head throughout.

> 🔑 **Carry this forward.** The dummy node is not a k-group trick — use it in *any* linked-list problem where the head might change: Remove Nth Node from End (19), Remove Duplicates, Partition List, Merge Two Lists (21).

**Trick 2 — seed `prev` with `nextGroup` instead of `null`.**
In the reversal loop, if you start `prev = nextGroup` rather than `prev = null`, the reversed group comes out **already attached** to the untouched remainder. No dangling null to patch afterwards.

### Version B — keeping `getSize`

```java
public ListNode reverseKGroup(ListNode head, int k) {
    int size = getSize(head);

    ListNode dummy = new ListNode(0, head);
    ListNode prevTail = dummy;

    while (size >= k) {
        ListNode groupHead = prevTail.next;   // ← this node becomes the group's TAIL
        ListNode prev = null;
        ListNode curr = groupHead;

        for (int i = 0; i < k; i++) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        // now: prev = new head of group, curr = start of next group

        prevTail.next = prev;      // link the previous group forward into this one
        groupHead.next = curr;     // link this group to the remainder
        prevTail = groupHead;      // groupHead is now this group's tail

        size -= k;
    }
    return dummy.next;
}
```

The leftover chunk needs **no special handling** — `groupHead.next = curr` already attached it during the last full group's iteration, and the loop then exits on `size >= k`.

### Version C — final, no getSize + O(1) space

```java
public ListNode reverseKGroup(ListNode head, int k) {
    ListNode dummy = new ListNode(0, head);
    ListNode prevTail = dummy;

    while (true) {
        // 1. lookahead: find the kth node from prevTail
        ListNode kth = prevTail;
        for (int i = 0; i < k && kth != null; i++) kth = kth.next;
        if (kth == null) break;                  // fewer than k nodes remain

        ListNode nextGroup = kth.next;
        ListNode groupHead = prevTail.next;      // becomes the tail after reversal

        // 2. reverse [groupHead .. kth], seeding prev with nextGroup
        ListNode prev = nextGroup, curr = groupHead;
        while (curr != nextGroup) {
            ListNode nxt = curr.next;
            curr.next = prev;
            prev = curr;
            curr = nxt;
        }

        // 3. re-link
        prevTail.next = kth;         // kth is the new head of this group
        prevTail = groupHead;        // groupHead is the new tail
    }
    return dummy.next;
}
```

> Note the lookahead here starts from `prevTail` (not `head`) and walks a full `k` steps — because `prevTail` sits one *before* the group, so k steps lands exactly **on** the kth node. Same unambiguity as the `k-1` version in §2, just measured from a different origin.

### Why `reverseLL` had to be inlined

A Java method returns **one** value. The reversal produces **two** things I need: the new head (`prev`) and the next-group start (`curr`). The old code worked around this by walking k nodes separately to recompute `temp`. Inlining removes the workaround. (Alternative if you insist on a helper: return a small `ListNode[]` or a record — usually not worth it.)

---

## 4. Full dry run — Version C

Input `1→2→3→4→5`, `k = 2`.

Initial: `dummy→1→2→3→4→5`, `prevTail = dummy`

### Iteration 1

```
lookahead: kth = dummy →(1) 1 →(2) 2      kth = node 2  ✅ not null
nextGroup = 2.next = 3
groupHead = prevTail.next = 1

reverse loop, prev = 3, curr = 1:
  curr=1 ≠ 3 : nxt=2, 1.next=3, prev=1, curr=2      list: 1→3→4→5
  curr=2 ≠ 3 : nxt=3, 2.next=1, prev=2, curr=3      2→1→3→4→5
  curr=3 == nextGroup → stop

prevTail.next = kth     → dummy.next = 2
prevTail = groupHead    → prevTail = node 1
```
State: `dummy→2→1→3→4→5`, `prevTail = 1`

### Iteration 2

```
lookahead: kth = 1 →(1) 3 →(2) 4          kth = node 4  ✅
nextGroup = 4.next = 5
groupHead = prevTail.next = 3

reverse loop, prev = 5, curr = 3:
  curr=3 ≠ 5 : nxt=4, 3.next=5, prev=3, curr=4
  curr=4 ≠ 5 : nxt=5, 4.next=3, prev=4, curr=5
  curr=5 == nextGroup → stop

prevTail.next = 4       → 1.next = 4
prevTail = groupHead    → prevTail = node 3
```
State: `dummy→2→1→4→3→5`, `prevTail = 3`

### Iteration 3

```
lookahead: kth = 3 →(1) 5 →(2) null       kth = null  → BREAK
```

Return `dummy.next` = **`2→1→4→3→5`** ✅ — and node 5 was never touched, exactly as required.

### Compact trace table (Version B, same input)

| iter | groupHead (→tail) | prev (→head) | curr (→next grp) | list after |
|---|---|---|---|---|
| 1 | 1 | 2 | 3 | `2→1→3→4→5` |
| 2 | 3 | 4 | 5 | `2→1→4→3→5` |

---

## 5. Complexity comparison

| Version | Time | Space | Passes over data | Notes |
|---|---|---|---|---|
| A1 — ArrayList + getSize | O(n) | **O(n/k)** | ~4 | getSize + reverse + tail-walks + lookaheads |
| A2 — ArrayList, no getSize | O(n) | O(n/k) | ~3 | dropped the counting pass |
| B — dummy + getSize | O(n) | **O(1)** | 2 | tail & nextGroup now free |
| **C — dummy, no getSize** | **O(n)** | **O(1)** | **1** | lookahead fuses into the traversal |

All are O(n) asymptotically — the difference is entirely in the **constant factor** and in meeting the O(1)-space follow-up, which is what an interviewer will actually push on.

Space note: recursive solutions to this problem look elegant but are **O(n/k) stack space**, so they don't satisfy the follow-up either. Iterative is the answer to give.

---

## 6. Edge cases checklist

| case | expected | does C handle it? |
|---|---|---|
| `head == null` | null | ✅ lookahead breaks immediately, returns `dummy.next` = null |
| `k == 1` | unchanged list | ✅ each "group" reverses to itself |
| `k == n` | whole list reversed | ✅ one iteration, then break |
| `k > n` | unchanged list | ✅ first lookahead fails, breaks |
| `n % k != 0` | leftover stays as-is | ✅ leftover never entered; already attached via `prev = nextGroup` seed |
| single node | unchanged | ✅ |

---

## 7. Takeaways for future problems

1. **`curr = next`, never `curr = curr.next`** after you've mutated `curr.next`. The temp exists for a reason.
2. **Post-reversal invariants** (drill these): `prev` = new head · `curr` = first node past the segment · **the original head node = the new tail**.
3. **Dummy node** whenever the head can change. Kills the "is this the first one?" branch. → LC 19, 21, 61, 82, 83, 86, 92, 203.
4. **Seed `prev` with the continuation** instead of `null` when reversing a *sub-segment*, so it comes out pre-attached.
5. **Write link assignments in data-flow order**: `previousTail.next = currentHead`. Say it aloud before typing.
6. **If you're walking the list to find something, ask whether you already had it.** The tail and the next-group pointer were both free — I paid for them twice.
7. **Beware ambiguous null checks in lookaheads.** "Walked k steps and hit null" ≠ "fewer than k remain". Land *on* the boundary node, not past it.
8. `while (true)` is fine with a single break at the top of the body.

---

## 8. Related problems (same muscles)

| # | Problem | Shared idea |
|---|---|---|
| 206 | Reverse Linked List | the base sub-routine |
| 92 | Reverse Linked List II | reverse a sub-segment + dummy node |
| 24 | Swap Nodes in Pairs | this problem with `k = 2` (do it as a warm-up) |
| 19 | Remove Nth Node From End | dummy node + lookahead pointer |
| 61 | Rotate List | size + re-link |
| 143 | Reorder List | reverse half, then interleave |
| 234 | Palindrome Linked List | reverse half in place |

---

*Solved: LC 25, Hard. Accepted with Approach A1 (beats ~34%). Optimisations B and C documented above for a later revisit.*