# LC 25 — Reverse Nodes in k-Group

**Difficulty:** Hard · **Sheets:** NeetCode 150 | Striver SDE | Love Babbar 450
**Pattern:** In-place segment reversal with boundary re-linking

---

## 1. Problem

Given a linked list, reverse the nodes **k at a time** and return the modified list. Nodes in a
final group of fewer than `k` stay **as they are**.

```
1→2→3→4→5, k=2  →  2→1→4→3→5
1→2→3→4→5, k=3  →  3→2→1→4→5
```

**Follow-up:** solve it in **O(1) extra space**.

---

## 2. Prerequisite — reversing a segment correctly

```java
private ListNode reverseLL(ListNode head, int k) {
    ListNode prev = null;
    ListNode curr = head;
    ListNode next = null;

    while (curr != null && k > 0) {
        next = curr.next;      // save — curr.next is about to be destroyed
        curr.next = prev;      // flip
        prev = curr;           // advance prev
        curr = next;           // advance curr
        k--;
    }
    return prev;               // prev = new head of the reversed portion
}
```

### ⚠️ Two classic bugs in this loop

**`curr = curr.next` instead of `curr = next`.** `curr.next` was overwritten two lines earlier, so
it now points at `prev`. Dry run on `1→2→3`:

| step | next | `curr.next = prev` | prev | `curr = curr.next` |
|---|---|---|---|---|
| start | – | – | null | 1 |
| iter 1 | 2 | `1.next = null` | 1 | `curr = 1.next = null` ← dead |

The loop exits after one iteration and returns just `1`.

> **The whole point of the `next` temp is that `curr.next` is destroyed mid-iteration.** If you're
> not reading from `next`, the temp is pointless.

**Guarding `curr.next != null` instead of `curr != null`.** On the final node of the list,
`curr.next` is already null, so the loop exits and that node never gets flipped. Test `curr`.

(`while (k > 0)` alone also works — but only because the caller guaranteed `k` nodes exist. That's
an assumption living in a different method; `curr != null` makes the helper safe on its own.)

### The invariants to memorise

After the loop:
- **`prev`** = new head of the reversed portion
- **`curr`** = the first node *past* the reversed portion
- **the original `head` node = the new tail**

That third one is the key to the entire optimisation.

---

## 3. Approach A — split into groups, collect, stitch

### Idea

1. **Split & reverse:** chop the list into chunks of `k`, reverse each in isolation. Each reversed
   chunk is a clean, null-terminated mini-list.
2. **Stitch:** walk the chunks in order and link tail-of-previous → head-of-current.

Storing chunk heads in an `ArrayList` means you never hold complicated pointers across iterations —
the list does the bookkeeping.

```java
public ListNode reverseKGroup(ListNode head, int k) {
    int size = getSize(head);
    List<ListNode> chunks = new ArrayList<>();

    // PHASE 1: split + reverse
    while (size >= k) {
        ListNode temp = head;
        for (int i = 0; i < k; i++) temp = temp.next;   // start of next group

        chunks.add(reverseLL(head, k));
        head = temp;
        size -= k;
    }
    if (head != null) chunks.add(head);   // leftover chunk, unreversed

    // PHASE 2: stitch
    ListNode prevTail = null;
    for (ListNode node : chunks) {
        ListNode tail = node;
        while (tail.next != null) tail = tail.next;   // walk to this chunk's tail

        if (prevTail == null) head = node;            // first chunk becomes the head
        else prevTail.next = node;

        prevTail = tail;
    }
    return head;
}
```

### ⚠️ The stitching bug

```java
tail.next = prevTail;   // 🚩 backwards
prevTail = tail;
```

This links `tail(i+1).next = tail(i)` — **pointing backwards** — and clobbers each chunk's forward
pointer. On `[1,2,3,4,5], k=2` with chunks `[2→1]`, `[4→3]`, `[5]`:

