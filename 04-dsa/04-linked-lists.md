# 🔗 Linked Lists — 25 Problems

> Linked list problems are almost entirely about **pointer discipline**. Three techniques cover nearly all of them: dummy heads, fast/slow pointers, and in-place reversal.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 The Three Techniques

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int x = 0, ListNode* n = nullptr) : val(x), next(n) {}
};
```

### 1. Dummy head — eliminates head-edge-case branching
```cpp
ListNode dummy(0, head);
ListNode* prev = &dummy;
// ... manipulate freely, head changes need no special handling ...
return dummy.next;                                 // ⭐ NOT `head` — it may have moved
```

### 2. Fast & slow pointers
```cpp
ListNode *slow = head, *fast = head;
while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
// slow is now at the middle (second middle for even length)
```

### 3. Iterative reversal
```cpp
ListNode *prev = nullptr, *cur = head;
while (cur) {
    ListNode* nxt = cur->next;   // SAVE
    cur->next = prev;            // REVERSE
    prev = cur; cur = nxt;       // ADVANCE
}
return prev;
```

```
   ⚠️ The four bugs that cause 90% of linked list failures:
   1. Forgetting to save `next` before overwriting it → lose the rest of the list
   2. Returning `head` instead of `dummy.next` when the head changed
   3. Not checking `fast && fast->next` → null dereference
   4. Off-by-one in "k-th from end" (use a gap of exactly k)
```

---

## A. Basics & Reversal

### 1. Reverse Linked List 🟢
> Reverse a singly linked list.

#### 💬 Think of it like this
Walk down the list flipping each arrow to point backwards.

The problem is that the moment you flip an arrow, you've destroyed your only route to the rest of the list. So the dance is always three steps, in this exact order:

1. **Save** where you were about to go
2. **Flip** the current arrow backwards
3. **Advance** both pointers forward

You need three pointers: `prev` (where the arrow should now point), `cur` (the node you're flipping), and `nxt` (the saved escape route).

#### 📊 Watching the arrows flip

```
   START      null    1 → 2 → 3 → null
                      ▲
                     cur
              prev = null

   ┌──────────────────────────────────────────────────────────────┐
   │ STEP 1                                                       │
   │   save:    nxt = 2                                           │
   │   flip:    null ← 1    2 → 3 → null                          │
   │   advance:      prev↑  cur↑                                  │
   ├──────────────────────────────────────────────────────────────┤
   │ STEP 2                                                       │
   │   save:    nxt = 3                                           │
   │   flip:    null ← 1 ← 2    3 → null                          │
   │   advance:          prev↑  cur↑                              │
   ├──────────────────────────────────────────────────────────────┤
   │ STEP 3                                                       │
   │   save:    nxt = null                                        │
   │   flip:    null ← 1 ← 2 ← 3    null                          │
   │   advance:              prev↑  cur↑                          │
   ├──────────────────────────────────────────────────────────────┤
   │ cur is null → loop ends.                                     │
   │ ⭐ RETURN prev — it's the new head (the old tail).            │
   └──────────────────────────────────────────────────────────────┘
```

⚠️ **The two mistakes:** forgetting to save `next` before flipping (you lose the rest of the list instantly), and returning `head` instead of `prev` (`head` is now the *tail*).

⭐ **Memorize this three-line pattern.** It appears inside reverse-in-k-groups, palindrome check, reorder list, and several others — usually as one phase of a larger algorithm.

```cpp
// Iterative — O(n) time, O(1) space  ⭐ preferred
ListNode* reverseList(ListNode* head) {
    ListNode *prev = nullptr, *cur = head;
    while (cur) {
        ListNode* nxt = cur->next;
        cur->next = prev;
        prev = cur; cur = nxt;
    }
    return prev;
}

// Recursive — O(n) time, O(n) stack
ListNode* reverseRec(ListNode* head) {
    if (!head || !head->next) return head;
    ListNode* newHead = reverseRec(head->next);
    head->next->next = head;                       // ⭐ the node ahead points back
    head->next = nullptr;
    return newHead;
}
```

---

### 2. Reverse Linked List II (positions m..n) 🟡
```cpp
ListNode* reverseBetween(ListNode* head, int m, int n) {
    ListNode dummy(0, head);
    ListNode* prev = &dummy;
    for (int i = 1; i < m; ++i) prev = prev->next;  // node before position m

    ListNode* cur = prev->next;
    for (int i = 0; i < n - m; ++i) {               // ⭐ head-insertion technique
        ListNode* move = cur->next;
        cur->next = move->next;
        move->next = prev->next;
        prev->next = move;
    }
    return dummy.next;
}
```
```
   Each iteration lifts the node after `cur` and inserts it right after `prev`:
   prev → [1] → [2] → [3] → ...
   prev → [2] → [1] → [3] → ...
   prev → [3] → [2] → [1] → ...
