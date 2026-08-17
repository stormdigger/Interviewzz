# 🧮 Dynamic Programming — 60 Problems

> DP scares people because it's taught backwards. Nobody invents a table out of thin air. **You write the obvious brute-force recursion, notice it recomputes the same things, and cache them.** That's it. This book teaches it in that order.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 What DP Actually Is

#### 💬 The plain-language version

Suppose I ask you to compute `fib(50)` with the obvious recursion. You'd write `fib(n) = fib(n-1) + fib(n-2)`. Correct — but catastrophically slow. Why?

```
                        fib(5)
                    ┌─────┴─────┐
                 fib(4)       fib(3)      ← fib(3) appears here
              ┌────┴───┐    ┌───┴───┐
           fib(3)   fib(2) fib(2) fib(1)  ← and AGAIN here
          ┌──┴──┐
       fib(2) fib(1)                       ← fib(2) computed 3 separate times

   fib(50) recomputes fib(30) about 800,000 times.
```

You're solving the same subproblem again and again. The fix is embarrassingly simple: **write the answer down the first time, and look it up after that.** That's dynamic programming. Everything else is bookkeeping.

#### The two requirements

```
   ┌──────────────────────────────────────────────────────────────┐
   │ 1. OPTIMAL SUBSTRUCTURE                                      │
   │    The best answer for a big problem is built from           │
   │    best answers to smaller problems.                         │
   │                                                              │
   │    ✅ Shortest path: best route A→C through B                 │
   │       = best A→B + best B→C                                  │
   │    ❌ LONGEST simple path: doesn't work — the best A→B        │
   │       might use up nodes that B→C needed                     │
   ├──────────────────────────────────────────────────────────────┤
   │ 2. OVERLAPPING SUBPROBLEMS                                   │
   │    The same subproblem comes up more than once.              │
   │                                                              │
   │    If subproblems DON'T overlap, caching helps nothing —     │
   │    that's just divide & conquer (e.g. merge sort).           │
   └──────────────────────────────────────────────────────────────┘
```

### The recipe — follow it every single time

```
   ┌─ STEP 1 ─────────────────────────────────────────────────────┐
   │ Write the BRUTE FORCE recursion. Don't think about DP yet.    │
   │ Just: "to solve this, what smaller problems do I need?"       │
   ├─ STEP 2 ─────────────────────────────────────────────────────┤
   │ Identify the STATE — what arguments actually change?          │
   │ Those become your table dimensions.                           │
   │   one changing value  → dp[i]                                 │
   │   two changing values → dp[i][j]                              │
   ├─ STEP 3 ─────────────────────────────────────────────────────┤
   │ Add MEMOIZATION. Same code + a cache array. Now it's DP.      │
   │ ⭐ You can stop here. This is a complete, correct solution.    │
   ├─ STEP 4 ─────────────────────────────────────────────────────┤
   │ (Optional) Flip to BOTTOM-UP. Same recurrence, but fill the   │
   │ table in dependency order with loops instead of recursion.    │
   ├─ STEP 5 ─────────────────────────────────────────────────────┤
   │ (Optional) OPTIMIZE SPACE. If dp[i] only reads dp[i-1],       │
   │ you don't need the whole array — just a couple of variables.  │
   └───────────────────────────────────────────────────────────────┘
```

🎤 **In an interview, say this out loud:** *"Let me start with the brute force recursion, then identify the repeated states and memoize."* That narration alone signals you understand DP rather than having memorized tables.

### Top-down vs bottom-up

```
   TOP-DOWN (memoization)              BOTTOM-UP (tabulation)
   ─────────────────────               ──────────────────────
   Recursion + cache array             Loops filling a table
   Mirrors your natural thinking       Needs you to know the fill order
   Only computes states you NEED       Computes every state
   Recursion stack overhead            No stack, usually faster
   Easier to write correctly ⭐         Easier to space-optimize ⭐

   → Write top-down first. Convert if asked, or if the recursion
     depth is a problem.
```

---

## A. 1D Linear DP

### 1. Climbing Stairs 🟢
> You can climb 1 or 2 steps at a time. How many distinct ways to reach step `n`?

#### 💬 Think of it like this
Stand on step `n` and look backwards. How did you get here? There are only two possibilities: you took a single step from `n-1`, or a double step from `n-2`. There is no third option.

So every way of reaching `n-1` becomes a way of reaching `n` (just add one more step), and likewise for `n-2`. The total is simply the sum:

```
   ways(n) = ways(n-1) + ways(n-2)
```

That's the Fibonacci sequence. The base cases are `ways(0) = 1` (you're already there — one way, do nothing) and `ways(1) = 1`.

#### 📊 Building the table

```
   n:        0    1    2    3    4    5
            ┌────┬────┬────┬────┬────┬────┐
   ways:    │ 1  │ 1  │ 2  │ 3  │ 5  │ 8  │
            └────┴────┴────┴────┴────┴────┘
                   └──┬──┘  ▲
                      └─────┘  each cell = sum of the two before it

   Verify n=3 by hand:  1+1+1,  1+2,  2+1   →  3 ways ✅
```

```cpp
// Space-optimized: we only ever look back two cells, so two variables suffice
int climbStairs(int n) {
    int prev2 = 1, prev1 = 1;
    for (int i = 2; i <= n; ++i) {
        int cur = prev1 + prev2;
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```
**Complexity:** O(n) time, O(1) space.

---

### 2. House Robber 🟡
> Houses in a row, each with money. You can't rob two adjacent houses. Maximize your take.

#### 💬 Think of it like this
Walk up to house `i` and face a genuine choice:

- **Rob it.** You get `nums[i]`, but house `i-1` is now off-limits, so your best possible total is `nums[i] + best(i-2)`.
- **Skip it.** You get nothing here, but house `i-1` was allowed, so your total is `best(i-1)`.

You have no other options, so take whichever is larger. That's the entire algorithm:

```
   best(i) = max( nums[i] + best(i-2),   best(i-1) )
                  └─── rob ───┘          └─ skip ─┘
```

#### 📊 Tracing `[2, 7, 9, 3, 1]`

```
   house:      0     1     2     3     4
   money:      2     7     9     3     1
              ┌─────┬─────┬─────┬─────┬─────┐
   best:      │  2  │  7  │ 11  │ 11  │ 12  │
              └─────┴─────┴─────┴─────┴─────┘

   i=0:  only house → 2
   i=1:  max(rob 7, skip→2)              = 7
   i=2:  max(9 + best(0)=2 → 11,  7)     = 11   ⭐ robbed houses 0 and 2
   i=3:  max(3 + best(1)=7 → 10,  11)    = 11   ⭐ skipped, keeping 0+2
   i=4:  max(1 + best(2)=11 → 12, 11)    = 12   ⭐ robbed 0, 2, and 4

   ANSWER = 12    (houses 0 + 2 + 4  =  2 + 9 + 1)
```

```cpp
int rob(vector<int>& nums) {
    int prev2 = 0;   // best up to i-2
    int prev1 = 0;   // best up to i-1
    for (int x : nums) {
        int cur = max(prev1, prev2 + x);   // skip  vs  rob
        prev2 = prev1;
        prev1 = cur;
    }
    return prev1;
}
```
**Complexity:** O(n) / O(1).

---

### 3. House Robber II (circular street) 🟡
> Same, but the houses form a **circle** — the first and last are now adjacent.

#### 💬 Think of it like this
The circle adds exactly one new constraint: you cannot rob both the first *and* the last house.

Rather than inventing a whole new algorithm, notice that any valid answer falls into one of two cases:
- It **excludes the last house** → the problem is just the linear version on `houses[0 .. n-2]`
- It **excludes the first house** → the linear version on `houses[1 .. n-1]`

Every valid solution fits one of those (a solution robbing neither first nor last is covered by both). So run the linear solver twice and take the better result.

```
   Circle:  [2, 7, 9, 3, 1]

   Case A — drop the last:   [2, 7, 9, 3]      → linear rob → 11
   Case B — drop the first:      [7, 9, 3, 1]  → linear rob → 11

   ANSWER = max(11, 11) = 11
```