| i | head | tail | did | result |
|---|---|---|---|---|
| 0 | 2 | 1 | `1.next = null` | `2→1` |
| 1 | 4 | 3 | `3.next = 1` | `4→3→1` — node 2 orphaned |
| 2 | 5 | 5 | `5.next = 3` | `5→3→1` |

**Two lessons:**
1. Write link assignments in the direction data flows: `previousTail.next = currentHead`. Say it
   aloud before typing.
2. Do the **link** before computing the current chunk's tail, or you clobber the pointer you need.

### ⚠️ The unconditional `chunks.add(head)`

If `head` is null (when `n % k == 0`) you add null to the list. It may pass anyway — a `findTail`
written `tail = node` before its loop returns null for null input, and `prevTail.next = null`
happens to be the correct terminator. **That's accidental correctness**, indistinguishable from
correct-by-design to the judge. Guard it: `if (head != null)`.

### Complexity

| | | why |
|---|---|---|
| **Time** | O(n) | but with a large constant — ~4 passes: `getSize`, the reversals, the tail walks, the k-step lookaheads |
| **Space** | **O(n/k)** | the `ArrayList` holds one pointer per group. For `k=1` that's O(n) — fails the follow-up |

### Deriving the O(n) carefully

```
getSize                     O(n)
outer loop iterations       n/k
  ├─ k-step lookahead       O(k)
  └─ reverseLL              O(k)          = O(n/k) × O(2k) = O(n)
stitch loop iterations      n/k
  └─ tail walk              O(k)          = O(n/k) × O(k)  = O(n)
                                          ───────────────────────
total                                     O(n)
```

The `n/k × k` cancellation is the thing to recognise on sight.

---

## 4. Optimisation 1 — remove `getSize()`

`getSize` is a whole pass to answer a **local** question: *"are there at least k nodes ahead of me
right now?"*

### ⚠️ The ambiguity trap

```java
ListNode temp = head;
for (int i = 0; i < k && temp != null; i++) temp = temp.next;
if (temp == null) break;   // 🚩 WRONG
```

`temp == null` means **either**:
- fewer than `k` nodes remain (should break), **or**
- exactly `k` remain and you legitimately landed one-past-the-end (should reverse!).

This silently drops the last full group.

### The fix — walk `k-1` steps, land *on* the kth node

```java
ListNode kth = head;
for (int i = 0; i < k - 1 && kth != null; i++) kth = kth.next;
if (kth == null) break;            // unambiguous: fewer than k nodes left

ListNode nextGroup = kth.next;     // may be null — just means this was the last group
```

> **General rule:** in a lookahead, land **on** the boundary node, not past it. "Walked k steps and
> hit null" ≠ "fewer than k remain."

### On `while (true)`

| verdict | condition |
|---|---|
| ✅ acceptable | the break is at the **top** of the body, and there is exactly **one** break |
| 🚩 smell | multiple scattered breaks, or a break buried in nesting |

The alternative — condition in the header — forces you to duplicate the lookahead once before the
loop and once at the bottom, which is its own bug source. Java has no do-while-with-condition-in-
the-middle, so you pick your poison.

---

## 5. Optimisation 2 — remove the `ArrayList` (O(1) space)

### Idea

Why is the list needed at all? Because you produce *all* the groups first, *then* connect them.
Connect each group **the moment you create it** and there's nothing to store.

You need three facts at the end of each reversal — **all three are already in your hands, free:**

| I need | Where it comes from | Cost |
|---|---|---|
| new head of the group | `prev` after the reverse loop | free |
| new **tail** of the group | the node you passed *in* — the original `groupHead` | free (save it first) |
| start of the next group | `curr` after the reverse loop | free |

Approach A paid for the tail with a `while (tail.next != null)` walk and for the next-group pointer
with a k-step lookahead. **Both were already available.**

> **This table is the entire optimisation.** Everything else is mechanics.

### Two tricks that make it clean

