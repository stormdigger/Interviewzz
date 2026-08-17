# 🧩 DSA Patterns & Foundations

> There are not 400 problems. There are ~20 patterns and 400 dressings of them. This book is the pattern library — read it first, return to it whenever you're stuck.

---

## 📑 Table of Contents

1. [How to Solve Any Problem](#1-how-to-solve-any-problem)
2. [Complexity Analysis](#2-complexity-analysis)
3. [The Pattern Decision Tree](#3-pattern-decision-tree)
4. [The 20 Core Patterns](#4-the-20-core-patterns)
5. [C++ Toolkit for Interviews](#5-cpp-toolkit)
6. [Complexity Cheat Table](#6-complexity-table)
7. [Edge Case Checklist](#7-edge-cases)
8. [Common Bugs](#8-common-bugs)

---

## 1. How to Solve Any Problem

```
   ┌─────────────────────────────────────────────────────────────┐
   │ 1. CLARIFY (2 min)                                          │
   │    • Input range? Sorted? Duplicates? Negatives? Empty?      │
   │    • Output: index or value? All answers or one? Any order?  │
   │    • Constraints → they TELL you the target complexity       │
   ├─────────────────────────────────────────────────────────────┤
   │ 2. EXAMPLE (2 min)                                          │
   │    Work a small example BY HAND. Then an edge case.          │
   │    Half of all "hard" problems become obvious here.          │
   ├─────────────────────────────────────────────────────────────┤
   │ 3. BRUTE FORCE (2 min)                                      │
   │    State it out loud with its complexity. Never skip this —  │
   │    it proves you understand the problem and gives a baseline.│
   ├─────────────────────────────────────────────────────────────┤
   │ 4. OPTIMIZE (5 min)                                         │
   │    What is the brute force REDOING?                          │
   │    → recomputation      → memoize / DP                       │
   │    → repeated searching → hash map / sort / binary search    │
   │    → repeated scanning  → two pointers / sliding window      │
   │    → repeated min/max   → heap / monotonic stack             │
   ├─────────────────────────────────────────────────────────────┤
   │ 5. CODE (15 min)                                            │
   │    Talk while writing. Name things well. Handle edges.       │
   ├─────────────────────────────────────────────────────────────┤
   │ 6. TEST (5 min)                                             │
   │    Trace your example. Then: empty, single, all-same,        │
   │    max size, negative, overflow.                             │
   └─────────────────────────────────────────────────────────────┘
```

### Constraints → complexity (memorize this)

```
   n ≤ 10           → O(n!) or O(2ⁿ ·n) permutations/backtracking
   n ≤ 20-25        → O(2ⁿ) subsets, bitmask DP
   n ≤ 100          → O(n⁴) or O(n³)
   n ≤ 500          → O(n³) Floyd-Warshall, interval DP
   n ≤ 5,000        → O(n²) DP
   n ≤ 100,000      → O(n log n) sort, heap, binary search  ⭐ most common
   n ≤ 1,000,000    → O(n) or O(n log n)
   n ≤ 10⁸          → O(n) with a tiny constant, or O(log n)
   n > 10⁹          → O(log n) or O(1) — binary search on answer, math
```

🎤 **Use this out loud:** "n is up to 10⁵, so an O(n²) solution won't pass — I need O(n log n) or better. That suggests sorting, a heap, or binary search."

---

## 2. Complexity Analysis

### Big-O rules

```
   1. Drop constants:        O(2n)      → O(n)
   2. Drop lower terms:      O(n² + n)  → O(n²)
   3. Different inputs = different variables:  O(a + b), NOT O(n)
   4. Nested loops multiply; sequential loops add
   5. Recursion: (number of calls) × (work per call)
```

### Recursion complexity

```
   MASTER THEOREM   T(n) = a·T(n/b) + f(n)

   Compare f(n) with n^(log_b a):
     f smaller → T(n) = Θ(n^(log_b a))
     f equal   → T(n) = Θ(n^(log_b a) · log n)
     f larger  → T(n) = Θ(f(n))

   Examples:
     Binary search    T(n) = T(n/2) + O(1)     → O(log n)
     Merge sort       T(n) = 2T(n/2) + O(n)    → O(n log n)
     Karatsuba        T(n) = 3T(n/2) + O(n)    → O(n^1.585)
```

**Recursion tree method** — often faster than the theorem:

```
   Fibonacci naive:  fib(n) = fib(n-1) + fib(n-2)

                    fib(5)
                  /        \
            fib(4)          fib(3)
           /     \          /    \
      fib(3)   fib(2)   fib(2)  fib(1)
       ...

   Branching factor 2, depth n  →  O(2ⁿ)
   With memoization: n distinct states, O(1) each → O(n)
```

### Amortized analysis

```
   Dynamic array push_back:
     Most pushes: O(1)
     Occasionally: O(n) to reallocate and copy

   But doubling means n pushes cost 1+2+4+...+n ≈ 2n total
   → AMORTIZED O(1) per push
```

### Space complexity — don't forget the stack

```
   Recursion depth d → O(d) stack space
   
   DFS on a path graph of n nodes → O(n) stack (can overflow!)
   Balanced BST recursion         → O(log n)
```

---

## 3. Pattern Decision Tree

```
                        Read the problem
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
   Is input SORTED?      Is it a TREE/GRAPH?    Are we OPTIMIZING
        │                      │                (max/min/count ways)?
        ▼                      ▼                      │
  ┌─────────────┐        ┌──────────────┐             ▼
  │two pointers │        │ BFS: shortest│      ┌──────────────┐
  │binary search│        │      levels  │      │ Overlapping  │
  └─────────────┘        │ DFS: paths,  │      │ subproblems? │
                         │      cycles  │      │  → DP        │
                         │ Union-Find:  │      │ Greedy works?│
                         │   connectivity│     │  → prove it! │
                         │ Topo: ordering│     └──────────────┘
                         │ Dijkstra: wts │
                         └──────────────┘

        ▼                      ▼                      ▼
   CONTIGUOUS            TOP-K / K-th            SUBSETS /
   subarray/substring?   / streaming median?     PERMUTATIONS /
        │                      │                 COMBINATIONS?
        ▼                      ▼                      │
   sliding window          HEAP (or                   ▼
   (+ hash map)            quickselect)          BACKTRACKING

        ▼                      ▼                      ▼
   "NEXT GREATER"        INTERVALS?              LINKED LIST
   "PREVIOUS SMALLER"          │                 cycle/middle?
   histogram/rain?             ▼                      │
        │                 sort by start,              ▼
        ▼                 then merge/sweep       FAST & SLOW
   MONOTONIC STACK                               POINTERS

        ▼                      ▼                      ▼
   PREFIX SUM / DIFF     STRING MATCHING         "MINIMUM x SUCH
   subarray sums?        prefix/autocomplete?    THAT f(x) WORKS"?
        │                      │                      │
        ▼                      ▼                      ▼
   prefix[] + hashmap        TRIE              BINARY SEARCH
                                               ON THE ANSWER
```

---

## 4. The 20 Core Patterns

### Pattern 1 — Two Pointers

**When:** sorted array, pair/triplet sums, palindromes, in-place partitioning, removing duplicates.

```cpp
// Opposite ends — converging
int l = 0, r = n - 1;
while (l < r) {
    int sum = a[l] + a[r];
    if (sum == target) return {l, r};
    if (sum < target) ++l;      // need bigger → move left pointer right
    else --r;                    // need smaller → move right pointer left
}

// Same direction — slow/fast (in-place filtering)
int slow = 0;
for (int fast = 0; fast < n; ++fast) {
    if (keep(a[fast])) a[slow++] = a[fast];
}
return slow;                     // new length
```

⭐ **Why it works on sorted input:** each comparison eliminates an entire row or column of the O(n²) search space, so you get O(n).

---

### Pattern 2 — Sliding Window

**When:** contiguous subarray/substring with a constraint (longest, shortest, count).

```cpp
// VARIABLE window — "longest/shortest subarray satisfying X"
int left = 0, best = 0;
unordered_map<char,int> cnt;
for (int right = 0; right < n; ++right) {
    cnt[s[right]]++;                          // EXPAND

    while (/* window is invalid */) {          // SHRINK
        if (--cnt[s[left]] == 0) cnt.erase(s[left]);
        ++left;
    }
    best = max(best, right - left + 1);        // RECORD
}

// FIXED window of size k
long long sum = 0, best = LLONG_MIN;
for (int i = 0; i < n; ++i) {
    sum += a[i];
    if (i >= k) sum -= a[i - k];               // slide
    if (i >= k - 1) best = max(best, sum);
}
```

⭐ **The template is always expand → while-invalid-shrink → record.** Each element enters and leaves the window at most once → O(n).

---

### Pattern 3 — Fast & Slow Pointers (Floyd)

**When:** cycle detection, middle of a list, k-th from end, happy number.

```cpp
// Cycle detection + finding the entry point
ListNode* detectCycle(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) {                 // cycle exists
            ListNode* p = head;
            while (p != slow) { p = p->next; slow = slow->next; }
            return p;                        // entry point
        }
    }
    return nullptr;
}
```

```
   WHY the second phase works:

   head ──a──▶ [entry] ──b──▶ [meet]
                  ▲              │
                  └──────c───────┘

   Slow travelled: a + b
   Fast travelled: a + b + c + b  = 2(a + b)
   →  a = c
   → moving one pointer from head and one from meet,
     both at speed 1, they meet exactly at the entry.
```

---

### Pattern 4 — Merge Intervals

**When:** overlapping ranges, scheduling, calendars.

```cpp
sort(iv.begin(), iv.end());                    // by start
vector<vector<int>> out;
for (auto& cur : iv) {
    if (out.empty() || out.back()[1] < cur[0]) out.push_back(cur);
    else out.back()[1] = max(out.back()[1], cur[1]);   // merge
}
```

```cpp
// SWEEP LINE — for "max concurrent" problems (meeting rooms II)
vector<pair<int,int>> ev;                       // {time, +1 start / -1 end}
for (auto& i : iv) { ev.push_back({i[0], 1}); ev.push_back({i[1], -1}); }
sort(ev.begin(), ev.end());                     // -1 before +1 at equal time
int cur = 0, best = 0;
for (auto& [t, d] : ev) { cur += d; best = max(best, cur); }
```

---

### Pattern 5 — Cyclic Sort

**When:** array contains numbers in a known range 1..n (or 0..n-1) — find missing/duplicate in O(n) time, O(1) space.

```cpp
int i = 0;
while (i < n) {
    int j = a[i] - 1;                  // where a[i] SHOULD live
    if (a[i] > 0 && a[i] <= n && a[i] != a[j]) swap(a[i], a[j]);
    else ++i;
}
for (int i = 0; i < n; ++i) if (a[i] != i + 1) return i + 1;   // missing
```

⭐ Each swap places one number correctly, so total swaps ≤ n → O(n) despite the nested-looking structure.

---

### Pattern 6 — In-Place Linked List Reversal

```cpp
ListNode* reverse(ListNode* head) {
    ListNode *prev = nullptr, *cur = head;
    while (cur) {
        ListNode* nxt = cur->next;     // save
        cur->next = prev;              // reverse
        prev = cur;                    // advance
        cur = nxt;
    }
    return prev;
}
```

```
   Visual:   null ← 1   2 → 3 → null
                    ▲   ▲
                  prev cur
```

---

### Pattern 7 — BFS (Level Order)

**When:** shortest path in an unweighted graph, level-by-level tree traversal, minimum steps.

```cpp
queue<Node> q;  q.push(start);
unordered_set<Node> seen{start};
int steps = 0;

while (!q.empty()) {
    int sz = q.size();                      // ⭐ freeze the level size
    for (int i = 0; i < sz; ++i) {
        auto cur = q.front(); q.pop();
        if (cur == target) return steps;
        for (auto& nxt : neighbors(cur)) {
            if (!seen.count(nxt)) { seen.insert(nxt); q.push(nxt); }
        }
    }
    ++steps;
}
```

⚠️ **Mark as seen when ENQUEUEING, not when dequeueing** — otherwise a node can be added to the queue many times before it's processed.

---

### Pattern 8 — DFS

**When:** all paths, connected components, cycle detection, tree recursion.

```cpp
// Recursive
void dfs(int u) {
    seen[u] = true;
    for (int v : adj[u]) if (!seen[v]) dfs(v);
}

// Iterative (avoids stack overflow on deep graphs)
stack<int> st; st.push(s);
while (!st.empty()) {
    int u = st.top(); st.pop();
    if (seen[u]) continue;
    seen[u] = true;
    for (int v : adj[u]) if (!seen[v]) st.push(v);
}

// Cycle detection in a DIRECTED graph — 3 colors
// 0 = unvisited, 1 = in current path, 2 = done
bool hasCycle(int u) {
    color[u] = 1;
    for (int v : adj[u]) {
        if (color[v] == 1) return true;              // back edge → cycle
        if (color[v] == 0 && hasCycle(v)) return true;
    }
    color[u] = 2;
    return false;
}
```

---

### Pattern 9 — Topological Sort

**When:** dependency ordering, course schedule, build order. Only for DAGs.

```cpp
// KAHN'S ALGORITHM (BFS) — also detects cycles
vector<int> indeg(n, 0);
for (int u = 0; u < n; ++u) for (int v : adj[u]) indeg[v]++;

queue<int> q;
for (int i = 0; i < n; ++i) if (indeg[i] == 0) q.push(i);

vector<int> order;
while (!q.empty()) {
    int u = q.front(); q.pop();
    order.push_back(u);
    for (int v : adj[u]) if (--indeg[v] == 0) q.push(v);
}
if ((int)order.size() != n) return {};      // ⭐ cycle detected
return order;
```

---

### Pattern 10 — Union-Find (DSU)

**When:** dynamic connectivity, Kruskal's MST, grouping, cycle detection in undirected graphs.

```cpp
struct DSU {
    vector<int> p, r;
    int comps;
    DSU(int n) : p(n), r(n, 0), comps(n) { iota(p.begin(), p.end(), 0); }

    int find(int x) {                          // path compression
        return p[x] == x ? x : p[x] = find(p[x]);
    }
    bool unite(int a, int b) {                 // union by rank
        a = find(a); b = find(b);
        if (a == b) return false;              // already connected
        if (r[a] < r[b]) swap(a, b);
        p[b] = a;
        if (r[a] == r[b]) ++r[a];
        --comps;
        return true;
    }
};
```

⭐ With both optimizations: near-O(1) amortized (inverse Ackermann, α(n) < 5 for any real n).

---

### Pattern 11 — Heap / Top-K

**When:** k largest/smallest, streaming median, merge k lists, scheduling.

```cpp
// K LARGEST → use a MIN-heap of size k (counterintuitive but correct)
priority_queue<int, vector<int>, greater<int>> pq;
for (int x : a) {
    pq.push(x);
    if ((int)pq.size() > k) pq.pop();          // drop the smallest
}
// pq.top() is the k-th largest;  O(n log k) beats O(n log n) sorting

// TWO HEAPS for a running median
priority_queue<int> lo;                                   // max-heap, lower half
priority_queue<int, vector<int>, greater<int>> hi;        // min-heap, upper half

void add(int x) {
    lo.push(x);
    hi.push(lo.top()); lo.pop();                // balance content
    if (hi.size() > lo.size()) { lo.push(hi.top()); hi.pop(); }
}
double median() {
    return lo.size() > hi.size() ? lo.top() : (lo.top() + hi.top()) / 2.0;
}
```

---

### Pattern 12 — Binary Search

```cpp
// STANDARD — find target
int lo = 0, hi = n - 1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;              // ⭐ avoids overflow
    if (a[mid] == target) return mid;
    if (a[mid] < target) lo = mid + 1;
    else hi = mid - 1;
}
return -1;

// LOWER BOUND — first index with a[i] >= target
int lo = 0, hi = n;                            // note: hi = n
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (a[mid] < target) lo = mid + 1;
    else hi = mid;
}
return lo;

// BINARY SEARCH ON THE ANSWER ⭐⭐ the highest-value variant
// "minimum capacity/speed/size such that check() passes"
int lo = minPossible, hi = maxPossible;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (feasible(mid)) hi = mid;               // try smaller
    else lo = mid + 1;
}
return lo;
```

```
   The monotonic predicate is the key:

   feasible:  F F F F T T T T T
                      ↑
                 find this boundary

   If the answer space is monotonic — "if capacity c works, so does c+1" —
   you can binary search it even when the input isn't sorted at all.
```

---

### Pattern 13 — Backtracking

**When:** generate all subsets/permutations/combinations, N-Queens, Sudoku, word search.

```cpp
void backtrack(vector<int>& path, State& st) {
    if (isComplete(st)) { result.push_back(path); return; }

    for (auto& choice : choices(st)) {
        if (!valid(choice, st)) continue;      // PRUNE early

        path.push_back(choice);  apply(choice, st);      // CHOOSE
        backtrack(path, st);                              // EXPLORE
        path.pop_back();         undo(choice, st);        // UN-CHOOSE
    }
}
```

```cpp
// SUBSETS — the include/exclude template
void subsets(int i, vector<int>& cur) {
    if (i == n) { res.push_back(cur); return; }
    subsets(i + 1, cur);                       // exclude
    cur.push_back(a[i]);
    subsets(i + 1, cur);                       // include
    cur.pop_back();
}

// PERMUTATIONS
void perms(vector<int>& cur, vector<bool>& used) {
    if (cur.size() == n) { res.push_back(cur); return; }
    for (int i = 0; i < n; ++i) {
        if (used[i]) continue;
        // skip duplicates (requires sorted input):
        if (i > 0 && a[i] == a[i-1] && !used[i-1]) continue;
        used[i] = true; cur.push_back(a[i]);
        perms(cur, used);
        cur.pop_back(); used[i] = false;
    }
}

// COMBINATIONS — start index prevents revisiting
void combine(int start, vector<int>& cur) {
    if (cur.size() == k) { res.push_back(cur); return; }
    for (int i = start; i <= n; ++i) {
        cur.push_back(i);
        combine(i + 1, cur);                   // i+1: no reuse; i: allows reuse
        cur.pop_back();
    }
}
```

---

### Pattern 14 — Dynamic Programming

```
   DP requires TWO properties:
   1. OPTIMAL SUBSTRUCTURE — the optimal solution contains
      optimal solutions to subproblems
   2. OVERLAPPING SUBPROBLEMS — the same subproblem recurs

   THE PROCESS:
   ┌────────────────────────────────────────────────────┐
   │ 1. Define the STATE: what parameters identify a     │
   │    subproblem? dp[i], dp[i][j], dp[i][mask]...      │
   │ 2. Write the RECURRENCE: dp[i] in terms of smaller  │
   │ 3. Set BASE CASES                                   │
   │ 4. Determine the ORDER of computation               │
   │ 5. Optimize SPACE (usually only the last 1-2 rows)  │
   └────────────────────────────────────────────────────┘
```

```cpp
// TOP-DOWN (memoization) — write this first, it mirrors the recursion
vector<int> memo(n + 1, -1);
int solve(int i) {
    if (i <= 1) return base;
    int& m = memo[i];
    if (m != -1) return m;
    return m = combine(solve(i - 1), solve(i - 2));
}

// BOTTOM-UP (tabulation)
vector<int> dp(n + 1);
dp[0] = b0; dp[1] = b1;
for (int i = 2; i <= n; ++i) dp[i] = combine(dp[i-1], dp[i-2]);

// SPACE OPTIMIZED
int a = b0, b = b1;
for (int i = 2; i <= n; ++i) { int c = combine(a, b); a = b; b = c; }
```

**The classic DP families:**

| Family | State | Examples |
|---|---|---|
| Linear | `dp[i]` | House robber, climbing stairs, LIS |
| Grid | `dp[i][j]` | Unique paths, min path sum, edit distance |
| Knapsack 0/1 | `dp[i][w]` | Subset sum, partition, target sum |
| Unbounded knapsack | `dp[w]` | Coin change, rod cutting |
| Two sequences | `dp[i][j]` | LCS, edit distance, regex match |
| Interval | `dp[i][j]` | Burst balloons, matrix chain, palindrome partition |
| Bitmask | `dp[mask]` | TSP, assignment (n ≤ 20) |
| Digit | `dp[pos][tight][...]` | Count numbers with a property |
| Tree | `dp[node][state]` | House robber III, tree diameter |

---

### Pattern 15 — Monotonic Stack

**When:** next/previous greater/smaller element, histogram, trapping rain, stock span.

```cpp
// NEXT GREATER ELEMENT — stack holds indices, values DECREASING
vector<int> nge(n, -1);
stack<int> st;
for (int i = 0; i < n; ++i) {
    while (!st.empty() && a[st.top()] < a[i]) {
        nge[st.top()] = a[i];                  // a[i] is the next greater
        st.pop();
    }
    st.push(i);
}
```

```
   INTUITION: the stack holds "elements still waiting for their answer."
   When a bigger element arrives, everyone waiting for a bigger
   element gets resolved at once. Each index is pushed and popped
   exactly once → O(n).
```

```cpp
// LARGEST RECTANGLE IN HISTOGRAM — the canonical hard application
int largestRectangle(vector<int>& h) {
    h.push_back(0);                            // ⭐ sentinel flushes the stack
    stack<int> st;
    int best = 0;
    for (int i = 0; i < (int)h.size(); ++i) {
        while (!st.empty() && h[st.top()] >= h[i]) {
            int height = h[st.top()]; st.pop();
            int left = st.empty() ? -1 : st.top();
            best = max(best, height * (i - left - 1));
        }
        st.push(i);
    }
    return best;
}
```

---

### Pattern 16 — Prefix Sum & Difference Array

```cpp
// PREFIX SUM — O(1) range sum queries after O(n) preprocessing
vector<long long> pre(n + 1, 0);
for (int i = 0; i < n; ++i) pre[i+1] = pre[i] + a[i];
long long rangeSum = pre[r+1] - pre[l];        // sum of a[l..r]

// PREFIX SUM + HASH MAP — count subarrays with sum k  ⭐ very common
unordered_map<long long,int> cnt{{0, 1}};      // empty prefix
long long sum = 0; int ans = 0;
for (int x : a) {
    sum += x;
    ans += cnt[sum - k];                       // how many prefixes make sum-k
    cnt[sum]++;
}

// DIFFERENCE ARRAY — O(1) range updates, O(n) final materialization
vector<int> diff(n + 1, 0);
// add v to a[l..r]:
diff[l] += v; diff[r+1] -= v;
// materialize:
for (int i = 1; i < n; ++i) diff[i] += diff[i-1];

// 2D PREFIX SUM
pre[i+1][j+1] = a[i][j] + pre[i][j+1] + pre[i+1][j] - pre[i][j];
// rectangle sum (r1,c1) to (r2,c2):
sum = pre[r2+1][c2+1] - pre[r1][c2+1] - pre[r2+1][c1] + pre[r1][c1];
```

---

### Pattern 17 — Trie

**When:** prefix search, autocomplete, word dictionaries, XOR maximization.

```cpp
struct Trie {
    struct Node {
        Node* child[26] = {};
        bool end = false;
        int count = 0;                          // words passing through
    };
    Node* root = new Node();

    void insert(const string& w) {
        Node* cur = root;
        for (char c : w) {
            int i = c - 'a';
            if (!cur->child[i]) cur->child[i] = new Node();
            cur = cur->child[i];
            cur->count++;
        }
        cur->end = true;
    }

    bool search(const string& w) {
        Node* cur = walk(w);
        return cur && cur->end;
    }
    bool startsWith(const string& p) { return walk(p) != nullptr; }

private:
    Node* walk(const string& s) {
        Node* cur = root;
        for (char c : s) {
            int i = c - 'a';
            if (!cur->child[i]) return nullptr;
            cur = cur->child[i];
        }
        return cur;
    }
};
```

Complexity: insert/search O(L) where L is word length — independent of dictionary size. That's the whole point.

---

### Pattern 18 — Bit Manipulation

```cpp
x & 1                    // is odd
x >> 1                   // divide by 2
x & (x - 1)              // ⭐ clear the lowest set bit
x & (-x)                 // ⭐ isolate the lowest set bit
x | (x + 1)              // set the lowest zero bit
__builtin_popcount(x)    // count set bits (popcountll for 64-bit)
__builtin_clz(x)         // leading zeros
__builtin_ctz(x)         // trailing zeros
(x & (x - 1)) == 0       // is a power of two (x > 0)

// Subset enumeration of a mask ⭐
for (int sub = mask; sub; sub = (sub - 1) & mask) { /* every subset */ }

// XOR properties — the basis of many tricks
a ^ a = 0    a ^ 0 = a    XOR is commutative and associative
→ "every element appears twice except one" → XOR everything
```

```cpp
// Single number when others appear 3 times
int ones = 0, twos = 0;
for (int x : a) {
    ones = (ones ^ x) & ~twos;
    twos = (twos ^ x) & ~ones;
}
return ones;
```

---

### Pattern 19 — Greedy

**When:** a locally optimal choice provably leads to the global optimum.

```
   ⚠️ Greedy REQUIRES PROOF. The two arguments:

   1. EXCHANGE ARGUMENT — take any optimal solution; show you can
      swap in the greedy choice without making it worse.
   2. GREEDY STAYS AHEAD — show that after each step, the greedy
      solution is at least as good as any other partial solution.

   Classic greedy problems:
     Activity selection    → sort by END time
     Fractional knapsack   → sort by value/weight
     Huffman coding        → always merge the two smallest
     Jump game             → track the furthest reachable index
     Gas station           → if total ≥ 0, a solution exists; reset at deficits
     Minimum platforms     → sweep line

   Classic greedy FAILURES:
     0/1 knapsack          → needs DP
     Coin change (arbitrary denominations) → needs DP
     Longest path          → NP-hard
```

---

### Pattern 20 — Graph Shortest Paths

```cpp
// DIJKSTRA — non-negative weights, O((V+E) log V)
vector<long long> dist(n, LLONG_MAX);
priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
dist[s] = 0; pq.push({0, s});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;                 // ⭐ stale entry
    for (auto& [v, w] : adj[u]) {
        if (dist[u] + w < dist[v]) {
            dist[v] = dist[u] + w;
            pq.push({dist[v], v});
        }
    }
}

// BELLMAN-FORD — handles NEGATIVE weights, detects negative cycles, O(V·E)
vector<long long> dist(n, INF); dist[s] = 0;
for (int i = 0; i < n - 1; ++i)
    for (auto& [u, v, w] : edges)
        if (dist[u] != INF && dist[u] + w < dist[v]) dist[v] = dist[u] + w;
for (auto& [u, v, w] : edges)                  // one more pass
    if (dist[u] != INF && dist[u] + w < dist[v]) return "negative cycle";

// FLOYD-WARSHALL — all pairs, O(V³), V ≤ ~500
for (int k = 0; k < n; ++k)
  for (int i = 0; i < n; ++i)
    for (int j = 0; j < n; ++j)
      d[i][j] = min(d[i][j], d[i][k] + d[k][j]);
```

| Algorithm | Weights | Complexity | Use |
|---|---|---|---|
| BFS | Unweighted (all = 1) | O(V+E) | Fewest edges |
| 0-1 BFS (deque) | Weights ∈ {0,1} | O(V+E) | Grid with free moves |
| Dijkstra | Non-negative | O((V+E) log V) | Standard shortest path |
| Bellman-Ford | Any (detects neg cycles) | O(V·E) | Negative weights, arbitrage |
| Floyd-Warshall | Any | O(V³) | All pairs, small V |
| A* | Non-negative + heuristic | Varies | Pathfinding with a goal |

---

## 5. C++ Toolkit

```cpp
#include <bits/stdc++.h>            // everything (fine for interviews/CP)
using namespace std;
using ll = long long;
using pii = pair<int,int>;

// ── Containers ────────────────────────────────────────────
vector<int> v(n, 0);
vector<vector<int>> g(n);                       // adjacency list
array<int, 5> a{};                              // fixed size, stack allocated
deque<int> dq;                                  // O(1) both ends
set<int> s;                                     // ordered, O(log n)
multiset<int> ms;                               // duplicates allowed
unordered_map<int,int> mp;                      // O(1) average
map<int,int> om;                                // ordered, O(log n)
priority_queue<int> maxh;
priority_queue<int, vector<int>, greater<int>> minh;

// ── Algorithms ────────────────────────────────────────────
sort(v.begin(), v.end());
sort(v.begin(), v.end(), [](int a, int b){ return a > b; });   // descending
stable_sort(v.begin(), v.end());
reverse(v.begin(), v.end());
lower_bound(v.begin(), v.end(), x) - v.begin(); // first >= x
upper_bound(v.begin(), v.end(), x) - v.begin(); // first > x
*max_element(v.begin(), v.end());
accumulate(v.begin(), v.end(), 0LL);            // ⭐ 0LL to avoid int overflow
count(v.begin(), v.end(), x);
v.erase(unique(v.begin(), v.end()), v.end());   // dedupe (after sorting)
next_permutation(v.begin(), v.end());
partial_sum(v.begin(), v.end(), pre.begin());
iota(v.begin(), v.end(), 0);                    // fill 0,1,2,...
nth_element(v.begin(), v.begin()+k, v.end());   // O(n) quickselect

// ── Strings ───────────────────────────────────────────────
string s = "abc";
s.substr(i, len);  s.find("x");  stoi(s);  to_string(42);
s += 'a';   reverse(s.begin(), s.end());
transform(s.begin(), s.end(), s.begin(), ::tolower);
istringstream iss(line);  while (iss >> word) { }

// ── Constants ─────────────────────────────────────────────
const int INF = 1e9;                            // fits in int, safe to add twice
const ll LINF = 1e18;
const int MOD = 1e9 + 7;

// ── Fast I/O (competitive programming) ────────────────────
ios_base::sync_with_stdio(false); cin.tie(nullptr);

// ── Custom comparator for a priority_queue ────────────────
struct Cmp {
    bool operator()(const pii& a, const pii& b) const { return a.second > b.second; }
};
priority_queue<pii, vector<pii>, Cmp> pq;

// ── Structured bindings + auto ────────────────────────────
for (auto& [key, val] : mp) { }
auto [a, b] = make_pair(1, 2);

// ── Grid directions ───────────────────────────────────────
const int dr[] = {-1, 1, 0, 0};                 // 4-directional
const int dc[] = {0, 0, -1, 1};
// 8-directional: {-1,-1,-1,0,0,1,1,1} and {-1,0,1,-1,1,-1,0,1}
for (int d = 0; d < 4; ++d) {
    int nr = r + dr[d], nc = c + dc[d];
    if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
    ...
}
```

---

## 6. Complexity Table

| Structure | Access | Search | Insert | Delete | Space |
|---|---|---|---|---|---|
| Array | O(1) | O(n) | O(n) | O(n) | O(n) |
| Dynamic array | O(1) | O(n) | O(1)* | O(n) | O(n) |
| Linked list | O(n) | O(n) | O(1)† | O(1)† | O(n) |
| Stack / Queue | O(n) | O(n) | O(1) | O(1) | O(n) |
| Hash table | — | O(1)‡ | O(1)‡ | O(1)‡ | O(n) |
| BST (balanced) | O(log n) | O(log n) | O(log n) | O(log n) | O(n) |
| Heap | O(1) top | O(n) | O(log n) | O(log n) | O(n) |
| Trie | O(L) | O(L) | O(L) | O(L) | O(ALPHABET·N·L) |
| Union-Find | — | O(α) | O(α) | — | O(n) |
| Segment tree | O(log n) | O(log n) | O(log n) | — | O(n) |
| Fenwick (BIT) | O(log n) | O(log n) | O(log n) | — | O(n) |

\* amortized  † with a pointer to the node  ‡ average; O(n) worst case

| Sort | Best | Average | Worst | Space | Stable |
|---|---|---|---|---|---|
| Quicksort | O(n log n) | O(n log n) | **O(n²)** | O(log n) | ❌ |
| Mergesort | O(n log n) | O(n log n) | O(n log n) | **O(n)** | ✅ |
| Heapsort | O(n log n) | O(n log n) | O(n log n) | **O(1)** | ❌ |
| Timsort | **O(n)** | O(n log n) | O(n log n) | O(n) | ✅ |
| Insertion | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Counting | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ |
| Radix | O(nk) | O(nk) | O(nk) | O(n+k) | ✅ |

---

## 7. Edge Cases

Run this list on **every** problem before declaring you're done:

```
   INPUT
   □ Empty input          []    ""    nullptr
   □ Single element       [x]
   □ Two elements         [x, y]
   □ All identical        [5,5,5,5]
   □ Already sorted / reverse sorted
   □ Negative numbers, zero
   □ Maximum size (does it TLE? does the stack overflow?)
   □ Maximum values (INT overflow? use long long)

   STRUCTURE
   □ Linked list: head is the target, single node, cycle
   □ Tree: empty, single node, skewed (a linked list), full
   □ Graph: disconnected, self-loop, multi-edge, single node
   □ Grid: 1×n, n×1, 1×1, blocked start/end
   □ Intervals: touching (end == start), fully nested, identical

   LOGIC
   □ Target not present
   □ Duplicates in the answer
   □ Integer division truncation
   □ Off-by-one at boundaries (i vs i+1, <= vs <)
```

---

## 8. Common Bugs

```cpp
// ❌ Integer overflow in the midpoint
int mid = (lo + hi) / 2;                 // overflows when lo+hi > INT_MAX
int mid = lo + (hi - lo) / 2;            // ✅

// ❌ Sum overflow
int sum = 0; for (int x : a) sum += x;   // 10⁵ × 10⁵ overflows
long long sum = 0;                        // ✅

// ❌ Modifying a container while iterating
for (auto x : v) if (bad(x)) v.erase(...);   // UB
v.erase(remove_if(v.begin(), v.end(), bad), v.end());  // ✅

// ❌ unordered_map operator[] INSERTS on read
if (mp[key] > 0) { }                      // creates key with value 0!
if (mp.count(key) && mp[key] > 0) { }     // ✅
auto it = mp.find(key); if (it != mp.end()) { }  // ✅ better

// ❌ Comparing signed and unsigned
for (int i = 0; i < v.size() - 1; ++i)    // v.size() is unsigned;
                                           // empty vector → huge number → crash
for (int i = 0; i + 1 < (int)v.size(); ++i)  // ✅

// ❌ Forgetting to mark visited when ENQUEUEING in BFS
q.push(v);                                // node added many times
seen.insert(v); q.push(v);                // ✅

// ❌ Reference invalidated by reallocation
int& x = v[0]; v.push_back(1); x = 5;     // dangling if v reallocated

// ❌ Returning a reference to a local
int& f() { int x = 5; return x; }         // dangling

// ❌ Recursion depth
// n = 10⁵ recursive DFS on a path graph → stack overflow.
// Convert to iterative, or increase the stack.
```

---

## Next Steps

Work through the problem books in this order. Each begins with its own pattern refresher.

| # | Book | Problems |
|---|---|---|
| 1 | [Arrays & Strings](01-arrays-strings.md) | ~70 |
| 2 | [Hashing](02-hashing.md) | ~30 |
| 3 | [Two Pointers & Sliding Window](03-two-pointers-sliding-window.md) | ~40 |
| 4 | [Linked Lists](04-linked-lists.md) | ~25 |
| 5 | [Stacks & Queues](05-stacks-queues.md) | ~30 |
| 6 | [Trees](06-trees.md) | ~50 |
| 7 | [Heaps & Intervals](07-heaps-intervals.md) | ~30 |
| 8 | [Graphs](08-graphs.md) | ~50 |
| 9 | [Dynamic Programming](09-dynamic-programming.md) | ~60 |
| 10 | [Greedy, Backtracking, Bits, Math](10-greedy-backtracking-misc.md) | ~45 |
| — | [Master Index](INDEX.md) | all 400+ |