```cpp
int robLinear(vector<int>& a, int lo, int hi) {
    int prev2 = 0, prev1 = 0;
    for (int i = lo; i <= hi; ++i) {
        int cur = max(prev1, prev2 + a[i]);
        prev2 = prev1; prev1 = cur;
    }
    return prev1;
}
int rob(vector<int>& nums) {
    int n = nums.size();
    if (n == 1) return nums[0];                    // edge case: circle of one
    return max(robLinear(nums, 0, n - 2),          // exclude last
               robLinear(nums, 1, n - 1));         // exclude first
}
```
**Key insight:** *Reduce a new problem to one you've already solved.* This is one of the most valuable moves in all of problem solving.

---

### 4. Min Cost Climbing Stairs 🟢
```cpp
int minCostClimbingStairs(vector<int>& cost) {
    int prev2 = 0, prev1 = 0;                      // you may start at step 0 or 1 free
    for (int i = 2; i <= (int)cost.size(); ++i) {
        int cur = min(prev1 + cost[i-1],           // arrive from one step below
                      prev2 + cost[i-2]);          // arrive from two steps below
        prev2 = prev1; prev1 = cur;
    }
    return prev1;
}
```

---

### 5. Longest Increasing Subsequence 🟡
> Find the length of the longest strictly increasing subsequence (elements need not be contiguous).

#### 💬 Approach 1 — the O(n²) DP
Define `dp[i]` = *the length of the longest increasing subsequence that **ends exactly at** index `i`*.

That "ends exactly at" framing is the crucial part. It makes the recurrence natural: to extend a subsequence and land on `i`, the previous element must be at some `j < i` with `nums[j] < nums[i]`. Look at every such `j` and take the best.

```
   nums:  [10, 9, 2, 5, 3, 7, 101, 18]
          ┌───┬───┬───┬───┬───┬───┬────┬────┐
   dp:    │ 1 │ 1 │ 1 │ 2 │ 2 │ 3 │  4 │  4 │
          └───┴───┴───┴───┴───┴───┴────┴────┘
                       ▲       ▲
                       │       └─ [2,3,7]  length 3
                       └───────── [2,5]    length 2

   ANSWER = max over the whole dp array = 4   (e.g. 2,3,7,101)
```

```cpp
int lengthOfLIS(vector<int>& nums) {
    int n = nums.size();
    vector<int> dp(n, 1);                          // every element alone is length 1
    int best = 1;
    for (int i = 1; i < n; ++i) {
        for (int j = 0; j < i; ++j)
            if (nums[j] < nums[i]) dp[i] = max(dp[i], dp[j] + 1);
        best = max(best, dp[i]);
    }
    return best;
}
```

#### 💬 Approach 2 — the O(n log n) patience trick

This one is genuinely clever and worth understanding properly.

Keep an array `tails` where `tails[k]` holds **the smallest possible ending value of any increasing subsequence of length k+1** seen so far.

Why the *smallest*? Because a subsequence ending in a small number is easier to extend later. Among all length-3 subsequences, the one ending in 7 is strictly more useful than one ending in 90.

For each new number, binary search for where it belongs:
- If it's bigger than everything → it extends the longest run. Append it.
- Otherwise → it replaces the first element that is ≥ it, improving that length's ending value.

```
   nums = [10, 9, 2, 5, 3, 7, 101, 18]

   10  →  tails = [10]                 new length 1
    9  →  tails = [9]                  9 < 10, better ending for length 1
    2  →  tails = [2]                  better still
    5  →  tails = [2, 5]               extends! now length 2 exists
    3  →  tails = [2, 3]               3 < 5, better ending for length 2
    7  →  tails = [2, 3, 7]            extends! length 3
  101  →  tails = [2, 3, 7, 101]       extends! length 4
   18  →  tails = [2, 3, 7, 18]        18 < 101, better ending for length 4

   ANSWER = tails.size() = 4
```

⚠️ **Important:** `tails` is **not** an actual subsequence — `[2,3,7,18]` happens to be one here, but in general the contents are a mix. Only its *length* is meaningful.

```cpp
int lengthOfLIS(vector<int>& nums) {
    vector<int> tails;
    for (int x : nums) {
        auto it = lower_bound(tails.begin(), tails.end(), x);   // first >= x
        if (it == tails.end()) tails.push_back(x);              // extend
        else *it = x;                                           // improve
    }
    return tails.size();
}
```
**Complexity:** O(n log n) / O(n).

---

### 6. Maximum Subarray (Kadane) 🟡
> Contiguous subarray with the largest sum.

#### 💬 Think of it like this
Walk left to right carrying a running sum. At each element ask one question: **"is the baggage I'm carrying helping me?"**

If your running sum is negative, it is actively hurting — anything you attach it to gets smaller. So drop it and start fresh from the current element. If it's positive, keep it and add the current element.

```
   nums:  [-2,  1, -3,  4, -1,  2,  1, -5,  4]

   i=0   cur = -2                       best = -2
   i=1   max(1, -2+1=-1) = 1  ⭐ restart best =  1
   i=2   max(-3, 1-3=-2) = -2           best =  1
   i=3   max(4, -2+4=2)  = 4  ⭐ restart best =  4
   i=4   max(-1, 4-1=3)  = 3            best =  4
   i=5   max(2, 3+2=5)   = 5            best =  5
   i=6   max(1, 5+1=6)   = 6            best =  6  ⭐ [4,-1,2,1]
   i=7   max(-5, 6-5=1)  = 1            best =  6
   i=8   max(4, 1+4=5)   = 5            best =  6

   ANSWER = 6
```

```cpp
int maxSubArray(vector<int>& nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < (int)nums.size(); ++i) {
        cur = max(nums[i], cur + nums[i]);   // restart  vs  extend
        best = max(best, cur);
    }
    return best;
}
```

---

### 7. Maximum Product Subarray 🟡
> Largest **product** of a contiguous subarray.

#### 💬 Why sums and products differ
With sums, a big negative running total is simply bad. With products, a big **negative** running total is potentially *fantastic* — multiply it by another negative and it becomes a big positive.

So tracking only the maximum is not enough. You must also track the **minimum**, because today's minimum can become tomorrow's maximum. And when the current number is negative, the roles swap.

```
   nums = [2, 3, -2, 4]

   i=0   mx = 2      mn = 2       best = 2
   i=1   mx = 6      mn = 3       best = 6
   i=2   x = -2 is negative → SWAP first: mx=3, mn=6
         mx = max(-2, 3·-2=-6)  = -2
         mn = min(-2, 6·-2=-12) = -12   ⭐ keep this, it could flip
         best = 6
   i=3   mx = max(4, -2·4=-8)   = 4
         mn = min(4, -12·4=-48) = -48
         best = 6

   ANSWER = 6
```

```cpp
int maxProduct(vector<int>& nums) {
    int mx = nums[0], mn = nums[0], best = nums[0];
    for (int i = 1; i < (int)nums.size(); ++i) {
        int x = nums[i];
        if (x < 0) swap(mx, mn);           // ⭐ a negative flips the roles
        mx = max(x, mx * x);
        mn = min(x, mn * x);
        best = max(best, mx);
    }
    return best;
}
```

---

### 8. Decode Ways 🟡
> `'A'`→1 … `'Z'`→26. How many ways can a digit string be decoded?

#### 💬 Think of it like this
Same shape as climbing stairs, but with validity rules. Standing at position `i`, ask:
- Can the single digit `s[i]` be a letter? Only if it isn't `'0'`. If so, add `dp[i-1]`.
- Can the two digits `s[i-1..i]` form a letter? Only if that pair is between 10 and 26. If so, add `dp[i-2]`.

```
   s = "226"

   i=0  ""       dp = 1     (empty string: one way — decode nothing)
   i=1  "2"      dp = 1     "B"
   i=2  "22"     dp = 2     "BB"  or  "V"
   i=3  "226"    dp = 3     single '6' valid → +dp[2]=2  ("BBF","VF")
                            pair "26" valid  → +dp[1]=1  ("BZ")
   ANSWER = 3

   ⚠️ "06" → 0 ways.  A leading zero can never start a letter,
      and "06" is not a valid pair (must be 10–26).
```