**Dummy node.** `ListNode dummy = new ListNode(0, head)` puts a fake node in front. Now
`prevTail.next` is *always* the right place to attach the next group, including the first. The
`if (prevTail == null) head = node;` special case vanishes and `dummy.next` tracks the real head
throughout.

> 🔑 **Not a k-group trick.** Use a dummy in *any* linked-list problem where the head might change —
> and reach for it **before** writing the loop, not as cleanup afterwards. A branchy loop has too
> many moving parts to trace, which is how people end up editing lines to see what sticks.
>
> Applies to: **19, 21, 24, 25, 61, 82, 83, 86, 92, 203.**

**Seed `prev` with the continuation.** Starting `prev = nextGroup` rather than `prev = null` makes
the reversed group come out **already attached** to the untouched remainder. No dangling null to
patch afterwards.

---

## 6. Version B — dummy + `getSize`, reversal inlined

```java
public ListNode reverseKGroup(ListNode head, int k) {
    int size = getSize(head);

    ListNode dummy = new ListNode(0, head);
    ListNode prevTail = dummy;

    while (size >= k) {
        ListNode groupHead = prevTail.next;   // ← becomes the group's TAIL after reversal
        ListNode prev = null;
        ListNode curr = groupHead;

        for (int i = 0; i < k; i++) {
            ListNode next = curr.next;
            curr.next = prev;
            prev = curr;
            curr = next;
        }
        // now: prev = new head of group, curr = start of next group

        prevTail.next  = prev;      // link the previous group forward into this one
        groupHead.next = curr;      // link this group to the remainder
        prevTail = groupHead;       // groupHead is now this group's tail

        size -= k;
    }
    return dummy.next;
}
```

The leftover chunk needs **no special handling** — `groupHead.next = curr` already attached it
during the last full group's iteration.

### Version B′ — keeping `reverseLL` as a helper

A method returns one value, but the reversal produces two useful things (the new head and the
next-group start). Options, in order of preference:

1. **Walk k steps in the caller before reversing.** Keeps the helper, costs an extra k hops per
   group. O(n) unchanged. Safe because `size >= k` already guaranteed the nodes exist.
2. **A small private static class or record** with named `head` and `next` fields.
3. **Return `ListNode[]`.** Works, but an interviewer will ask what index 1 means.
4. **Inline it** (B and C). Simplest pointer-wise, spends modularity. Fine on a whiteboard.

```java
public ListNode reverseKGroup(ListNode head, int k) {
    int size = getSize(head);

    ListNode dummy = new ListNode(-1);
    ListNode prevTail = dummy;
    ListNode nextStart = head;
    ListNode groupStart = head;

    while (size >= k) {
        for (int i = 0; i < k; i++) nextStart = nextStart.next;   // start of next group

        prevTail.next = reverseLL(groupStart, k);   // prev tail → this group's new head
        prevTail = groupStart;                      // groupStart was the head → now the tail

        groupStart = nextStart;
        size -= k;
    }

    prevTail.next = nextStart;   // attach leftover; null when n % k == 0
    return dummy.next;
}
```

**Why the final line needs no guard.** When `n % k == 0`, `nextStart` is null — and assigning null
is exactly the terminator the list needs. When a leftover exists, `nextStart` points at it. Both
cases, one line, no branch.

> **That is the *good* version of "it works either way":** you can state why both branches are
> correct. Contrast with the `chunks.add(head)` case in §3, where it works and you can't.

> ⚠️ `nextStart` and `groupStart` are always equal at the point of use, because the last statement
> in the body assigns one to the other. **When two variables are always equal, one is redundant** —
> and you may be reading the wrong one by luck. Computing the lookahead from `groupStart` inside
> the loop removes the second variable entirely.

---

## 7. Version C — final: dummy, no `getSize`, O(1) space

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

