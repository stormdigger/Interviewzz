# 📇 Master Problem Index

> All 430+ problems, cross-referenced by pattern, difficulty, and frequency. Use this to build practice sets and to look up "what pattern is this?"

---

## 🔍 Reverse Lookup — "What Pattern Is This?"

Find the phrase closest to your problem statement.

| The problem says... | Reach for | Book |
|---|---|---|
| "sorted array" + "find a pair/triplet" | Two pointers from both ends | [03](03-two-pointers-sliding-window.md) |
| "contiguous subarray/substring" + max/min length | Sliding window | [03](03-two-pointers-sliding-window.md) |
| "subarray sum equals k" + **negatives possible** | Prefix sum + hash map | [01](01-arrays-strings.md#14-subarray-sum-equals-k-) |
| "subarray sum equals k" + all positive | Sliding window | [03](03-two-pointers-sliding-window.md) |
| "exactly K distinct/odd/…" | `atMost(K) − atMost(K−1)` | [03](03-two-pointers-sliding-window.md#29-subarrays-with-k-different-integers-) |
| "top k" / "k-th largest" | Heap of size k, or quickselect | [07](07-heaps-intervals.md) |
| "median of a stream" | Two heaps | [07](07-heaps-intervals.md#8-find-median-from-data-stream-) |
| "merge k sorted …" | Heap, or divide & conquer | [07](07-heaps-intervals.md) |
| "next/previous greater/smaller" | Monotonic stack | [05](05-stacks-queues.md) |
| "largest rectangle" / "trapping water" | Monotonic stack | [05](05-stacks-queues.md#16-largest-rectangle-in-histogram-) |
| "max/min of every window of size k" | Monotonic deque | [05](05-stacks-queues.md#26-sliding-window-maximum-) |
| "intervals" + merge/insert/intersect | Sort by **start** | [07](07-heaps-intervals.md) |
| "maximum non-overlapping" / "min arrows" | Sort by **end** (activity selection) | [07](07-heaps-intervals.md#23-non-overlapping-intervals-) |
| "minimum rooms" / "max concurrent" | Sweep line (+1/−1 events) | [07](07-heaps-intervals.md#22-meeting-rooms-ii-) |
| "cycle in a linked list" / "find the middle" | Fast & slow pointers | [04](04-linked-lists.md) |
| "reverse a list" / "reorder" | In-place reversal | [04](04-linked-lists.md) |
| "level by level" / "shortest in unweighted" | BFS with `int sz = q.size()` | [08](08-graphs.md) |
| "distance from **many** sources" | Multi-source BFS | [08](08-graphs.md#3-rotting-oranges-) |
| "shortest path" + weights ≥ 0 | Dijkstra | [08](08-graphs.md#33-network-delay-time-dijkstra-) |
| "shortest path" + at most k edges | Bellman-Ford (copy dist per round) | [08](08-graphs.md#34-cheapest-flights-within-k-stops-) |
| "prerequisites" / "build order" | Topological sort (Kahn's) | [08](08-graphs.md#19-course-schedule-) |
| "connected components" + **dynamic** edges | Union-Find | [08](08-graphs.md#d-union-find) |
| "minimum spanning tree" / "connect all at min cost" | Kruskal (DSU) or Prim (heap) | [08](08-graphs.md) |
| "count ways" / "min cost" + choices repeat | DP | [09](09-dynamic-programming.md) |
| "two strings" + match/transform | 2D DP `dp[i][j]` | [09](09-dynamic-programming.md#d-two-sequence-dp) |
| "pick items with a capacity limit" | Knapsack (mind loop direction!) | [09](09-dynamic-programming.md#c-knapsack-family) |
| "burst/merge within a range" | Interval DP (loop by length) | [09](09-dynamic-programming.md#e-interval-dp) |
| "subset of ≤20 items" | Bitmask DP | [09](09-dynamic-programming.md#f-bitmask-dp) |
| "generate all …" / "all combinations" | Backtracking | [10](10-greedy-backtracking-misc.md#part-2--backtracking) |
| "minimum/maximum" + a provable local choice | Greedy (**prove it!**) | [10](10-greedy-backtracking-misc.md#part-1--greedy) |
| "appears once, others twice" | XOR everything | [10](10-greedy-backtracking-misc.md#33-single-number-) |
| "prefix" / "autocomplete" / "dictionary" | Trie | [00](00-patterns.md#pattern-17--trie) |
| "minimum X such that f(X) works" | **Binary search on the answer** | [00](00-patterns.md#pattern-12--binary-search) |
| "values are 1..n, find missing/duplicate" | Cyclic sort or sign marking | [01](01-arrays-strings.md#40-first-missing-positive-) |

---

## ⭐ The Blind 75 Core — Start Here

If you only have time for 40 problems, these are the ones. Each unlocks a whole family.

| # | Problem | Pattern | Book |
|---|---|---|---|
| 1 | Two Sum | Hash map complement | [02](02-hashing.md#1-two-sum-) |
| 2 | Best Time to Buy/Sell Stock | Running min | [01](01-arrays-strings.md#2-best-time-to-buy-and-sell-stock-) |
| 3 | Product Except Self | Prefix × suffix | [01](01-arrays-strings.md#4-product-of-array-except-self-) |
| 4 | Maximum Subarray | Kadane | [01](01-arrays-strings.md#5-maximum-subarray-kadane-) |
| 5 | Maximum Product Subarray | Track max **and** min | [01](01-arrays-strings.md#6-maximum-product-subarray-) |
| 6 | Find Min in Rotated Sorted | Binary search | [01](01-arrays-strings.md#7-find-minimum-in-rotated-sorted-array-) |
| 7 | Search in Rotated Sorted | Binary search, one half sorted | [01](01-arrays-strings.md#8-search-in-rotated-sorted-array-) |
| 8 | 3Sum | Sort + two pointers | [03](03-two-pointers-sliding-window.md#2-3sum-) |
| 9 | Container With Most Water | Two pointers, move shorter | [03](03-two-pointers-sliding-window.md#6-container-with-most-water-) |
| 10 | Trapping Rain Water | Two pointers / monotonic stack | [03](03-two-pointers-sliding-window.md#7-trapping-rain-water-) |
| 11 | Longest Substring No Repeat | Sliding window | [03](03-two-pointers-sliding-window.md#20-longest-substring-without-repeating-characters-) |
| 12 | Longest Repeating Char Replace | Sliding window + maxCount | [03](03-two-pointers-sliding-window.md#21-longest-repeating-character-replacement-) |
| 13 | Minimum Window Substring | Sliding window + missing counter | [03](03-two-pointers-sliding-window.md#22-minimum-window-substring-) |
| 14 | Group Anagrams | Count-array key | [02](02-hashing.md#3-group-anagrams-) |
| 15 | Longest Consecutive Sequence | Set + start-of-run guard | [02](02-hashing.md#14-longest-consecutive-sequence-) |
| 16 | LRU Cache | Hash map + doubly linked list | [02](02-hashing.md#19-lru-cache-) |
| 17 | Reverse Linked List | 3-pointer reversal | [04](04-linked-lists.md#1-reverse-linked-list-) |
| 18 | Linked List Cycle II | Floyd + restart from head | [04](04-linked-lists.md#8-linked-list-cycle-ii-find-the-entry-) |
| 19 | Merge k Sorted Lists | Heap / divide & conquer | [04](04-linked-lists.md#13-merge-k-sorted-lists-) |
| 20 | Valid Parentheses | Stack | [05](05-stacks-queues.md#1-valid-parentheses-) |
| 21 | Daily Temperatures | Monotonic stack | [05](05-stacks-queues.md#15-daily-temperatures-) |
| 22 | Largest Rectangle Histogram | Monotonic stack | [05](05-stacks-queues.md#16-largest-rectangle-in-histogram-) |
| 23 | Sliding Window Maximum | Monotonic deque | [05](05-stacks-queues.md#26-sliding-window-maximum-) |
| 24 | Maximum Depth of Binary Tree | Tree recursion | [06](06-trees.md#9-maximum-depth-) |
| 25 | Diameter of Binary Tree | ⭐ both children vs one | [06](06-trees.md#12-diameter-of-binary-tree-) |
| 26 | Binary Tree Max Path Sum | Same pattern as diameter | [06](06-trees.md#22-binary-tree-maximum-path-sum-) |
| 27 | Validate BST | Pass bounds down | [06](06-trees.md#27-validate-bst-) |
| 28 | Lowest Common Ancestor | Return "found or LCA" | [06](06-trees.md#33-lowest-common-ancestor-of-a-binary-tree-) |
| 29 | Serialize/Deserialize Tree | Preorder + null markers | [06](06-trees.md#44-serialize-and-deserialize-binary-tree-) |
| 30 | Number of Islands | Flood fill | [08](08-graphs.md#1-number-of-islands-) |
| 31 | Rotting Oranges | Multi-source BFS | [08](08-graphs.md#3-rotting-oranges-) |
| 32 | Course Schedule | Topological sort | [08](08-graphs.md#19-course-schedule-) |
| 33 | Word Ladder | Implicit graph BFS | [08](08-graphs.md#13-word-ladder-) |
| 34 | Network Delay Time | Dijkstra | [08](08-graphs.md#33-network-delay-time-dijkstra-) |
| 35 | Climbing Stairs | 1D DP intro | [09](09-dynamic-programming.md#1-climbing-stairs-) |
| 36 | House Robber | Rob vs skip | [09](09-dynamic-programming.md#2-house-robber-) |
| 37 | Coin Change | Unbounded knapsack | [09](09-dynamic-programming.md#19-coin-change-minimum-coins-) |
| 38 | Longest Increasing Subsequence | DP + patience O(n log n) | [09](09-dynamic-programming.md#5-longest-increasing-subsequence-) |
| 39 | Edit Distance | 2D DP, 3 operations | [09](09-dynamic-programming.md#29-edit-distance-) |
| 40 | Word Break | DP + dictionary set | [09](09-dynamic-programming.md#9-word-break-) |

---

## 📚 Full Index by Book

### [01 — Arrays & Strings](01-arrays-strings.md) · 70 problems

**A. Fundamentals (1-12)**
Two Sum 🟢 · Best Time Buy/Sell 🟢 · Contains Duplicate 🟢 · Product Except Self 🟡 · Maximum Subarray 🟡 · Maximum Product Subarray 🟡 · Find Min Rotated 🟡 · Search Rotated 🟡 · Search Rotated II 🟡 · First/Last Position 🟡 · Missing Number 🟢 · Single Number 🟢

**B. Prefix Sum & Ranges (13-24)**
Range Sum Immutable 🟢 · Subarray Sum = K 🟡 · Continuous Subarray Sum 🟡 · Subarrays Div by K 🟡 · Contiguous Array 🟡 · Max Size Subarray Sum K 🟡 · Range Addition 🟡 · Corp Flight Bookings 🟡 · Car Pooling 🟡 · Find Pivot Index 🟢 · Range Sum 2D 🟡 · Max Sum Rectangle 🔴

**C. Sorting-Based (25-34)**
Merge Sorted Array 🟢 · Sort Colors 🟡 · Merge Intervals 🟡 · Insert Interval 🟡 · Non-overlapping Intervals 🟡 · Meeting Rooms II 🟡 · Largest Number 🟡 · Sort by Parity 🟢 · H-Index 🟡 · Top K Frequent 🟡

**D. In-Place (35-44)**
Remove Duplicates 🟢 · Remove Duplicates II 🟡 · Remove Element 🟢 · Move Zeroes 🟢 · Rotate Array 🟡 · First Missing Positive 🔴 · Find All Duplicates 🟡 · Find Disappeared 🟢 · Set Matrix Zeroes 🟡 · Next Permutation 🟡

**E. Matrix (45-54)**
Rotate Image 🟡 · Spiral Matrix 🟡 · Spiral Matrix II 🟡 · Search 2D Matrix 🟡 · Search 2D Matrix II 🟡 · Diagonal Traverse 🟡 · Game of Life 🟡 · Transpose 🟢 · Valid Sudoku 🟡 · Diagonal Sum 🟢

**F. Strings (55-70)**
Valid Anagram 🟢 · Group Anagrams 🟡 · Valid Palindrome 🟢 · Valid Palindrome II 🟢 · Longest Common Prefix 🟢 · String to Integer 🟡 · strStr/KMP 🟡 · Longest Palindromic Substring 🟡 · Palindromic Substrings 🟡 · Reverse Words 🟡 · Zigzag Conversion 🟡 · Compare Versions 🟡 · Multiply Strings 🟡 · Add Strings 🟢 · Text Justification 🔴 · Encode/Decode Strings 🟡

---

### [02 — Hashing](02-hashing.md) · 33 problems
Two Sum 🟢 · Valid Anagram 🟢 · Group Anagrams 🟡 · Top K Frequent 🟡 · Top K Frequent Words 🟡 · First Unique Char 🟢 · Sort Chars by Freq 🟡 · Find All Anagrams 🟡 · Ransom Note 🟢 · Isomorphic Strings 🟢 · Word Pattern 🟢 · Contains Duplicate II 🟢 · Contains Duplicate III 🔴 · Longest Consecutive 🟡 · Happy Number 🟢 · Intersection 🟢 · Intersection II 🟢 · Valid Sudoku 🟡 · **LRU Cache** 🟡 · **LFU Cache** 🔴 · Insert/Delete/GetRandom 🟡 · …with Duplicates 🔴 · Design HashMap 🟢 · Design Twitter 🟡 · Underground System 🟡 · Time-Based KV Store 🟡 · Logger Rate Limiter 🟢 · Hit Counter 🟡 · Longest Substring No Repeat 🟡 · Subarray Sum = K 🟡 · **4Sum II** 🟡 · Copy List with Random 🟡 · Word Break 🟡

---

### [03 — Two Pointers & Sliding Window](03-two-pointers-sliding-window.md) · 40 problems

**A. Opposite Ends (1-12)**
Two Sum II 🟡 · 3Sum 🟡 · 3Sum Closest 🟡 · 3Sum Smaller 🟡 · 4Sum 🟡 · Container Most Water 🟡 · **Trapping Rain Water** 🔴 · Valid Palindrome 🟢 · Reverse Vowels 🟢 · Sorted Squares 🟢 · Boats to Save People 🟡 · Partition Labels 🟡

**B. Same Direction (13-19)**
Remove Duplicates 🟢 · Move Zeroes 🟢 · Sort Colors 🟡 · Backspace Compare 🟢 · Is Subsequence 🟢 · Merge Sorted Array 🟢 · Interval Intersections 🟡

**C. Variable Window (20-34)**
Longest No Repeat 🟡 · Longest Repeating Replace 🟡 · **Minimum Window Substring** 🔴 · Permutation in String 🟡 · Find All Anagrams 🟡 · At Most K Distinct 🟡 · Fruit Into Baskets 🟡 · Min Size Subarray Sum 🟡 · Max Consecutive Ones III 🟡 · **K Different Integers** 🔴 · Nice Subarrays 🟡 · Product Less Than K 🟡 · Longest 1s After Delete 🟡 · Reduce X to Zero 🟡 · Max Points from Cards 🟡

**D. Fixed Window (35-40)**
Max Average Subarray 🟢 · **Sliding Window Maximum** 🔴 · Sliding Window Median 🔴 · Repeated DNA 🟡 · Minimum Window Subsequence 🔴 · At Least K Repeating 🟡

---

### [04 — Linked Lists](04-linked-lists.md) · 28 problems
Reverse List 🟢 · Reverse List II 🟡 · **Reverse in k-Group** 🔴 · Swap Pairs 🟡 · Palindrome List 🟢 · Middle of List 🟢 · Cycle 🟢 · **Cycle II** 🟡 · Remove Nth From End 🟡 · Happy Number 🟢 · **Find Duplicate Number** 🟡 · Merge Two Lists 🟢 · **Merge k Lists** 🔴 · Sort List 🟡 · Insertion Sort List 🟡 · Remove Duplicates 🟢 · Remove Duplicates II 🟡 · Remove Elements 🟢 · Partition List 🟡 · Odd Even List 🟡 · Rotate List 🟡 · **Reorder List** 🟡 · Add Two Numbers 🟡 · Add Two Numbers II 🟡 · **Copy List with Random** 🟡 · Flatten Multilevel 🟡 · LRU Cache 🟡 · Intersection of Lists 🟢

---

### [05 — Stacks & Queues](05-stacks-queues.md) · 32 problems

**A. Classic (1-12)**
Valid Parentheses 🟢 · Min Stack 🟡 · Eval RPN 🟡 · Basic Calculator 🔴 · Basic Calculator II 🟡 · Decode String 🟡 · Simplify Path 🟡 · Remove Adjacent Dups 🟢 · …II 🟡 · Min Remove Valid Parens 🟡 · Score of Parentheses 🟡 · **Longest Valid Parentheses** 🔴

**B. Monotonic Stack (13-23)**
Next Greater I 🟢 · Next Greater II 🟡 · **Daily Temperatures** 🟡 · **Largest Rectangle** 🔴 · **Maximal Rectangle** 🔴 · Trapping Rain Water 🔴 · Sum of Subarray Mins 🟡 · Remove K Digits 🟡 · Create Max Number 🔴 · Remove Duplicate Letters 🟡 · 132 Pattern 🟡

**C. Queues & Deques (24-30)**
Queue from Stacks 🟢 · Stack from Queues 🟢 · **Sliding Window Maximum** 🔴 · **Shortest Subarray Sum ≥ K** 🔴 · Circular Queue 🟡 · Hit Counter 🟡 · Asteroid Collision 🟡 · Exclusive Time 🟡 · Basic Calculator III 🔴

---

### [06 — Trees](06-trees.md) · 50 problems

**A. Traversal (1-8)**
Inorder 🟢 · Preorder/Postorder Iterative 🟡 · **Morris Traversal** 🔴 · Level Order 🟡 · Zigzag 🟡 · Right Side View 🟡 · Vertical Order 🔴 · Average of Levels 🟢

**B. Depth & Structure (9-18)**
Max Depth 🟢 · Min Depth 🟢 · Balanced Tree 🟢 · **Diameter** 🟢 · Same Tree 🟢 · Symmetric 🟢 · Subtree of Another 🟢 · Invert 🟢 · Merge Trees 🟢 · Count Complete Nodes 🟡

**C. Paths (19-26)**
Path Sum 🟢 · Path Sum II 🟡 · **Path Sum III** 🟡 · **Max Path Sum** 🔴 · Sum Root to Leaf 🟡 · Binary Tree Paths 🟢 · Longest Univalue Path 🟡 · Distance K 🟡

**D. BST (27-40)**
**Validate BST** 🟡 · Search 🟢 · Insert 🟡 · Delete 🟡 · Kth Smallest 🟡 · LCA of BST 🟢 · **LCA General** 🟡 · Sorted Array→BST 🟢 · Sorted List→BST 🟡 · BST Iterator 🟡 · Recover BST 🔴 · Range Sum 🟢 · Min Abs Difference 🟢 · Two Sum IV 🟢

**E. Construction (41-47)**
From Preorder+Inorder 🟡 · From Inorder+Postorder 🟡 · BST from Preorder 🟡 · **Serialize/Deserialize** 🔴 · Serialize BST 🟡 · Flatten to List 🟡 · Populate Next Right 🟡

**F. Advanced (48-50)**
Binary Tree Cameras 🔴 · House Robber III 🟡 · Distribute Coins 🟡

---

### [07 — Heaps & Intervals](07-heaps-intervals.md) · 32 problems

**A. Top-K (1-7)**
Kth Largest 🟡 · Top K Frequent 🟡 · K Closest Points 🟡 · Kth Largest in Stream 🟢 · Sort by Frequency 🟡 · Kth Smallest in Matrix 🟡 · K Pairs Smallest Sums 🟡

**B. Two Heaps (8-10)**
**Median from Stream** 🔴 · Sliding Window Median 🔴 · IPO 🔴

**C. Merging & Scheduling (11-18)**
Merge k Lists 🔴 · **Smallest Range** 🔴 · Task Scheduler 🟡 · Reorganize String 🟡 · Rearrange k Distance 🔴 · Connect Sticks 🟡 · Last Stone Weight 🟢 · Ugly Number II 🟡

**D. Intervals (19-30)**
Merge Intervals 🟡 · Insert Interval 🟡 · Meeting Rooms 🟢 · **Meeting Rooms II** 🟡 · Non-overlapping 🟡 · Min Arrows 🟡 · Interval Intersections 🟡 · Employee Free Time 🔴 · My Calendar I 🟡 · II 🟡 · III 🔴 · Data Stream Disjoint Intervals 🔴 · Car Pooling 🟡 · **The Skyline Problem** 🔴

---

### [08 — Graphs](08-graphs.md) · 50 problems

**A. Grid (1-12)**
**Number of Islands** 🟡 · Max Area Island 🟡 · **Rotting Oranges** 🟡 · 01 Matrix 🟡 · Surrounded Regions 🟡 · Pacific Atlantic 🟡 · Word Search 🟡 · **Word Search II** 🔴 · Distinct Islands 🟡 · Shortest Path Binary Matrix 🟡 · Walls and Gates 🟡 · Number of Enclaves 🟡

**B. BFS Shortest Path (13-18)**
**Word Ladder** 🔴 · Word Ladder II 🔴 · Open the Lock 🟡 · Min Genetic Mutation 🟡 · Jump Game IV 🟡 · Shortest Path w/ Obstacles 🔴

**C. Topological Sort (19-24)**
**Course Schedule** 🟡 · Course Schedule II 🟡 · **Alien Dictionary** 🔴 · Min Height Trees 🟡 · Sequence Reconstruction 🟡 · Parallel Courses 🟡

**D. Union-Find (25-32)**
Connected Components 🟡 · Graph Valid Tree 🟡 · Redundant Connection 🟡 · Accounts Merge 🟡 · **Islands II** 🔴 · Most Stones Removed 🟡 · Equality Equations 🟡 · Evaluate Division 🟡

**E. Weighted Shortest Path (33-39)**
**Network Delay Time** 🟡 · **Cheapest Flights K Stops** 🟡 · Max Probability Path 🟡 · Min Effort Path 🟡 · Swim in Rising Water 🔴 · Negative Cycle 🔴 · Floyd-Warshall City 🟡

**F. MST & Advanced (40-50)**
Connect Points (Prim) 🟡 · Connecting Cities (Kruskal) 🟡 · **Critical Connections** 🔴 · Clone Graph 🟡 · Course Schedule IV 🟡 · Is Bipartite 🟡 · Possible Bipartition 🟡 · Eventual Safe States 🟡 · **Reconstruct Itinerary** 🔴 · Min Vertices Reach All 🟡 · Snakes and Ladders 🟡

---

### [09 — Dynamic Programming](09-dynamic-programming.md) · 60 problems

**A. 1D Linear (1-12)**
Climbing Stairs 🟢 · **House Robber** 🟡 · House Robber II 🟡 · Min Cost Stairs 🟢 · **LIS** 🟡 · Maximum Subarray 🟡 · Max Product Subarray 🟡 · Decode Ways 🟡 · Word Break 🟡 · Jump Game 🟡 · Jump Game II 🟡 · Stock w/ Cooldown 🟡

**B. 2D Grid (13-18)**
Unique Paths 🟡 · Unique Paths II 🟡 · Min Path Sum 🟡 · Triangle 🟡 · **Maximal Square** 🟡 · Dungeon Game 🔴

**C. Knapsack (19-27)**
**Coin Change** 🟡 · Coin Change II 🟡 · Combination Sum IV 🟡 · **Partition Equal Subset** 🟡 · Target Sum 🟡 · Last Stone Weight II 🟡 · Ones and Zeroes 🟡 · Perfect Squares 🟡 · Integer Break 🟡

**D. Two Sequences (28-35)**
**LCS** 🟡 · **Edit Distance** 🔴 · Distinct Subsequences 🔴 · Interleaving String 🟡 · **Regex Matching** 🔴 · Wildcard Matching 🔴 · Longest Palindromic Subseq 🟡 · Shortest Common Supersequence 🔴

**E. Interval (36-40)**
**Burst Balloons** 🔴 · Matrix Chain 🔴 · Palindrome Partitioning II 🔴 · Stone Game 🟡 · Min Cost Cut Stick 🔴

**F. Bitmask (41-43)**
TSP (Held-Karp) 🔴 · Partition K Subsets 🔴 · Shortest Path All Nodes 🔴

**G. Tree & Digit (44-47)**
House Robber III 🟡 · Max Path Sum 🔴 · Unique Digits 🟡 · Digit DP 🔴

**H. Stock & Game (48-54)**
Stock I 🟢 · II 🟡 · III 🔴 · IV 🔴 · w/ Fee 🟡 · Predict Winner 🟡 · Can I Win 🟡

**I. Classics (55-60)**
Longest Increasing Path 🔴 · Russian Doll Envelopes 🔴 · Job Scheduling 🔴 · Frog Jump 🔴 · Count Palindromic Subseq 🔴 · Cherry Pickup 🔴

---

### [10 — Greedy, Backtracking, Bits, Math](10-greedy-backtracking-misc.md) · 45 problems

**Greedy (1-15)**
Jump Game 🟡 · Jump Game II 🟡 · **Gas Station** 🟡 · Task Scheduler 🟡 · Partition Labels 🟡 · Non-overlapping Intervals 🟡 · Min Arrows 🟡 · Two City Scheduling 🟡 · Boats 🟡 · **Candy** 🔴 · Queue Reconstruction 🟡 · Merge Triplets 🟡 · Valid Parenthesis String 🟡 · Min Deletions Unique Freq 🟡 · Hand of Straights 🟡

**Backtracking (16-32)**
**Subsets** 🟡 · Subsets II 🟡 · **Permutations** 🟡 · Permutations II 🟡 · Combinations 🟡 · **Combination Sum** 🟡 · II 🟡 · III 🟡 · Letter Combinations 🟡 · **Generate Parentheses** 🟡 · Word Search 🟡 · Palindrome Partitioning 🟡 · **N-Queens** 🔴 · Sudoku Solver 🔴 · Restore IP 🟡 · Word Break II 🔴 · Beautiful Arrangement 🟡

**Bits (33-42)**
Single Number 🟢 · II 🟡 · **III** 🟡 · Number of 1 Bits 🟢 · Counting Bits 🟢 · Reverse Bits 🟢 · Missing Number 🟢 · Sum Without + 🟡 · **Max XOR** 🟡 · Subsets via Bitmask 🟡

**Math (43-45)**
**Pow(x,n)** 🟡 · Sqrt(x) 🟢 · **Count Primes** 🟡 · GCD/LCM · Modular Arithmetic

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
