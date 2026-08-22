# 📇 Master Problem Index

> All problems, cross-referenced by pattern, difficulty, and frequency. Use this to build practice sets and to look up "what pattern is this?"
>
> ⚠️ **Format note:** Books 01–10 have been rebuilt with Mermaid approach-ladder diagrams — every problem now shows all viable approaches ranked worst→best by time/space, with plain-language explanations before code. See [FORMAT-SAMPLE.md](FORMAT-SAMPLE.md). Arrays & Strings is now split across three files: [01](01-arrays-strings.md) (1–20), [01b](01b-arrays-strings.md) (21–45), [01c](01c-arrays-strings.md) (46–70). Anchor links below point to section headers, not numbered problem IDs — use each book's own Contents table for exact links.

---

## 🔍 Reverse Lookup — "What Pattern Is This?"

Find the phrase closest to your problem statement.

| The problem says... | Reach for | Book |
|---|---|---|
| "sorted array" + "find a pair/triplet" | Two pointers from both ends | [03](03-two-pointers-sliding-window.md#1-two-sum-ii-sorted) |
| "contiguous subarray/substring" + max/min length | Sliding window | [03](03-two-pointers-sliding-window.md#-the-universal-sliding-window-template) |
| "subarray sum equals k" + **negatives possible** | Prefix sum + hash map | [02](02-hashing.md#2-subarray-sum-equals-k) |
| "subarray sum equals k" + all positive | Sliding window | [03](03-two-pointers-sliding-window.md#9-minimum-size-subarray-sum) |
| "exactly K distinct/odd/…" | `atMost(K) − atMost(K−1)` | [03](03-two-pointers-sliding-window.md#14-subarrays-with-k-different-integers) |
| "top k" / "k-th largest" | Heap of size k, or quickselect | [07](07-heaps-intervals.md#1-kth-largest-element-in-an-array) |
| "median of a stream" | Two heaps | [07](07-heaps-intervals.md#4-find-median-from-data-stream) |
| "merge k sorted …" | Heap, or divide & conquer | [04](04-linked-lists.md#5-merge-k-sorted-lists) |
| "next/previous greater/smaller" | Monotonic stack | [05](05-stacks-queues.md#5-next-greater-element-i--ii) |
| "largest rectangle" / "trapping water" | Monotonic stack | [05](05-stacks-queues.md#8-largest-rectangle-in-histogram) |
| "max/min of every window of size k" | Monotonic deque | [03](03-two-pointers-sliding-window.md#12-sliding-window-maximum) |
| "intervals" + merge/insert/intersect | Sort by **start** | [01b](01b-arrays-strings.md#23-merge-intervals) |
| "maximum non-overlapping" / "min arrows" | Sort by **end** (activity selection) | [01b](01b-arrays-strings.md#25-non-overlapping-intervals) |
| "minimum rooms" / "max concurrent" | Sweep line (+1/−1 events) | [01b](01b-arrays-strings.md#27-meeting-rooms-ii) |
| "cycle in a linked list" / "find the middle" | Fast & slow pointers | [03](03-two-pointers-sliding-window.md#15-linked-list-cycle-floyds) |
| "reverse a list" / "reorder" | In-place reversal | [04](04-linked-lists.md#1-reverse-linked-list) |
| "level by level" / "shortest in unweighted" | BFS with `int sz = q.size()` | [06](06-trees.md#8-level-order-traversal) |
| "distance from **many** sources" | Multi-source BFS | [08](08-graphs.md#5-rotting-oranges) |
| "shortest path" + weights ≥ 0 | Dijkstra | [08](08-graphs.md#17-network-delay-time-dijkstra) |
| "shortest path" + at most k edges | Bellman-Ford (copy dist per round) | [08](08-graphs.md#18-cheapest-flights-within-k-stops) |
| "prerequisites" / "build order" | Topological sort (Kahn's) | [08](08-graphs.md#8-course-schedule-i--ii) |
| "connected components" + **dynamic** edges | Union-Find | [08](08-graphs.md#14-union-find-the-structure) |
| "minimum spanning tree" / "connect all at min cost" | Kruskal (DSU) or Prim (heap) | [08](08-graphs.md#21-minimum-spanning-tree-kruskal--prim) |
| "count ways" / "min cost" + choices repeat | DP | [09](09-dynamic-programming.md) |
| "two strings" + match/transform | 2D DP `dp[i][j]` | [09](09-dynamic-programming.md#9-longest-common-subsequence) |
| "pick items with a capacity limit" | Knapsack (mind loop direction!) | [09](09-dynamic-programming.md#5-01-knapsack) |
| "burst/merge within a range" | Interval DP (loop by length) | [09](09-dynamic-programming.md#21-burst-balloons) |
| "generate all …" / "all combinations" | Backtracking | [10](10-greedy-backtracking-misc.md#-backtracking-the-universal-skeleton) |
| "minimum/maximum" + a provable local choice | Greedy (**prove it!**) | [10](10-greedy-backtracking-misc.md#-greedy-proving-its-safe) |
| "appears once, others twice" | XOR everything | [10](10-greedy-backtracking-misc.md#13-single-number-i-ii-iii) |
| "prefix" / "autocomplete" / "dictionary" | Trie | [06](06-trees.md#19-trie-prefix-tree) |
| "minimum X such that f(X) works" | **Binary search on the answer** | [07](07-heaps-intervals.md#7-kth-smallest-in-a-sorted-matrix) |
| "values are 1..n, find missing/duplicate" | Cyclic sort or sign marking | [01b](01b-arrays-strings.md#35-first-missing-positive) |

---

## ⭐ The Blind 75 Core — Start Here

If you only have time for 40 problems, these are the ones. Each unlocks a whole family.

| # | Problem | Pattern | Book |
|---|---|---|---|
| 1 | Two Sum | Hash map complement | [01](01-arrays-strings.md#1-two-sum) |
| 2 | Best Time to Buy/Sell Stock | Running min | [01](01-arrays-strings.md#2-best-time-to-buy-and-sell-stock) |
| 3 | Product Except Self | Prefix × suffix | [01](01-arrays-strings.md#4-product-of-array-except-self) |
| 4 | Maximum Subarray | Kadane | [01](01-arrays-strings.md#5-maximum-subarray-kadane) |
| 5 | Maximum Product Subarray | Track max **and** min | [01](01-arrays-strings.md#6-maximum-product-subarray) |
| 6 | Find Min in Rotated Sorted | Binary search | [01](01-arrays-strings.md#7-find-minimum-in-rotated-sorted-array) |
| 7 | Search in Rotated Sorted | Binary search, one half sorted | [01](01-arrays-strings.md#8-search-in-rotated-sorted-array) |
| 8 | 3Sum | Sort + two pointers | [03](03-two-pointers-sliding-window.md#2-3sum) |
| 9 | Container With Most Water | Two pointers, move shorter | [03](03-two-pointers-sliding-window.md#4-container-with-most-water) |
| 10 | Trapping Rain Water | Two pointers / monotonic stack | [03](03-two-pointers-sliding-window.md#5-trapping-rain-water) |
| 11 | Longest Substring No Repeat | Sliding window | [01c](01c-arrays-strings.md#59-longest-substring-without-repeating) |
| 12 | Longest Repeating Char Replace | Sliding window + maxCount | [01c](01c-arrays-strings.md#60-longest-repeating-character-replacement) |
| 13 | Minimum Window Substring | Sliding window + missing counter | [03](03-two-pointers-sliding-window.md#10-minimum-window-substring) |
| 14 | Group Anagrams | Count-array key | [01c](01c-arrays-strings.md#47-group-anagrams) |
| 15 | Longest Consecutive Sequence | Set + start-of-run guard | [02](02-hashing.md#1-longest-consecutive-sequence) |
| 16 | LRU Cache | Hash map + doubly linked list | [02](02-hashing.md#3-lru-cache) |
| 17 | Reverse Linked List | 3-pointer reversal | [04](04-linked-lists.md#1-reverse-linked-list) |
| 18 | Linked List Cycle II | Floyd + restart from head | [03](03-two-pointers-sliding-window.md#15-linked-list-cycle-floyds) |
| 19 | Merge k Sorted Lists | Heap / divide & conquer | [04](04-linked-lists.md#5-merge-k-sorted-lists) |
| 20 | Valid Parentheses | Stack | [01c](01c-arrays-strings.md#67-valid-parentheses) |
| 21 | Daily Temperatures | Monotonic stack | [05](05-stacks-queues.md#6-daily-temperatures) |
| 22 | Largest Rectangle Histogram | Monotonic stack | [05](05-stacks-queues.md#8-largest-rectangle-in-histogram) |
| 23 | Sliding Window Maximum | Monotonic deque | [03](03-two-pointers-sliding-window.md#12-sliding-window-maximum) |
| 24 | Maximum Depth of Binary Tree | Tree recursion | [06](06-trees.md#1-maximum-depth-of-binary-tree) |
| 25 | Diameter of Binary Tree | ⭐ both children vs one | [06](06-trees.md#3-diameter-of-binary-tree) |
| 26 | Binary Tree Max Path Sum | Same pattern as diameter | [06](06-trees.md#4-binary-tree-maximum-path-sum) |
| 27 | Validate BST | Pass bounds down | [06](06-trees.md#10-validate-binary-search-tree) |
| 28 | Lowest Common Ancestor | Return "found or LCA" | [06](06-trees.md#12-lowest-common-ancestor-bst--binary-tree) |
| 29 | Serialize/Deserialize Tree | Preorder + null markers | [06](06-trees.md#14-serialize-and-deserialize) |
| 30 | Number of Islands | Flood fill | [08](08-graphs.md#1-number-of-islands) |
| 31 | Rotting Oranges | Multi-source BFS | [08](08-graphs.md#5-rotting-oranges) |
| 32 | Course Schedule | Topological sort | [08](08-graphs.md#8-course-schedule-i--ii) |
| 33 | Word Ladder | Implicit graph BFS | [08](08-graphs.md#7-word-ladder) |
| 34 | Network Delay Time | Dijkstra | [08](08-graphs.md#17-network-delay-time-dijkstra) |
| 35 | Climbing Stairs | 1D DP intro | [09](09-dynamic-programming.md#1-climbing-stairs--fibonacci-family) |
| 36 | House Robber | Rob vs skip | [09](09-dynamic-programming.md#2-house-robber-i--ii) |
| 37 | Coin Change | Unbounded knapsack | [09](09-dynamic-programming.md#3-coin-change) |
| 38 | Longest Increasing Subsequence | DP + patience O(n log n) | [09](09-dynamic-programming.md#8-longest-increasing-subsequence) |
| 39 | Edit Distance | 2D DP, 3 operations | [09](09-dynamic-programming.md#10-edit-distance) |
| 40 | Word Break | DP + dictionary set | [01c](01c-arrays-strings.md#70-word-break) |

---

## 📚 Full Index by Book

Each book now carries its own **Contents table** at the top with correct, current links — treat that as the authoritative per-problem index. This section gives the topic breakdown only.

### [01 / 01b / 01c — Arrays & Strings](01-arrays-strings.md) · 70 problems across 3 files
Part 1 (1–20): fundamentals, Kadane, binary search, prefix sums · Part 2 (21–45): sorting, in-place, matrix · Part 3 (46–70): strings, KMP, parsing

### [02 — Hashing](02-hashing.md) · 14 full-ladder entries
Longest Consecutive Sequence · Subarray Sum Equals K · LRU/LFU Cache · Insert Delete GetRandom · 4Sum II · Isomorphic Strings · Find All Anagrams · Copy List w/ Random Pointer · Design HashMap · Consistent Hashing

### [03 — Two Pointers & Sliding Window](03-two-pointers-sliding-window.md) · 16 full-ladder entries
Two Sum II · 3Sum family · Container/Trapping Water · Longest Substring K Distinct · Min Window Substring · Sliding Window Maximum · Subarrays w/ K Different · Floyd's Cycle Detection

### [04 — Linked Lists](04-linked-lists.md) · 20 full-ladder entries
Reverse (+ II, k-Group) · Merge Two/k Lists · Remove Nth From End · Palindrome/Reorder List · Intersection of Lists · Add Two Numbers (I/II) · Sort List (merge sort) · Flatten Multilevel List

### [05 — Stacks & Queues](05-stacks-queues.md) · 20 full-ladder entries
Min Stack · Queue/Stack from each other · Next Greater Element · Largest Rectangle in Histogram · Remove K Digits · 132 Pattern · Basic Calculator · Asteroid Collision · Design Circular Queue

### [06 — Trees](06-trees.md) · 20 full-ladder entries
Max Depth/Balanced/Diameter · Level Order family · Validate BST · Kth Smallest in BST · LCA (BST + general) · Construct from Traversals · Serialize/Deserialize · Path Sum I/II/III · Morris Traversal · Trie · Segment/Fenwick Tree

### [07 — Heaps & Intervals](07-heaps-intervals.md) · 20 full-ladder entries
Kth Largest (quickselect) · Find Median from Stream · Kth Smallest in Sorted Matrix · Task Scheduler · IPO · Employee Free Time · My Calendar · The Skyline Problem · Interval List Intersections · Min Taps/Jump Game II

### [08 — Graphs](08-graphs.md) · 25 full-ladder entries
Number of Islands · Surrounded Regions · Rotting Oranges · Word Ladder · Course Schedule I/II · Clone Graph · Union-Find · Network Delay Time (Dijkstra) · Cheapest Flights (Bellman-Ford) · MST (Kruskal/Prim) · Critical Connections (Tarjan) · Bipartite Check · Reconstruct Itinerary (Hierholzer)

### [09 — Dynamic Programming](09-dynamic-programming.md) · 28 full-ladder entries
Climbing Stairs · House Robber I/II · Coin Change I/II · 0/1 Knapsack family · LIS (O(n log n)) · LCS/Edit Distance family · Unique Paths · Maximal Square · Stock Buy/Sell state machine (I–IV, cooldown) · Burst Balloons · Regex/Wildcard Matching · Dungeon Game · Longest Increasing Path

### [10 — Greedy, Backtracking, Bits & Math](10-greedy-backtracking-misc.md) · 25 full-ladder entries
Jump Game · Gas Station · Candy · Permutations/Combinations/Subsets (+ dup variants) · N-Queens · Sudoku Solver · Generate Parentheses · Single Number family (XOR) · Pow(x,n) · GCD/Sieve of Eratosthenes · Random Pick with Weight · Majority Element (Boyer-Moore)

---

## 🗓️ Practice Sets

Copy one of these into your [progress tracker](../00-meta/progress-tracker.md).

### Set A — Week 1 warm-up (interleaved, easy→medium)
```
Day 1:  Two Sum · Valid Parentheses · Max Depth of Tree
Day 2:  Best Time Buy/Sell · Reverse Linked List · Contains Duplicate
Day 3:  Merge Sorted Array · Valid Anagram · Climbing Stairs
Day 4:  Product Except Self · Middle of List · Invert Tree
Day 5:  Maximum Subarray · Min Stack · Number of Islands
Day 6:  3Sum · Linked List Cycle · Same Tree
Day 7:  MIXED REVIEW — 5 random from days 1-6, no notes
```

### Set B — Pattern drilling (one pattern per day, 4 problems)
```
Day 1:  Sliding window   — Longest No Repeat · Min Size Subarray · Fruit Baskets · Char Replacement
Day 2:  Two pointers     — 3Sum · Container Water · Trapping Rain · Sorted Squares
Day 3:  Monotonic stack  — Daily Temps · Next Greater II · Largest Rectangle · Remove K Digits
Day 4:  BFS              — Rotting Oranges · Word Ladder · Open the Lock · Binary Matrix Path
Day 5:  DFS/backtrack    — Subsets · Permutations · Word Search · Combination Sum
Day 6:  1D DP            — House Robber · Coin Change · LIS · Decode Ways
Day 7:  MIXED — 6 unlabeled problems
```

### Set C — Company frequency (highest-yield 25)
```
Two Sum · LRU Cache · Number of Islands · Merge Intervals · Longest No Repeat
Trapping Rain Water · Median from Stream · Word Ladder · Course Schedule
Serialize/Deserialize Tree · Max Path Sum · Merge k Lists · Coin Change
Edit Distance · Word Break · Product Except Self · Meeting Rooms II
Min Window Substring · Sliding Window Max · Alien Dictionary · Design Twitter
Kth Largest · Valid Parentheses · Clone Graph · Subsets
```

---

## 📊 Difficulty Distribution

```
   🟢 Easy     ████████████                    ~95   (22%)
   🟡 Medium   ██████████████████████████████ ~245   (57%)
   🔴 Hard     ████████████                    ~90   (21%)
                                             ─────
                                              430
```

```
   BY PATTERN

   Dynamic Programming   ████████████████████████  60
   Graphs                ████████████████████      50
   Trees                 ████████████████████      50
   Arrays & Strings      ████████████████████████████  70
   Greedy/Backtrack/Bits ██████████████████        45
   Two Ptr / Window      ████████████████          40
   Hashing               █████████████             33
   Heaps & Intervals     ████████████              32
   Stacks & Queues       ████████████              32
   Linked Lists          ███████████               28
```

---

## ✅ How to Use This Index

1. **Stuck on a problem?** → Reverse Lookup table at the top.
2. **Building a practice session?** → grab a Practice Set, or pick one problem from each of three different books (interleaving beats blocking).
3. **Post-mortem after failing a problem?** → find it here, note the *pattern* you missed in your [tracker](../00-meta/progress-tracker.md), and schedule a re-test in 7 days.
4. **Two weeks before an interview?** → Set C only.

⭐ = highest-frequency / highest-teaching-value problems. If short on time, do only those.

---

**Back to:** [Patterns & Foundations](00-patterns.md) · [Library Home](../README.md)