```cpp
int numDecodings(string s) {
    if (s.empty() || s[0] == '0') return 0;
    int prev2 = 1, prev1 = 1;                      // dp[0]=1 (empty), dp[1]=1
    for (int i = 1; i < (int)s.size(); ++i) {
        int cur = 0;
        if (s[i] != '0') cur += prev1;                          // single digit
        int two = (s[i-1] - '0') * 10 + (s[i] - '0');
        if (two >= 10 && two <= 26) cur += prev2;               // valid pair
        prev2 = prev1; prev1 = cur;
        if (cur == 0) return 0;                                 // dead end
    }
    return prev1;
}
```

---

### 9. Word Break 🟡
> Can `s` be segmented into a sequence of dictionary words?

#### 💬 Think of it like this
Define `dp[i]` = *"can the first `i` characters be fully segmented?"*

To decide `dp[i]`, try every possible position `j` for the start of the **last** word. If the prefix up to `j` was segmentable (`dp[j]` is true) **and** the chunk `s[j..i)` is in the dictionary, then `dp[i]` is true.

```
   s = "leetcode",  dict = {"leet", "code"}

   index:  0   1   2   3   4   5   6   7   8
           │ l │ e │ e │ t │ c │ o │ d │ e │
           ▲               ▲               ▲
          dp[0]=T        dp[4]=T         dp[8]=T
          (empty)      "leet" ∈ dict   "code" ∈ dict
                       and dp[0]=T     and dp[4]=T

   dp:  [T, F, F, F, T, F, F, F, T]
                                   └── ANSWER = true
```

```cpp
bool wordBreak(string s, vector<string>& dict) {
    unordered_set<string> d(dict.begin(), dict.end());
    int n = s.size();
    vector<bool> dp(n + 1, false);
    dp[0] = true;                                  // empty prefix is segmentable
    for (int i = 1; i <= n; ++i)
        for (int j = 0; j < i; ++j)
            if (dp[j] && d.count(s.substr(j, i - j))) { dp[i] = true; break; }
    return dp[n];
}
```
**Complexity:** O(n²·L).

---

### 10. Jump Game 🟡
```cpp
bool canJump(vector<int>& nums) {
    int reach = 0;
    for (int i = 0; i < (int)nums.size(); ++i) {
        if (i > reach) return false;               // ⭐ can't even get here
        reach = max(reach, i + nums[i]);
    }
    return true;
}
```
**Key insight:** This is *greedy*, not DP — track only the furthest index reachable so far. Mentioning that you recognized DP isn't needed scores points.

---

### 11. Jump Game II (minimum jumps) 🟡
#### 💬 Think of it like this
This is BFS over "levels," done with two pointers instead of a queue. From your current jump range you can reach some furthest point — that entire span is "one jump away." When you exhaust the current range, you've been forced into another jump.

```
   nums = [2, 3, 1, 1, 4]

   ┌──── jump 1 range ────┐
   [2,   3,   1,   1,   4]
    ▲    └────┬────┘
   start   reachable in 1 jump: indices 1..2
           from those, furthest reachable = max(1+3, 2+1) = 4

   ┌──────── jump 2 range ────────┐
   [2,   3,   1,   1,   4]
              └──────┬──────┘
                indices 3..4 — target reached!

   ANSWER = 2
```

```cpp
int jump(vector<int>& nums) {
    int jumps = 0, curEnd = 0, farthest = 0;
    for (int i = 0; i < (int)nums.size() - 1; ++i) {
        farthest = max(farthest, i + nums[i]);
        if (i == curEnd) { ++jumps; curEnd = farthest; }   // ⭐ forced to jump
    }
    return jumps;
}
```

---

### 12. Best Time to Buy and Sell Stock with Cooldown 🟡
#### 💬 State machines in DP
When "what you're allowed to do next depends on what you just did," model it as **states** and track the best value for each.

```
   Three states, and the legal moves between them:

        ┌──────────┐  buy   ┌──────────┐
        │   FREE   │───────▶│   HELD   │
        │ (no stock│        │ (holding │
        │  can buy)│        │  stock)  │
        └────▲─────┘        └────┬─────┘
             │                   │ sell
             │ cooldown ends     ▼
             │              ┌──────────┐
             └──────────────│   SOLD   │
                            │(must rest│
                            │ one day) │
                            └──────────┘
```

```cpp
int maxProfit(vector<int>& prices) {
    int free = 0, held = INT_MIN, sold = 0;
    for (int p : prices) {
        int prevSold = sold;
        sold = held + p;                 // sell today
        held = max(held, free - p);      // keep holding, or buy today
        free = max(free, prevSold);      // stay free, or cooldown finished
    }
    return max(free, sold);              // never end while still holding
}
```

---

## B. 2D Grid DP

### 13. Unique Paths 🟡
> Robot at the top-left of an `m×n` grid can only move right or down. How many paths to the bottom-right?

#### 💬 Think of it like this
Stand on any cell and look backwards. You could only have arrived from **above** or from **the left**. Nowhere else. So the number of ways to reach this cell is simply the sum of the ways to reach those two cells.

Fill the top row and left column with 1 (only one straight-line path to each), then let the sums cascade.

#### 📊 A 3×4 grid filling in

```
        c=0   c=1   c=2   c=3
       ┌─────┬─────┬─────┬─────┐
  r=0  │  1  │  1  │  1  │  1  │   top row: only one way (keep going right)
       ├─────┼─────┼─────┼─────┤
  r=1  │  1  │  2  │  3  │  4  │
       ├─────┼─────┼─────┼─────┤
  r=2  │  1  │  3  │  6  │ 10  │ ← ANSWER
       └─────┴─────┴─────┴─────┘

   How cell (2,2) = 6 is formed:

            (1,2)=3
               │
               ▼
   (2,1)=3 ──▶ (2,2) = 3 + 3 = 6
```