```

---

### 3. Reverse Nodes in k-Group 🔴
```cpp
ListNode* reverseKGroup(ListNode* head, int k) {
    // 1. Check that k nodes exist
    ListNode* node = head;
    for (int i = 0; i < k; ++i) { if (!node) return head; node = node->next; }

    // 2. Reverse this group; `node` is the head of the remainder
    ListNode *prev = nullptr, *cur = head;
    for (int i = 0; i < k; ++i) {
        ListNode* nxt = cur->next;
        cur->next = prev;
        prev = cur; cur = nxt;
    }
    // 3. Recurse on the rest and attach
    head->next = reverseKGroup(node, k);           // `head` is now the group's TAIL
    return prev;                                   // `prev` is the new group head
}
```
🎤 **Follow-up (O(1) space):** iterative version with a dummy head and a group-tail pointer.

---

### 4. Swap Nodes in Pairs 🟡
```cpp
ListNode* swapPairs(ListNode* head) {
    ListNode dummy(0, head);
    ListNode* prev = &dummy;
    while (prev->next && prev->next->next) {
        ListNode* a = prev->next;
        ListNode* b = a->next;
        a->next = b->next;
        b->next = a;
        prev->next = b;
        prev = a;
    }
    return dummy.next;
}
```

---

### 5. Palindrome Linked List 🟢
```cpp
bool isPalindrome(ListNode* head) {
    // 1. Find the middle
    ListNode *slow = head, *fast = head;
    while (fast->next && fast->next->next) { slow = slow->next; fast = fast->next->next; }

    // 2. Reverse the second half
    ListNode *prev = nullptr, *cur = slow->next;
    while (cur) { ListNode* n = cur->next; cur->next = prev; prev = cur; cur = n; }

    // 3. Compare
    ListNode *p = head, *q = prev;
    bool ok = true;
    while (q) { if (p->val != q->val) { ok = false; break; } p = p->next; q = q->next; }

    // 4. ⭐ Restore the list (good practice; interviewers ask)
    cur = prev; prev = nullptr;
    while (cur) { ListNode* n = cur->next; cur->next = prev; prev = cur; cur = n; }
    slow->next = prev;

    return ok;
}
```
**Complexity:** O(n) / O(1).

---

## B. Fast & Slow Pointers

### 6. Middle of the Linked List 🟢
```cpp
ListNode* middleNode(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
    return slow;                                   // SECOND middle when even
}
// For the FIRST middle when even: while (fast->next && fast->next->next)
```

---

### 7. Linked List Cycle 🟢
```cpp
bool hasCycle(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

---

### 8. Linked List Cycle II (find the entry) 🟡
```cpp
ListNode* detectCycle(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) {
            ListNode* p = head;
            while (p != slow) { p = p->next; slow = slow->next; }
            return p;
        }
    }
    return nullptr;
}
```
```
   PROOF:
   head ──a──▶ [entry] ──b──▶ [meet]
                  ▲              │
                  └──────c───────┘

   slow travelled a + b
   fast travelled a + b + c + b = 2(a + b)   →   a = c

   So walking `a` steps from head and `c` steps from meet
   arrives at the entry simultaneously.
```

---

### 9. Remove Nth Node From End 🟡
```cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0, head);
    ListNode *fast = &dummy, *slow = &dummy;
    for (int i = 0; i <= n; ++i) fast = fast->next;   // ⭐ gap of n+1
    while (fast) { fast = fast->next; slow = slow->next; }
    ListNode* del = slow->next;
    slow->next = del->next;
    delete del;
    return dummy.next;
}
```
**Key insight:** Advancing `fast` by `n+1` from the dummy leaves `slow` at the node *before* the target — exactly what's needed for deletion.

---

### 10. Happy Number (cycle detection on a function) 🟢
```cpp
int sq(int n) { int s = 0; while (n) { int d = n % 10; s += d * d; n /= 10; } return s; }
bool isHappy(int n) {
    int slow = n, fast = n;
    do { slow = sq(slow); fast = sq(sq(fast)); } while (slow != fast);
    return slow == 1;
}
```
**Key insight:** Floyd's algorithm works on any function iteration, not just linked lists.

---

### 11. Find the Duplicate Number 🟡
> Array of n+1 integers in `[1, n]`. Find the duplicate without modifying the array, O(1) space.

```cpp
int findDuplicate(vector<int>& a) {
    int slow = a[0], fast = a[0];
    do { slow = a[slow]; fast = a[a[fast]]; } while (slow != fast);
    slow = a[0];
    while (slow != fast) { slow = a[slow]; fast = a[fast]; }
    return slow;
}
```
**Key insight:** Treat `i → a[i]` as a linked list. Since values are in `[1,n]` and there are `n+1` of them, some value repeats — which means two indices point to the same node, forming a cycle whose entry is the duplicate. Brilliant reduction.

---

## C. Merging & Sorting

### 12. Merge Two Sorted Lists 🟢
```cpp
ListNode* mergeTwoLists(ListNode* a, ListNode* b) {
    ListNode dummy;
    ListNode* tail = &dummy;
    while (a && b) {
        if (a->val <= b->val) { tail->next = a; a = a->next; }
        else                  { tail->next = b; b = b->next; }
        tail = tail->next;
    }
    tail->next = a ? a : b;                        // attach the remainder
    return dummy.next;
}
```

---

### 13. Merge k Sorted Lists 🔴
```cpp
// Approach 1: min-heap — O(N log k)
ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto* l : lists) if (l) pq.push(l);

    ListNode dummy;
    ListNode* tail = &dummy;
    while (!pq.empty()) {
        ListNode* n = pq.top(); pq.pop();
        tail->next = n; tail = n;
        if (n->next) pq.push(n->next);
    }
    return dummy.next;
}

// Approach 2: divide and conquer — O(N log k), O(1) extra space  ⭐ often preferred
ListNode* mergeKLists2(vector<ListNode*>& lists) {
    if (lists.empty()) return nullptr;
    int n = lists.size();
    while (n > 1) {
        int k = (n + 1) / 2;
        for (int i = 0; i < n / 2; ++i) lists[i] = mergeTwoLists(lists[i], lists[i + k]);
        n = k;
    }
    return lists[0];
}
```
⚠️ **Naive sequential merging is O(N·k)** — merging list 1 into an accumulator re-traverses everything each time. Pairwise merging is O(N log k).

---

### 14. Sort List (merge sort) 🟡
```cpp
ListNode* sortList(ListNode* head) {
    if (!head || !head->next) return head;

    // Split at the middle
    ListNode *slow = head, *fast = head->next;     // ⭐ fast starts ahead
    while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
    ListNode* mid = slow->next;
    slow->next = nullptr;                          // cut

    return mergeTwoLists(sortList(head), sortList(mid));
}
```
**Complexity:** O(n log n) time, O(log n) stack.
**Key insight:** Merge sort is the natural choice for linked lists — no random access needed, and merging is O(1) space. `fast = head->next` guarantees `slow` lands on the *first* middle, so the split is always non-trivial and recursion terminates.

---

### 15. Insertion Sort List 🟡
```cpp
ListNode* insertionSortList(ListNode* head) {
    ListNode dummy;
    while (head) {
        ListNode* nxt = head->next;
        ListNode* p = &dummy;
        while (p->next && p->next->val < head->val) p = p->next;
        head->next = p->next;
        p->next = head;
        head = nxt;
    }
    return dummy.next;
}
```

---

## D. Structural Manipulation

### 16. Remove Duplicates from Sorted List 🟢
```cpp
ListNode* deleteDuplicates(ListNode* head) {
    ListNode* cur = head;
    while (cur && cur->next) {
        if (cur->val == cur->next->val) {
            ListNode* d = cur->next;
            cur->next = d->next;
            delete d;
        } else cur = cur->next;
    }
    return head;
}
```

---

### 17. Remove Duplicates from Sorted List II (delete all copies) 🟡
```cpp
ListNode* deleteDuplicates(ListNode* head) {
    ListNode dummy(0, head);
    ListNode* prev = &dummy;
    while (head) {
        if (head->next && head->val == head->next->val) {
            int v = head->val;
            while (head && head->val == v) {       // skip the ENTIRE run
                ListNode* d = head; head = head->next; delete d;
            }
            prev->next = head;                     // ⭐ don't advance prev
        } else {
            prev = head;
            head = head->next;
        }
    }
    return dummy.next;
}
```

---

### 18. Remove Linked List Elements 🟢
```cpp
ListNode* removeElements(ListNode* head, int val) {
    ListNode dummy(0, head);
    ListNode* prev = &dummy;
    while (prev->next) {
        if (prev->next->val == val) {
            ListNode* d = prev->next;
            prev->next = d->next;
            delete d;
        } else prev = prev->next;
    }
    return dummy.next;
}
```

---

### 19. Partition List 🟡
```cpp
ListNode* partition(ListNode* head, int x) {
    ListNode lessDummy, geDummy;
    ListNode *less = &lessDummy, *ge = &geDummy;
    while (head) {
        if (head->val < x) { less->next = head; less = head; }
        else               { ge->next = head;   ge = head; }
        head = head->next;
    }
    ge->next = nullptr;                            // ⭐ MUST terminate, or cycle
    less->next = geDummy.next;
    return lessDummy.next;
}
```
**Key insight:** Two dummy-headed lists, then splice. Forgetting `ge->next = nullptr` creates an infinite loop — a very common bug.

---

### 20. Odd Even Linked List 🟡
```cpp
ListNode* oddEvenList(ListNode* head) {
    if (!head || !head->next) return head;
    ListNode *odd = head, *even = head->next, *evenHead = even;
    while (even && even->next) {
        odd->next = even->next;   odd = odd->next;
        even->next = odd->next;   even = even->next;
    }
    odd->next = evenHead;
    return head;
}
```

---

### 21. Rotate List 🟡
```cpp
ListNode* rotateRight(ListNode* head, int k) {
    if (!head || !head->next || k == 0) return head;

    int n = 1;
    ListNode* tail = head;
    while (tail->next) { tail = tail->next; ++n; }
    tail->next = head;                             // ⭐ make it circular

    k %= n;
    int steps = n - k;                             // new tail position
    ListNode* newTail = head;
    for (int i = 1; i < steps; ++i) newTail = newTail->next;

    ListNode* newHead = newTail->next;
    newTail->next = nullptr;                       // break the circle
    return newHead;
}
```

---

### 22. Reorder List 🟡
> L0 → L1 → … → Ln  becomes  L0 → Ln → L1 → Ln-1 → …

```cpp
void reorderList(ListNode* head) {
    if (!head || !head->next) return;

    // 1. Find the middle
    ListNode *slow = head, *fast = head;
    while (fast->next && fast->next->next) { slow = slow->next; fast = fast->next->next; }

    // 2. Reverse the second half
    ListNode *prev = nullptr, *cur = slow->next;
    slow->next = nullptr;                          // ⭐ cut the list
    while (cur) { ListNode* n = cur->next; cur->next = prev; prev = cur; cur = n; }

    // 3. Interleave
    ListNode *p = head, *q = prev;
    while (q) {
        ListNode *pn = p->next, *qn = q->next;
        p->next = q;
        q->next = pn;
        p = pn; q = qn;
    }
}
```
**Key insight:** Three-phase composition — find middle, reverse, merge. This decomposition pattern recurs constantly.

---

## E. Arithmetic & Complex Structures

### 23. Add Two Numbers 🟡
```cpp
ListNode* addTwoNumbers(ListNode* a, ListNode* b) {
    ListNode dummy;
    ListNode* tail = &dummy;
    int carry = 0;
    while (a || b || carry) {
        int s = carry;
        if (a) { s += a->val; a = a->next; }
        if (b) { s += b->val; b = b->next; }
        carry = s / 10;
        tail->next = new ListNode(s % 10);
        tail = tail->next;
    }
    return dummy.next;
}
```
**Key insight:** The `|| carry` in the loop condition handles the final carry (999 + 1) without a special case.

---

### 24. Add Two Numbers II (most significant first) 🟡
```cpp
ListNode* addTwoNumbers(ListNode* a, ListNode* b) {
    stack<int> sa, sb;
    for (auto* p = a; p; p = p->next) sa.push(p->val);
    for (auto* p = b; p; p = p->next) sb.push(p->val);

    ListNode* head = nullptr;
    int carry = 0;
    while (!sa.empty() || !sb.empty() || carry) {
        int s = carry;
        if (!sa.empty()) { s += sa.top(); sa.pop(); }
        if (!sb.empty()) { s += sb.top(); sb.pop(); }
        carry = s / 10;
        head = new ListNode(s % 10, head);         // ⭐ build by prepending
    }
    return head;
}
```
**Key insight:** Stacks reverse without modifying the input. Prepending each new node builds the result in the correct order.

---

### 25. Copy List with Random Pointer 🟡
```cpp
// O(1) space — interleave, wire, split
Node* copyRandomList(Node* head) {
    if (!head) return nullptr;

    // 1. Interleave copies:  A → A' → B → B' → C → C'
    for (Node* p = head; p; ) {
        Node* copy = new Node(p->val);
        copy->next = p->next;
        p->next = copy;
        p = copy->next;
    }

    // 2. Wire random pointers using the interleaving
    for (Node* p = head; p; p = p->next->next)
        if (p->random) p->next->random = p->random->next;   // ⭐ the copy of random

    // 3. Split the two lists apart
    Node* newHead = head->next;
    for (Node* p = head; p; ) {
        Node* copy = p->next;
        p->next = copy->next;
        copy->next = p->next ? p->next->next : nullptr;
        p = p->next;
    }
    return newHead;
}
```
**Complexity:** O(n) / O(1) — vs O(n) space for the hash map version.
**Key insight:** The interleaving makes `original->next` the copy, so `original->random->next` is exactly the copy of the random target. No auxiliary map needed.

---

### Bonus: Flatten a Multilevel Doubly Linked List 🟡
```cpp
Node* flatten(Node* head) {
    for (Node* p = head; p; p = p->next) {
        if (!p->child) continue;
        Node* nxt = p->next;
        Node* child = p->child;

        p->next = child;
        child->prev = p;
        p->child = nullptr;

        Node* tail = child;
        while (tail->next) tail = tail->next;      // find the child list's tail

        tail->next = nxt;
        if (nxt) nxt->prev = tail;
    }
    return head;
}
```

---

### Bonus: LRU Cache
> See [Hashing §19](02-hashing.md#19-lru-cache-) — hash map + doubly linked list.

---

### Bonus: Intersection of Two Linked Lists 🟢
```cpp
ListNode* getIntersectionNode(ListNode* a, ListNode* b) {
    ListNode *p = a, *q = b;
    while (p != q) {
        p = p ? p->next : b;                       // ⭐ switch to the other list
        q = q ? q->next : a;
    }
    return p;                                      // meeting point, or nullptr
}
```
**Key insight:** Each pointer travels `lenA + lenB` total, so they arrive at the intersection simultaneously regardless of the length difference. If there's no intersection, both hit `nullptr` at the same time and the loop exits.

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║                  LINKED LISTS — PATTERN RECALL                    ║
╠═══════════════════════════════════════════════════════════════════╣
║ DUMMY HEAD whenever the head might change → return dummy.next     ║
║ FAST & SLOW → middle, cycle, k-th from end                        ║
║ REVERSAL → save next, point back, advance (3 lines, memorize)     ║
╠═══════════════════════════════════════════════════════════════════╣
║ "middle"              → slow/fast, fast starts at head or head->next║
║                          depending on which middle you want        ║
║ "cycle entry"         → Floyd, then restart one pointer at head   ║
║ "nth from end"        → gap of n+1 from a dummy                   ║
║ "reverse in groups"   → count first, reverse, recurse             ║
║ "reorder/palindrome"  → find mid + reverse half + merge           ║
║ "merge k lists"       → heap O(N log k) or pairwise divide&conquer║
║                          NEVER sequential (that's O(N·k))          ║
║ "sort a list"         → MERGE sort (no random access needed)      ║
║ "copy with random"    → interleave copies for O(1) space          ║
║ "two lists intersect" → switch lists at the end; equal distance   ║
║ "duplicate in array"  → treat i→a[i] as a list, Floyd's cycle     ║
╠═══════════════════════════════════════════════════════════════════╣
║ BUG CHECKLIST                                                     ║
║   □ saved `next` before overwriting?                              ║
║   □ returning dummy.next, not head?                               ║
║   □ checked `fast && fast->next` before ->next->next?             ║
║   □ terminated the tail with nullptr? (partition, split)          ║
║   □ handled 0-node and 1-node lists?                              ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Stacks & Queues →](05-stacks-queues.md)