> The lookahead starts from `prevTail` (not `head`) and walks a full `k` steps — because `prevTail`
> sits one *before* the group, so k steps lands exactly **on** the kth node. Same unambiguity as the
> `k-1` version in §4, measured from a different origin.

---

## 8. Common pitfalls

- **Reintroducing the tail walk.** The group's tail is free: it's the node you passed into the
  reversal. If you're walking a list to find something, ask whether you already had it.
- **Hand-rolling the head special case** (`if (prevTail == null) head = node;`). Deletable in every
  version via the dummy node.
- **Advance overshoot.** After a line rewires `curr.next`, re-read what it points at. A two-hop
  advance becomes a one-hop once a link has changed.
- **Forgetting to advance at all.** Calling the reversal on the same node every iteration terminates
  (because `size -= k`) while silently reversing the same prefix repeatedly.
- **Returning `head` instead of the new head.** After reversing the first group, the original head
  node is the *tail* of that group.
- **Ambiguous lookahead null check** (§4).
- **Leftover group unwired.** `prevTail.next` still points into the old chain, so the tail is either
  lost or you keep a stale link.

---

## 9. Dry runs

### Version C, `1→2→3→4→5`, k = 2

Initial: `dummy→1→2→3→4→5`, `prevTail = dummy`

**Iteration 1**
```
lookahead: kth = dummy →(1) 1 →(2) 2      kth = node 2 ✅
nextGroup = 2.next = 3
groupHead = prevTail.next = 1

reverse loop, prev = 3, curr = 1:
  curr=1 ≠ 3 : nxt=2, 1.next=3, prev=1, curr=2     →  1→3→4→5
  curr=2 ≠ 3 : nxt=3, 2.next=1, prev=2, curr=3     →  2→1→3→4→5
  curr=3 == nextGroup → stop

prevTail.next = kth     → dummy.next = 2
prevTail = groupHead    → prevTail = node 1
```
State: `dummy→2→1→3→4→5`, `prevTail = 1`