```cpp
int uniquePaths(int m, int n) {
    vector<int> dp(n, 1);                          // the top row
    for (int r = 1; r < m; ++r)
        for (int c = 1; c < n; ++c)
            dp[c] += dp[c-1];                      // ⭐ dp[c] is "above", dp[c-1] is "left"
    return dp[n-1];
}
```
**Space trick:** we only need one row. When we reach `dp[c]`, it still holds the *previous* row's value (that's "above"), and `dp[c-1]` was already updated this row (that's "left"). One array does the job.

---

### 14. Unique Paths II (with obstacles) 🟡
```cpp
int uniquePathsWithObstacles(vector<vector<int>>& g) {
    int m = g.size(), n = g[0].size();
    vector<long long> dp(n, 0);
    dp[0] = (g[0][0] == 0);
    for (int r = 0; r < m; ++r)
        for (int c = 0; c < n; ++c) {
            if (g[r][c] == 1) dp[c] = 0;           // ⭐ obstacle: zero ways through
            else if (c > 0) dp[c] += dp[c-1];
        }
    return dp[n-1];
}
```

---

### 15. Minimum Path Sum 🟡
```cpp
int minPathSum(vector<vector<int>>& g) {
    int m = g.size(), n = g[0].size();
    vector<int> dp(n, INT_MAX);
    dp[0] = 0;
    for (int r = 0; r < m; ++r) {
        dp[0] += g[r][0];                          // leftmost column accumulates
        for (int c = 1; c < n; ++c)
            dp[c] = min(dp[c], dp[c-1]) + g[r][c]; // ⭐ min(from above, from left)
    }
    return dp[n-1];
}
```

---

### 16. Triangle (minimum path sum) 🟡
#### 💬 Why go bottom-up here
Working top-down forces you to handle edge cells specially — the leftmost and rightmost entries of each row have only one parent. Working **bottom-up**, every cell has exactly two children below it, so there are no special cases at all.

```
        [2]                bottom-up:
       [3,4]               row 2:  [6, 5, 7]
      [6,5,7]              row 1:  3+min(6,5)=8   4+min(5,7)=9   → [8, 9]
     [4,1,8,3]             row 0:  2+min(8,9)=10                 → [10]

                           ANSWER = 10   (path 2→3→5→1)
```

```cpp
int minimumTotal(vector<vector<int>>& t) {
    vector<int> dp = t.back();                     // start from the last row
    for (int r = t.size() - 2; r >= 0; --r)
        for (int c = 0; c <= r; ++c)
            dp[c] = t[r][c] + min(dp[c], dp[c+1]);
    return dp[0];
}
```

---

### 17. Maximal Square 🟡
> Largest square of `1`s in a binary matrix.

#### 💬 The key realization
Let `dp[r][c]` = *the side length of the largest square whose **bottom-right corner** is at (r,c)*.

For a square of side `k` to end here, you need three overlapping squares of side `k-1` immediately above, to the left, and diagonally up-left. The limiting one is the smallest — so take `min` of the three and add 1.

```
   Why min of THREE?

   ┌───┬───┐        To place a 3×3 square ending at ●,
   │ ↖ │ ↑ │        all three of these must already
   ├───┼───┤        support a 2×2 square:
   │ ← │ ● │            ↖ up-left    ↑ above    ← left
   └───┴───┘
                    If any one is smaller, it caps you.

   grid              dp
   1 1 1 0           1 1 1 0
   1 1 1 0    →      1 2 2 0
   1 1 1 0           1 2 3 0   ⭐ side 3 → area 9
   0 0 0 0           0 0 0 0
```

```cpp
int maximalSquare(vector<vector<char>>& m) {
    int R = m.size(), C = m[0].size(), best = 0;
    vector<int> dp(C + 1, 0);
    int prevDiag = 0;                              // holds dp[r-1][c-1]
    for (int r = 1; r <= R; ++r) {
        prevDiag = 0;
        for (int c = 1; c <= C; ++c) {
            int tmp = dp[c];                       // save dp[r-1][c] before overwriting
            if (m[r-1][c-1] == '1') {
                dp[c] = min({dp[c], dp[c-1], prevDiag}) + 1;
                best = max(best, dp[c]);
            } else dp[c] = 0;
            prevDiag = tmp;
        }
    }
    return best * best;
}
```

---

### 18. Dungeon Game 🔴
#### 💬 Why this one must run backwards
Your instinct is to go left-to-right tracking accumulated health. That fails, because the *minimum starting health* depends on what's still ahead, not on what's behind. A path that looks great early can require enormous starting health because of a monster at the end.

So compute backwards: `dp[r][c]` = *the minimum health you need **on entering** this cell to survive to the exit*.

```cpp
int calculateMinimumHP(vector<vector<int>>& d) {
    int R = d.size(), C = d[0].size();
    vector<vector<int>> dp(R + 1, vector<int>(C + 1, INT_MAX));
    dp[R][C-1] = dp[R-1][C] = 1;                   // need at least 1 HP at the exit
    for (int r = R - 1; r >= 0; --r)
        for (int c = C - 1; c >= 0; --c) {
            int need = min(dp[r+1][c], dp[r][c+1]) - d[r][c];
            dp[r][c] = max(1, need);               // ⭐ never drop below 1 HP
        }
    return dp[0][0];
}
```

---

## C. Knapsack Family

### The two knapsacks — know the difference cold

```
   0/1 KNAPSACK                    UNBOUNDED KNAPSACK
   Each item used AT MOST ONCE     Each item reusable INFINITELY
   ────────────────────────        ──────────────────────────
   for (item : items)              for (item : items)
     for (w = W; w >= item; --w)     for (w = item; w <= W; ++w)
                    ▲                              ▲
              ⭐ BACKWARD                     ⭐ FORWARD

   WHY the direction flips:

   Backward: when we read dp[w - item], it has NOT been
             touched this round → it's from the PREVIOUS item
             → the item is used at most once. ✅

   Forward:  when we read dp[w - item], it MAY have been
             updated this round → it may already include
             this item → the item can be reused. ✅

   Getting this backwards is the single most common DP bug.
```

### 19. Coin Change (minimum coins) 🟡
> Fewest coins summing to `amount`. Coins are reusable.

#### 💬 Think of it like this
Define `dp[a]` = *fewest coins to make amount `a`*. To make amount `a`, the last coin you added was some coin `c`. Before that you had `a - c`, which took `dp[a-c]` coins. So `dp[a] = dp[a-c] + 1`. Try every coin and keep the best.

```
   coins = [1, 2, 5],  amount = 11

   a:      0   1   2   3   4   5   6   7   8   9  10  11
         ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
   dp:   │ 0 │ 1 │ 1 │ 2 │ 2 │ 1 │ 2 │ 2 │ 3 │ 3 │ 2 │ 3 │
         └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

   dp[11] built from:
     use coin 1 → dp[10] + 1 = 2 + 1 = 3   ⭐ best
     use coin 2 → dp[9]  + 1 = 3 + 1 = 4
     use coin 5 → dp[6]  + 1 = 2 + 1 = 3   ⭐ tie

   ANSWER = 3   (5 + 5 + 1)
```

⚠️ **Greedy fails here.** With coins `[1, 3, 4]` and amount 6, greedy takes 4+1+1 = 3 coins, but the optimum is 3+3 = 2 coins. That's precisely why DP is required.

```cpp
int coinChange(vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, amount + 1);        // amount+1 acts as "infinity"
    dp[0] = 0;
    for (int a = 1; a <= amount; ++a)
        for (int c : coins)
            if (c <= a) dp[a] = min(dp[a], dp[a - c] + 1);
    return dp[amount] > amount ? -1 : dp[amount];
}
```

---

### 20. Coin Change II (count combinations) 🟡
> How many **distinct combinations** make up `amount`? Order doesn't matter.

#### 💬 The loop order decides the meaning — this is the subtle part

```
   COINS OUTER, amount inner  →  COMBINATIONS  (order doesn't matter)
   for (c : coins)                {1,2} and {2,1} counted ONCE
     for (a = c; a <= amount; ++a)

   AMOUNT OUTER, coins inner →  PERMUTATIONS  (order matters)
   for (a = 1; a <= amount; ++a)  {1,2} and {2,1} counted TWICE
     for (c : coins)

   ⭐ Coins-outer means we finish considering coin 1 entirely
      before ever touching coin 2 — so a combination is always
      built in non-decreasing coin order, and can't be recounted.
```

```cpp
int change(int amount, vector<int>& coins) {
    vector<unsigned long long> dp(amount + 1, 0);
    dp[0] = 1;                                     // one way to make 0: take nothing
    for (int c : coins)                            // ⭐ COINS OUTER
        for (int a = c; a <= amount; ++a)
            dp[a] += dp[a - c];
    return dp[amount];
}
```

---

### 21. Combination Sum IV (permutations) 🟡
```cpp
int combinationSum4(vector<int>& nums, int target) {
    vector<unsigned long long> dp(target + 1, 0);
    dp[0] = 1;
    for (int t = 1; t <= target; ++t)              // ⭐ TARGET OUTER → permutations
        for (int x : nums)
            if (x <= t) dp[t] += dp[t - x];
    return dp[target];
}
```
Despite the name "combination," this problem actually counts **permutations** — hence the flipped loop order.

---

### 22. Partition Equal Subset Sum 🟡
> Can the array be split into two subsets of equal sum?

#### 💬 Reduce it to something known
If the total sum is odd, it's immediately impossible. Otherwise, splitting into two equal halves is the same as asking: **"can I pick a subset that sums to exactly `total/2`?"** — because whatever's left over automatically forms the other half.

That's a classic 0/1 knapsack: each number is an item, used at most once.

```cpp
bool canPartition(vector<int>& nums) {
    int total = accumulate(nums.begin(), nums.end(), 0);
    if (total % 2) return false;                   // odd → impossible
    int target = total / 2;

    vector<bool> dp(target + 1, false);
    dp[0] = true;                                  // sum 0 always achievable
    for (int x : nums)
        for (int s = target; s >= x; --s)          // ⭐ BACKWARD (0/1: use once)
            dp[s] = dp[s] || dp[s - x];
    return dp[target];
}
```

---

### 23. Target Sum 🟡
#### 💬 The algebra that turns this into knapsack
You assign `+` or `−` to every number. Let `P` be the numbers you make positive and `N` the ones you make negative.

```
   P − N = target                    and    P + N = total
   ─────────────────────────────────────────────────────
   Add the two equations:  2P = target + total
                            P = (target + total) / 2

   → "Count subsets summing to (target+total)/2"
   → a plain 0/1 knapsack counting problem ✅
```

```cpp
int findTargetSumWays(vector<int>& nums, int target) {
    int total = accumulate(nums.begin(), nums.end(), 0);
    if (abs(target) > total || (target + total) % 2) return 0;
    int s = (target + total) / 2;

    vector<int> dp(s + 1, 0);
    dp[0] = 1;
    for (int x : nums)
        for (int j = s; j >= x; --j)               // ⭐ BACKWARD
            dp[j] += dp[j - x];
    return dp[s];
}
```

---

### 24. Last Stone Weight II 🟡
```cpp
int lastStoneWeightII(vector<int>& stones) {
    int total = accumulate(stones.begin(), stones.end(), 0);
    int half = total / 2;
    vector<bool> dp(half + 1, false);
    dp[0] = true;
    for (int x : stones)
        for (int s = half; s >= x; --s) dp[s] = dp[s] || dp[s - x];
    for (int s = half; s >= 0; --s)
        if (dp[s]) return total - 2 * s;           // ⭐ minimize the gap
    return 0;
}
```
**Key insight:** Smashing stones is really splitting them into two piles; the result is the difference between the piles. Minimizing that difference means getting one pile as close to `total/2` as possible.

---

### 25. Ones and Zeroes (2D knapsack) 🟡
```cpp
int findMaxForm(vector<string>& strs, int m, int n) {
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (auto& s : strs) {
        int zeros = count(s.begin(), s.end(), '0');
        int ones = s.size() - zeros;
        for (int i = m; i >= zeros; --i)           // ⭐ BOTH dimensions backward
            for (int j = n; j >= ones; --j)
                dp[i][j] = max(dp[i][j], dp[i - zeros][j - ones] + 1);
    }
    return dp[m][n];
}
```

---

### 26. Perfect Squares 🟡
```cpp
int numSquares(int n) {
    vector<int> dp(n + 1, INT_MAX);
    dp[0] = 0;
    for (int i = 1; i <= n; ++i)
        for (int j = 1; j * j <= i; ++j)
            dp[i] = min(dp[i], dp[i - j*j] + 1);
    return dp[n];
}
```

---

### 27. Rod Cutting / Integer Break 🟡
```cpp
int integerBreak(int n) {
    vector<int> dp(n + 1, 0);
    dp[1] = 1;
    for (int i = 2; i <= n; ++i)
        for (int j = 1; j < i; ++j)
            dp[i] = max({dp[i], j * (i - j), j * dp[i - j]});
                              //  ▲              ▲
                              //  split in 2     split further
    return dp[n];
}
```

---

## D. Two-Sequence DP

#### 💬 The universal shape
Whenever a problem involves **two strings or arrays**, the state is almost always `dp[i][j]` = *"the answer considering the first `i` characters of A and the first `j` of B."*

The recurrence is a two-way fork:

```
   ┌────────────────────────────────────────────────┐
   │  if A[i-1] == B[j-1]:                          │
   │      characters MATCH → consume both,           │
   │      answer builds on dp[i-1][j-1]              │
   │                                                 │
   │  else:                                          │
   │      no match → try skipping one from A         │
   │      (dp[i-1][j]) or one from B (dp[i][j-1])    │
   │      and take the better/cheaper option         │
   └────────────────────────────────────────────────┘

   The three cells you always read:

        dp[i-1][j-1]  ──▶  dp[i-1][j]
             │                  │
             ▼                  ▼
        dp[i][j-1]    ──▶  dp[i][j]  ← computing this
```

### 28. Longest Common Subsequence 🟡

```
   text1 = "abcde",  text2 = "ace"

         ""   a    c    e
      ┌────┬────┬────┬────┐
   "" │  0 │  0 │  0 │  0 │
      ├────┼────┼────┼────┤
   a  │  0 │  1 │  1 │  1 │   'a'=='a' → 1 + diagonal(0)
      ├────┼────┼────┼────┤
   b  │  0 │  1 │  1 │  1 │   no match → max(left, up)
      ├────┼────┼────┼────┤
   c  │  0 │  1 │  2 │  2 │   'c'=='c' → 1 + diagonal(1)
      ├────┼────┼────┼────┤
   d  │  0 │  1 │  2 │  2 │
      ├────┼────┼────┼────┤
   e  │  0 │  1 │  2 │  3 │   'e'=='e' → 1 + diagonal(2)
      └────┴────┴────┴────┘
                        └── ANSWER = 3  ("ace")
```

```cpp
int longestCommonSubsequence(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j)
            dp[i][j] = (a[i-1] == b[j-1])
                     ? dp[i-1][j-1] + 1                 // match: extend the diagonal
                     : max(dp[i-1][j], dp[i][j-1]);     // no match: best of skipping one
    return dp[m][n];
}
```

---

### 29. Edit Distance 🔴
> Minimum insert / delete / replace operations to turn word A into word B.

#### 💬 Think of it like this
Look at the last character of each word.

- **If they match**, they cost nothing — just align them and solve the smaller problem.
- **If they don't**, you must pay 1 operation, and you have exactly three choices:

```
   Converting  "horse" → "ros"

   ┌──────────────────────────────────────────────────────┐
   │ REPLACE   change A's last char to B's last char      │
   │           → dp[i-1][j-1] + 1     (diagonal)          │
   ├──────────────────────────────────────────────────────┤
   │ DELETE    remove A's last char                       │
   │           → dp[i-1][j]   + 1     (from above)        │
   ├──────────────────────────────────────────────────────┤
   │ INSERT    add B's last char to A                     │
   │           → dp[i][j-1]   + 1     (from the left)     │
   └──────────────────────────────────────────────────────┘

         ""   r    o    s
      ┌────┬────┬────┬────┐
   "" │  0 │  1 │  2 │  3 │  ← turning "" into "ros" = 3 inserts
      ├────┼────┼────┼────┤
   h  │  1 │  1 │  2 │  3 │
      ├────┼────┼────┼────┤
   o  │  2 │  2 │  1 │  2 │  ← 'o'=='o', free diagonal move
      ├────┼────┼────┼────┤
   r  │  3 │  2 │  2 │  2 │
      ├────┼────┼────┼────┤
   s  │  4 │  3 │  3 │  2 │
      ├────┼────┼────┼────┤
   e  │  5 │  4 │  4 │  3 │  ← ANSWER = 3
      └────┴────┴────┴────┘

   The 3 ops: horse → rorse (replace h→r)
                    → rose  (delete r)
                    → ros   (delete e)
```

```cpp
int minDistance(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1));
    for (int i = 0; i <= m; ++i) dp[i][0] = i;     // delete everything
    for (int j = 0; j <= n; ++j) dp[0][j] = j;     // insert everything

    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j)
            dp[i][j] = (a[i-1] == b[j-1])
                     ? dp[i-1][j-1]                              // free match
                     : 1 + min({dp[i-1][j-1],   // replace
                                dp[i-1][j],     // delete
                                dp[i][j-1]});   // insert
    return dp[m][n];
}
```

---

### 30. Distinct Subsequences 🔴
```cpp
int numDistinct(string s, string t) {
    int m = s.size(), n = t.size();
    vector<unsigned long long> dp(n + 1, 0);
    dp[0] = 1;
    for (int i = 1; i <= m; ++i)
        for (int j = n; j >= 1; --j)               // ⭐ backward: dp[j-1] must be old
            if (s[i-1] == t[j-1]) dp[j] += dp[j-1];
    return dp[n];
}
```

---

### 31. Interleaving String 🟡
```cpp
bool isInterleave(string a, string b, string c) {
    int m = a.size(), n = b.size();
    if (m + n != (int)c.size()) return false;
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;
    for (int i = 0; i <= m; ++i)
        for (int j = 0; j <= n; ++j) {
            if (i && dp[i-1][j] && a[i-1] == c[i+j-1]) dp[i][j] = true;
            if (j && dp[i][j-1] && b[j-1] == c[i+j-1]) dp[i][j] = true;
        }
    return dp[m][n];
}
```

---

### 32. Regular Expression Matching 🔴
> Support `.` (any single char) and `*` (zero or more of the preceding element).

```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n; ++j)                   // patterns like a*b*c* match ""
        if (p[j-1] == '*') dp[0][j] = dp[0][j-2];

    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j) {
            if (p[j-1] == '*') {
                dp[i][j] = dp[i][j-2];             // ⭐ use the star ZERO times
                if (p[j-2] == '.' || p[j-2] == s[i-1])
                    dp[i][j] = dp[i][j] || dp[i-1][j];   // ⭐ use it ONE MORE time
            } else if (p[j-1] == '.' || p[j-1] == s[i-1]) {
                dp[i][j] = dp[i-1][j-1];
            }
        }
    return dp[m][n];
}
```
**Key insight:** `*` has exactly two behaviours — consume nothing (skip the pair `x*`, look back 2), or consume one more character (stay on the pattern, advance in the string). Handle both and OR them.

---

### 33. Wildcard Matching 🔴
```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n; ++j) if (p[j-1] == '*') dp[0][j] = dp[0][j-1];
    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j) {
            if (p[j-1] == '*')
                dp[i][j] = dp[i-1][j] || dp[i][j-1];   // ⭐ match one more, or none
            else if (p[j-1] == '?' || p[j-1] == s[i-1])
                dp[i][j] = dp[i-1][j-1];
        }
    return dp[m][n];
}
```

---

### 34. Longest Palindromic Subsequence 🟡
```cpp
int longestPalindromeSubseq(string s) {
    int n = s.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = n - 1; i >= 0; --i) {             // ⭐ i descending, j ascending
        dp[i][i] = 1;
        for (int j = i + 1; j < n; ++j)
            dp[i][j] = (s[i] == s[j])
                     ? dp[i+1][j-1] + 2            // both ends match
                     : max(dp[i+1][j], dp[i][j-1]);
    }
    return dp[0][n-1];
}
```
**Alternative framing:** it's simply `LCS(s, reverse(s))`.

---

### 35. Shortest Common Supersequence 🔴
```cpp
string shortestCommonSupersequence(string a, string b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j)
            dp[i][j] = (a[i-1] == b[j-1]) ? dp[i-1][j-1] + 1
                                          : max(dp[i-1][j], dp[i][j-1]);

    string out;                                    // ⭐ walk BACKWARDS through the table
    int i = m, j = n;
    while (i && j) {
        if (a[i-1] == b[j-1]) { out += a[--i]; --j; }
        else if (dp[i-1][j] > dp[i][j-1]) out += a[--i];
        else out += b[--j];
    }
    while (i) out += a[--i];
    while (j) out += b[--j];
    reverse(out.begin(), out.end());
    return out;
}
```
**Key insight:** Reconstructing the actual answer (not just its length) means walking the DP table backwards from the final cell.

---

## E. Interval DP

#### 💬 The shape
For interval DP the state is a **range** `dp[i][j]`, and you decide a *split point* or a *last operation* inside it. You must fill by increasing range length, because a big range depends on smaller ranges inside it.

```
   for (len = 2; len <= n; ++len)          ⭐ length OUTERMOST
     for (i = 0; i + len - 1 < n; ++i) {
       j = i + len - 1;
       for (k = i; k < j; ++k)             the split point
         dp[i][j] = best(dp[i][k], dp[k+1][j], cost)
     }
```

### 36. Burst Balloons 🔴
> Bursting balloon `i` earns `nums[left] * nums[i] * nums[right]`. Maximize total coins.

#### 💬 The trick that makes this solvable
Your instinct is to ask "which balloon do I burst **first**?" — but that fails, because bursting changes who the neighbours are, so the subproblems aren't independent.

Flip it: ask **"which balloon do I burst LAST in this range?"** If balloon `k` is last, then everything left of `k` and everything right of `k` were already burst independently — and when `k` finally pops, its neighbours are exactly the fixed boundaries of the range. The subproblems become independent.

```
   Range (i, j) exclusive, k is the LAST to burst inside:

   [i] ....... [k] ....... [j]
    ▲    ▲      ▲     ▲     ▲
    │  solved   │  solved   │
    │  first    │  first    │
    └───────────┴───────────┘
        when k finally pops, its neighbours are i and j
        (everything between is already gone)

   dp[i][j] = max over k of:
              dp[i][k] + nums[i]*nums[k]*nums[j] + dp[k][j]
```

```cpp
int maxCoins(vector<int>& nums) {
    int n = nums.size();
    vector<int> a(n + 2, 1);                       // ⭐ pad with 1s at both ends
    for (int i = 0; i < n; ++i) a[i+1] = nums[i];
    n += 2;

    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int len = 2; len < n; ++len)              // range length
        for (int i = 0; i + len < n; ++i) {
            int j = i + len;
            for (int k = i + 1; k < j; ++k)        // k = LAST balloon burst
                dp[i][j] = max(dp[i][j], dp[i][k] + a[i]*a[k]*a[j] + dp[k][j]);
        }
    return dp[0][n-1];
}
```
**Complexity:** O(n³).

---

### 37. Matrix Chain Multiplication 🔴
```cpp
int matrixChain(vector<int>& dims) {               // dims has n+1 entries for n matrices
    int n = dims.size() - 1;
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int len = 2; len <= n; ++len)
        for (int i = 0; i + len - 1 < n; ++i) {
            int j = i + len - 1;
            dp[i][j] = INT_MAX;
            for (int k = i; k < j; ++k)            // split between k and k+1
                dp[i][j] = min(dp[i][j],
                    dp[i][k] + dp[k+1][j] + dims[i]*dims[k+1]*dims[j+1]);
        }
    return dp[0][n-1];
}
```

---

### 38. Palindrome Partitioning II 🔴
```cpp
int minCut(string s) {
    int n = s.size();
    vector<vector<bool>> pal(n, vector<bool>(n, false));
    vector<int> dp(n + 1);
    for (int i = 0; i <= n; ++i) dp[i] = i - 1;    // worst case: cut everywhere

    for (int i = 0; i < n; ++i)
        for (int j = 0; j <= i; ++j)
            if (s[j] == s[i] && (i - j < 2 || pal[j+1][i-1])) {
                pal[j][i] = true;
                dp[i+1] = min(dp[i+1], dp[j] + 1);
            }
    return dp[n];
}
```

---

### 39. Stone Game 🟡
```cpp
bool stoneGame(vector<int>& piles) {
    int n = piles.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = 0; i < n; ++i) dp[i][i] = piles[i];
    for (int len = 2; len <= n; ++len)
        for (int i = 0; i + len - 1 < n; ++i) {
            int j = i + len - 1;
            dp[i][j] = max(piles[i] - dp[i+1][j],  // ⭐ take left, opponent faces (i+1,j)
                           piles[j] - dp[i][j-1]); //    take right
        }
    return dp[0][n-1] > 0;
}
```
**Key insight:** `dp[i][j]` = *my score minus my opponent's* on this range. Subtracting the opponent's optimal result models perfect play by both sides in one line.

---

### 40. Minimum Cost to Cut a Stick 🔴
```cpp
int minCost(int n, vector<int>& cuts) {
    cuts.push_back(0); cuts.push_back(n);
    sort(cuts.begin(), cuts.end());
    int m = cuts.size();
    vector<vector<int>> dp(m, vector<int>(m, 0));
    for (int len = 2; len < m; ++len)
        for (int i = 0; i + len < m; ++i) {
            int j = i + len;
            dp[i][j] = INT_MAX;
            for (int k = i + 1; k < j; ++k)        // k = FIRST cut in this segment
                dp[i][j] = min(dp[i][j], dp[i][k] + dp[k][j] + cuts[j] - cuts[i]);
        }
    return dp[0][m-1];
}
```

---

## F. Bitmask DP

#### 💬 When to reach for it
When `n ≤ 20` and you need to track **which subset** of items you've used, encode the subset as bits of an integer. Bit `i` set means item `i` is used.

```
   n = 4 items,  mask = 1011 (binary) = 11 (decimal)

   bit:    3  2  1  0
   value:  1  0  1  1
           │  │  │  └── item 0 used ✅
           │  │  └───── item 1 used ✅
           │  └──────── item 2 NOT used ❌
           └─────────── item 3 used ✅

   2²⁰ = about 1 million states — comfortably feasible.
   2³⁰ = a billion — no longer feasible. That's why n ≤ 20.
```

### 41. Travelling Salesman (Held-Karp) 🔴
```cpp
int tsp(vector<vector<int>>& dist) {
    int n = dist.size();
    vector<vector<int>> dp(1 << n, vector<int>(n, INT_MAX / 2));
    dp[1][0] = 0;                                  // start at city 0, only it visited

    for (int mask = 1; mask < (1 << n); ++mask)
        for (int u = 0; u < n; ++u) {
            if (!(mask & (1 << u))) continue;      // u must be in the visited set
            for (int v = 0; v < n; ++v) {
                if (mask & (1 << v)) continue;     // v already visited
                int nm = mask | (1 << v);
                dp[nm][v] = min(dp[nm][v], dp[mask][u] + dist[u][v]);
            }
        }

    int best = INT_MAX;
    for (int u = 1; u < n; ++u)
        best = min(best, dp[(1 << n) - 1][u] + dist[u][0]);   // return home
    return best;
}
```
**Complexity:** O(2ⁿ · n²) — vastly better than O(n!) brute force.

---

### 42. Partition to K Equal Sum Subsets 🔴
```cpp
bool canPartitionKSubsets(vector<int>& nums, int k) {
    int total = accumulate(nums.begin(), nums.end(), 0);
    if (total % k) return false;
    int target = total / k, n = nums.size();
    sort(nums.rbegin(), nums.rend());              // ⭐ big items first → prune faster
    if (nums[0] > target) return false;

    vector<int> dp(1 << n, -1);                    // dp[mask] = current bucket fill
    dp[0] = 0;
    for (int mask = 0; mask < (1 << n); ++mask) {
        if (dp[mask] < 0) continue;
        for (int i = 0; i < n; ++i) {
            if (mask & (1 << i)) continue;
            if (dp[mask] + nums[i] > target) continue;
            dp[mask | (1 << i)] = (dp[mask] + nums[i]) % target;
        }
    }
    return dp[(1 << n) - 1] == 0;
}
```

---

### 43. Shortest Path Visiting All Nodes 🔴
```cpp
int shortestPathLength(vector<vector<int>>& graph) {
    int n = graph.size(), full = (1 << n) - 1;
    queue<pair<int,int>> q;                        // {node, visited mask}
    vector<vector<bool>> seen(n, vector<bool>(1 << n, false));
    for (int i = 0; i < n; ++i) {                  // ⭐ start from EVERY node
        q.push({i, 1 << i});
        seen[i][1 << i] = true;
    }

    int steps = 0;
    while (!q.empty()) {
        int sz = q.size();
        for (int s = 0; s < sz; ++s) {
            auto [u, mask] = q.front(); q.pop();
            if (mask == full) return steps;
            for (int v : graph[u]) {
                int nm = mask | (1 << v);
                if (seen[v][nm]) continue;
                seen[v][nm] = true;
                q.push({v, nm});
            }
        }
        ++steps;
    }
    return 0;
}
```
**Key insight:** BFS where the state is `(where I am, what I've visited)`. Revisiting a node is allowed — what must not repeat is the full state.

---

## G. Tree & Digit DP

### 44. House Robber III 🟡
```cpp
pair<int,int> robTree(TreeNode* n) {               // {rob this node, skip this node}
    if (!n) return {0, 0};
    auto [rl, sl] = robTree(n->left);
    auto [rr, sr] = robTree(n->right);
    return { n->val + sl + sr,                     // rob → children MUST be skipped
             max(rl, sl) + max(rr, sr) };          // skip → children free to choose
}
int rob(TreeNode* root) { auto [a, b] = robTree(root); return max(a, b); }
```
**Key insight:** Returning a **pair of states** lets the parent choose. This is the standard shape of tree DP.

---

### 45. Binary Tree Maximum Path Sum 🔴
```cpp
int best = INT_MIN;
int gain(TreeNode* n) {
    if (!n) return 0;
    int l = max(0, gain(n->left));                 // negative subtrees are worth skipping
    int r = max(0, gain(n->right));
    best = max(best, n->val + l + r);              // path THROUGH this node
    return n->val + max(l, r);                     // but only ONE branch goes up
}
```

---

### 46. Count Numbers with Unique Digits 🟡
```cpp
int countNumbersWithUniqueDigits(int n) {
    if (n == 0) return 1;
    int total = 10, unique = 9;                    // 1-digit: 0..9
    for (int i = 2; i <= min(n, 10); ++i) {
        unique *= (11 - i);                        // 9×9, then ×8, ×7, ...
        total += unique;
    }
    return total;
}
```

---

### 47. Numbers At Most N Given Digit Set (digit DP) 🔴
```cpp
int atMostNGivenDigitSet(vector<string>& digits, int n) {
    string s = to_string(n);
    int k = s.size(), d = digits.size(), ans = 0;

    for (int i = 1; i < k; ++i) ans += pow(d, i);  // all shorter lengths are free

    for (int i = 0; i < k; ++i) {                  // same length, digit by digit
        bool prefixMatched = false;
        for (auto& dg : digits) {
            if (dg[0] < s[i]) ans += pow(d, k - i - 1);   // strictly smaller → all free
            else if (dg[0] == s[i]) { prefixMatched = true; break; }
        }
        if (!prefixMatched) return ans;            // ⭐ can't continue matching
    }
    return ans + 1;                                // +1 for n itself
}
```

---

## H. Stock & Game DP

### 48. Best Time to Buy and Sell Stock 🟢
```cpp
int maxProfit(vector<int>& p) {
    int lo = INT_MAX, best = 0;
    for (int x : p) { lo = min(lo, x); best = max(best, x - lo); }
    return best;
}
```

---

### 49. Stock II (unlimited transactions) 🟡
```cpp
int maxProfit(vector<int>& p) {
    int total = 0;
    for (int i = 1; i < (int)p.size(); ++i)
        total += max(0, p[i] - p[i-1]);            // ⭐ grab every upward step
    return total;
}
```
**Key insight:** With unlimited transactions, capturing every rise is optimal — a multi-day climb equals the sum of its daily gains.

---

### 50. Stock III (at most 2 transactions) 🔴
```cpp
int maxProfit(vector<int>& p) {
    int buy1 = INT_MIN, sell1 = 0, buy2 = INT_MIN, sell2 = 0;
    for (int x : p) {
        buy1  = max(buy1,  -x);                    // best after 1st buy
        sell1 = max(sell1, buy1 + x);              // best after 1st sell
        buy2  = max(buy2,  sell1 - x);             // ⭐ 2nd buy uses 1st profit
        sell2 = max(sell2, buy2 + x);
    }
    return sell2;
}
```

---

### 51. Stock IV (at most k transactions) 🔴
```cpp
int maxProfit(int k, vector<int>& p) {
    int n = p.size();
    if (k >= n / 2) {                              // ⭐ effectively unlimited
        int t = 0;
        for (int i = 1; i < n; ++i) t += max(0, p[i] - p[i-1]);
        return t;
    }
    vector<int> buy(k + 1, INT_MIN), sell(k + 1, 0);
    for (int x : p)
        for (int j = 1; j <= k; ++j) {
            buy[j]  = max(buy[j],  sell[j-1] - x);
            sell[j] = max(sell[j], buy[j] + x);
        }
    return sell[k];
}
```

---

### 52. Stock with Transaction Fee 🟡
```cpp
int maxProfit(vector<int>& p, int fee) {
    int held = INT_MIN, free = 0;
    for (int x : p) {
        int prevFree = free;
        free = max(free, held + x - fee);          // sell, pay the fee
        held = max(held, prevFree - x);            // buy
    }
    return free;
}
```

---

### 53. Predict the Winner 🟡
```cpp
bool PredictTheWinner(vector<int>& nums) {
    int n = nums.size();
    vector<int> dp(nums.begin(), nums.end());      // dp[i] over shrinking ranges
    for (int len = 2; len <= n; ++len)
        for (int i = 0; i + len - 1 < n; ++i)
            dp[i] = max(nums[i] - dp[i+1], nums[i+len-1] - dp[i]);
    return dp[0] >= 0;
}
```

---

### 54. Can I Win 🟡
```cpp
bool canIWin(int maxChoosable, int target) {
    if (maxChoosable >= target) return true;
    if (maxChoosable * (maxChoosable + 1) / 2 < target) return false;   // sum too small

    unordered_map<int,bool> memo;                  // memo on the used-bitmask
    function<bool(int,int)> win = [&](int used, int remaining) -> bool {
        if (memo.count(used)) return memo[used];
        for (int i = 0; i < maxChoosable; ++i) {
            if (used & (1 << i)) continue;
            if (i + 1 >= remaining || !win(used | (1 << i), remaining - i - 1))
                return memo[used] = true;          // ⭐ opponent loses → I win
        }
        return memo[used] = false;
    };
    return win(0, target);
}
```

---

## I. Miscellaneous Classics

### 55. Longest Increasing Path in a Matrix 🔴
```cpp
int longestIncreasingPath(vector<vector<int>>& m) {
    int R = m.size(), C = m[0].size(), best = 0;
    vector<vector<int>> memo(R, vector<int>(C, 0));
    const int dr[] = {-1,1,0,0}, dc[] = {0,0,-1,1};

    function<int(int,int)> dfs = [&](int r, int c) -> int {
        if (memo[r][c]) return memo[r][c];
        int len = 1;
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= R || nc < 0 || nc >= C || m[nr][nc] <= m[r][c]) continue;
            len = max(len, 1 + dfs(nr, nc));
        }
        return memo[r][c] = len;
    };
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c) best = max(best, dfs(r, c));
    return best;
}
```
**Key insight:** Because paths must strictly increase, the graph is acyclic — so no visited-set is needed and plain memoization works.

---

### 56. Russian Doll Envelopes 🔴
```cpp
int maxEnvelopes(vector<vector<int>>& e) {
    sort(e.begin(), e.end(), [](auto& a, auto& b) {
        return a[0] == b[0] ? a[1] > b[1]          // ⭐ width tie → height DESCENDING
                            : a[0] < b[0];
    });
    vector<int> tails;                             // LIS on heights
    for (auto& x : e) {
        auto it = lower_bound(tails.begin(), tails.end(), x[1]);
        if (it == tails.end()) tails.push_back(x[1]);
        else *it = x[1];
    }
    return tails.size();
}
```
**Key insight:** Sorting heights *descending* within equal widths prevents two same-width envelopes from both being picked — the LIS is strictly increasing, so it can't take a descending pair.

---

### 57. Maximum Profit in Job Scheduling 🔴
```cpp
int jobScheduling(vector<int>& start, vector<int>& end, vector<int>& profit) {
    int n = start.size();
    vector<array<int,3>> jobs(n);
    for (int i = 0; i < n; ++i) jobs[i] = {end[i], start[i], profit[i]};
    sort(jobs.begin(), jobs.end());                // by END time

    map<int,int> dp{{0, 0}};                       // endTime -> best profit up to it
    for (auto& [e, s, p] : jobs) {
        auto it = prev(dp.upper_bound(s));         // ⭐ best profit finishing by s
        int cand = it->second + p;
        if (cand > dp.rbegin()->second) dp[e] = cand;   // only keep improvements
    }
    return dp.rbegin()->second;
}
```

---

### 58. Frog Jump 🔴
```cpp
bool canCross(vector<int>& stones) {
    unordered_map<int, unordered_set<int>> jumps;  // stone -> set of arriving jump sizes
    for (int s : stones) jumps[s] = {};
    jumps[0].insert(0);
    for (int s : stones)
        for (int k : jumps[s])
            for (int step : {k-1, k, k+1})
                if (step > 0 && jumps.count(s + step)) jumps[s + step].insert(step);
    return !jumps[stones.back()].empty();
}
```
**Key insight:** The state is `(stone, last jump size)` — position alone isn't enough, because your options depend on how you arrived.

---

### 59. Count Different Palindromic Subsequences 🔴
```cpp
int countPalindromicSubsequences(string s) {
    const int MOD = 1e9 + 7;
    int n = s.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = 0; i < n; ++i) dp[i][i] = 1;

    for (int len = 2; len <= n; ++len)
        for (int i = 0; i + len - 1 < n; ++i) {
            int j = i + len - 1;
            if (s[i] == s[j]) {
                int lo = i + 1, hi = j - 1;
                while (lo <= hi && s[lo] != s[i]) ++lo;
                while (lo <= hi && s[hi] != s[i]) --hi;

                if (lo > hi)       dp[i][j] = 2LL * dp[i+1][j-1] % MOD + 2;
                else if (lo == hi) dp[i][j] = 2LL * dp[i+1][j-1] % MOD + 1;
                else               dp[i][j] = (2LL * dp[i+1][j-1] - dp[lo+1][hi-1]) % MOD;
            } else {
                dp[i][j] = (dp[i+1][j] + dp[i][j-1] - dp[i+1][j-1]) % MOD;  // inclusion-exclusion
            }
            dp[i][j] = (dp[i][j] + MOD) % MOD;
        }
    return dp[0][n-1];
}
```

---

### 60. Cherry Pickup 🔴
```cpp
int cherryPickup(vector<vector<int>>& g) {
    int n = g.size();
    vector<vector<int>> dp(n, vector<int>(n, INT_MIN));
    dp[0][0] = g[0][0];

    for (int step = 1; step <= 2 * n - 2; ++step) {
        vector<vector<int>> nd(n, vector<int>(n, INT_MIN));
        for (int r1 = max(0, step - n + 1); r1 <= min(n - 1, step); ++r1)
            for (int r2 = max(0, step - n + 1); r2 <= min(n - 1, step); ++r2) {
                int c1 = step - r1, c2 = step - r2;
                if (g[r1][c1] == -1 || g[r2][c2] == -1) continue;

                int best = INT_MIN;
                for (int a = 0; a < 2; ++a)
                    for (int b = 0; b < 2; ++b) {
                        int p1 = r1 - a, p2 = r2 - b;
                        if (p1 >= 0 && p2 >= 0) best = max(best, dp[p1][p2]);
                    }
                if (best == INT_MIN) continue;
                nd[r1][r2] = best + g[r1][c1] + (r1 == r2 ? 0 : g[r2][c2]);
                //                                ⭐ same cell → count only once
            }
        dp = move(nd);
    }
    return max(0, dp[n-1][n-1]);
}
```
**Key insight:** Two round trips is equivalent to **two people walking simultaneously**. Both take the same number of steps, so `c = step - r` — the state collapses from four dimensions to three.

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║              DYNAMIC PROGRAMMING — PATTERN RECALL                 ║
╠═══════════════════════════════════════════════════════════════════╣
║ THE PROCESS (say it out loud in interviews)                       ║
║   brute-force recursion → find the state → memoize →              ║
║   (optional) tabulate → (optional) shrink space                   ║
╠═══════════════════════════════════════════════════════════════════╣
║ HOW TO SPOT THE STATE                                             ║
║   one index changing        → dp[i]                               ║
║   two sequences             → dp[i][j]                            ║
║   index + a budget/capacity → dp[i][w]                            ║
║   a range                   → dp[i][j], fill by LENGTH            ║
║   subset of ≤20 items       → dp[mask]                            ║
║   tree node                 → return a PAIR of states             ║
║   position + resource left  → dp[pos][resource]                   ║
╠═══════════════════════════════════════════════════════════════════╣
║ ⭐ KNAPSACK LOOP DIRECTION — the #1 DP bug                         ║
║   0/1 (use once)     → weight loop BACKWARD                       ║
║   unbounded (reuse)  → weight loop FORWARD                        ║
║                                                                   ║
║ ⭐ COIN CHANGE LOOP ORDER                                          ║
║   coins outer  → COMBINATIONS ({1,2} == {2,1})                    ║
║   amount outer → PERMUTATIONS ({1,2} != {2,1})                    ║
╠═══════════════════════════════════════════════════════════════════╣
║ TWO-SEQUENCE TEMPLATE                                             ║
║   match    → dp[i-1][j-1] (+1 or +0)                              ║
║   no match → best of dp[i-1][j] and dp[i][j-1]                    ║
║                                                                   ║
║ INTERVAL DP → loop by LENGTH first, then start, then split point  ║
║   "burst balloons" trick: ask which is LAST, not first            ║
║                                                                   ║
║ GAME DP → dp = my score MINUS opponent's; subtract for their turn ║
║ STOCK DP → model as states (free / held / sold) and transitions   ║
╠═══════════════════════════════════════════════════════════════════╣
║ WHEN DP IS **NOT** THE ANSWER                                     ║
║   no overlapping subproblems      → plain divide & conquer        ║
║   a provable greedy choice exists → greedy (jump game, stock II)  ║
║   n is tiny (≤ 12)                → brute force is fine and safer ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Greedy, Backtracking & Misc →](10-greedy-backtracking-misc.md)