**Iteration 2**
```
lookahead: kth = 1 →(1) 3 →(2) 4          kth = node 4 ✅
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

**Iteration 3**
```
lookahead: kth = 3 →(1) 5 →(2) null       kth = null → BREAK
```

Return `dummy.next` = **`2→1→4→3→5`** ✅ — node 5 never touched, as required.

### Version B′, `1→2→3→4→5→6`, k = 3

| iter | size | groupStart (→tail) | nextStart | new group head | after re-link |
|---|---|---|---|---|---|
| start | 6 | 1 | 1 | – | `dummy→1→2→3→4→5→6` |
| 1 | 6≥3 ✅ | 1 | walks to 4 | 3 | `dummy→3→2→1→…`, `prevTail=1`, `groupStart=4` |
| 2 | 3≥3 ✅ | 4 | walks to null | 6 | `1.next=6` → `dummy→3→2→1→6→5→4`, `prevTail=4` |
| exit | 0 | – | null | – | `prevTail.next = null` → `4.next = null` ✅ |

Result: `3→2→1→6→5→4`.

---

## 10. Complexity comparison

| Version | Time | Space | Passes | Notes |
|---|---|---|---|---|
| A1 — ArrayList + getSize | O(n) | **O(n/k)** | ~4 | getSize + reversals + tail walks + lookaheads |
| A2 — ArrayList, no getSize | O(n) | O(n/k) | ~3 | dropped the counting pass |
| B′ — dummy + getSize + helper | O(n) | **O(1)** | ~3 | tail is free; lookahead still paid separately |
| B — dummy + getSize, inlined | O(n) | **O(1)** | 2 | tail *and* nextGroup both free |
| **C — dummy, no getSize** | **O(n)** | **O(1)** | **1** | lookahead fuses into the traversal |

All are O(n) asymptotically — the difference is the constant factor and meeting the O(1)-space
follow-up, which is what an interviewer pushes on.

### ⚠️ The recursion follow-up

A recursive solution recurses **once per group**, not once per node. Depth = **n/k** ⇒ **O(n/k)
stack space**. With `k = 1` that's O(n) frames, and on LC 25's constraints (n up to 5000) it can
genuinely stack-overflow.

> **The line to say:** *"The recursive version looks in-place because there's no array, but the call
> stack carries the same O(n/k) the ArrayList did. Only the iterative version actually satisfies the
> O(1)-space follow-up."*

---

## 11. Edge cases

| case | expected | C | B′ |
|---|---|---|---|
| `head == null` | null | ✅ lookahead breaks immediately | ✅ `getSize`=0, loop skipped, `dummy.next` null |
| `k == 1` | unchanged | ✅ each "group" reverses to itself | ✅ |
| `k == n` | whole list reversed | ✅ one iteration then break | ✅ |
| `k > n` | unchanged | ✅ first lookahead fails | ✅ `size >= k` false immediately |
| `n % k != 0` | leftover stays as-is | ✅ attached via the `prev = nextGroup` seed | ✅ final `prevTail.next = nextStart` |
| `n % k == 0` | no leftover | ✅ | ✅ final line assigns null = correct terminator |
| single node | unchanged | ✅ | ✅ |

---

## 12. Interview Q&A

**Q: Why does reversing a group give you its tail for free?**
The node you passed in was the group's head before reversal; reversal makes it the tail. Save it in
one line and you never need to walk for it.

**Q: Can you do it recursively?**
Yes, and it reads cleanly — but it's O(n/k) stack space, so it doesn't satisfy the follow-up.

**Q: Why the dummy node?**
So the first group needs no special case. `prevTail.next` is always the correct attachment point,
and `dummy.next` tracks the real head throughout.

**Q: What if the leftover group should also be reversed?**
Drop the `size >= k` / lookahead guard and reverse whatever remains. The re-link logic is unchanged.

**Q: How do you know there are k nodes left without counting the whole list?**
Walk k−1 steps from the group head (or k from `prevTail`) and check whether you landed on a real
node. Landing *on* the boundary rather than past it makes the null check unambiguous.

---

## 13. Pattern

> **Reversing a segment turns its head into its tail — so the join point for the next segment is
> already in your hand.**

That single fact eliminates both the tail walk and the storage: if the tail is free, each group can
be linked in the moment it's produced, and nothing needs collecting.

---

## 14. Takeaways

1. **`curr = next`, never `curr = curr.next`**, after mutating `curr.next`.
2. **Guard `curr`, not `curr.next`**, in a reversal loop.
3. **Post-reversal invariants:** `prev` = new head · `curr` = first node past the segment · **the
   original head = the new tail**.
4. **Dummy node whenever the head can change — write it first, not last.**
5. **Seed `prev` with the continuation** instead of `null` when reversing a sub-segment.
6. **Write link assignments in data-flow order:** `previousTail.next = currentHead`.
7. **If you're walking to find something, ask whether you already had it.**
8. **Land *on* the boundary node in a lookahead**, not past it.
9. **Two variables that are always equal = one variable.**
10. **An unconditional assignment is fine only if you can state why both cases are correct.**
11. **Recursion depth is per *group*, not per node** → O(n/k) stack, not O(1), not O(k).
12. **`while (true)` is fine** with a single break at the top of the body.

---

## 15. Related problems

| # | Problem | Shared idea |
|---|---|---|
| 206 | Reverse Linked List | the base sub-routine |
| 92 | Reverse Linked List II | reverse a sub-segment + dummy node |
| 24 | Swap Nodes in Pairs | this problem with `k = 2` — good warm-up |
| 19 | Remove Nth Node From End | dummy node + lookahead pointer |
| 61 | Rotate List | size + re-link |
| 143 | Reorder List | reverse half, then interleave |
| 234 | Palindrome Linked List | reverse half in place |
| 138 | Copy List with Random Pointer | in-place pointer weaving; same "the list is its own scratch space" idea |