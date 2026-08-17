# 🎯 Greedy, Backtracking, Bits & Math — 45 Problems

> Three very different mindsets. **Greedy** commits immediately and never looks back. **Backtracking** tries everything but abandons dead ends early. **Bit tricks** exploit binary structure. Knowing which one applies is most of the battle.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## Part 1 — Greedy

### 🧠 The greedy mindset

#### 💬 What greedy actually means
A greedy algorithm makes the choice that looks best **right now** and never reconsiders. No backtracking, no table of subproblems. Just: pick the locally best option, move on.

The catch is that this is often **wrong**. Making a locally optimal choice can lock you out of the globally optimal solution.

```
   Coin change, coins = [1, 3, 4], target = 6

   GREEDY: take the biggest coin that fits
     4 → remaining 2
     1 → remaining 1
     1 → remaining 0
     TOTAL: 3 coins ❌

   OPTIMAL:
     3 + 3
     TOTAL: 2 coins ✅

   → Greedy FAILS here. This problem needs DP.
```

So the real skill is not "write greedy code" — it's **proving greedy is safe** before you commit.

#### The two proof techniques

```
   ┌─────────────────────────────────────────────────────────────┐
   │ 1. EXCHANGE ARGUMENT                                        │
   │    Take ANY optimal solution. Show you can swap in the      │
   │    greedy choice without making it worse.                   │
   │                                                             │
   │    Example (activity selection):                            │
   │    "Suppose the optimal schedule doesn't start with the     │
   │     earliest-finishing meeting. Swap that meeting in.       │
   │     It finishes no later, so nothing else breaks, and       │
   │     the count is unchanged. So greedy is at least as good." │
   ├─────────────────────────────────────────────────────────────┤
   │ 2. GREEDY STAYS AHEAD                                       │
   │    Show that after every step, greedy's partial solution    │
   │    is at least as good as any other partial solution.       │
   └─────────────────────────────────────────────────────────────┘
```

🎤 **In an interview, say:** *"Let me check whether greedy is safe here — can I construct a counterexample?"* Then actually try. If you can't find one in a minute, sketch the exchange argument. This is exactly what separates a strong answer from a lucky one.

---

### 1. Jump Game 🟡
> Each `nums[i]` is your max jump length from index `i`. Can you reach the last index?

#### 💬 Think of it like this
You don't need to know *which* jumps to take. You only need to know: **how far can I possibly get?**

Walk left to right tracking a single number — the furthest index reachable so far. At each position, first check whether you can even stand here (is `i` within reach?). If yes, update your reach with whatever this position offers.

```
   nums = [2, 3, 1, 1, 4]

   i=0  reach = max(0, 0+2) = 2      ┌──────┐
                                     [2, 3, 1, 1, 4]

   i=1  1 ≤ 2 ✅  reach = max(2, 1+3) = 4
                                      ┌────────────┐
                                     [2, 3, 1, 1, 4]

   i=2  2 ≤ 4 ✅  reach = max(4, 2+1) = 4
   i=3  3 ≤ 4 ✅  reach = max(4, 3+1) = 4
   i=4  4 ≤ 4 ✅  reached the end → TRUE

   Failure case: nums = [3, 2, 1, 0, 4]
   i=0..3 → reach = 3
   i=4  →  4 > 3  ❌  stuck at index 3, return FALSE
```

```cpp
bool canJump(vector<int>& nums) {
    int reach = 0;
    for (int i = 0; i < (int)nums.size(); ++i) {
        if (i > reach) return false;               // ⭐ can't even stand here
        reach = max(reach, i + nums[i]);
    }
    return true;
}
```
**Why greedy is safe:** reachability is monotonic — if you can reach index `i`, you can reach everything before it. There is no benefit to "saving" jumps.

---

### 2. Jump Game II 🟡
```cpp
int jump(vector<int>& nums) {
    int jumps = 0, curEnd = 0, farthest = 0;
    for (int i = 0; i < (int)nums.size() - 1; ++i) {
        farthest = max(farthest, i + nums[i]);
        if (i == curEnd) { ++jumps; curEnd = farthest; }   // ⭐ range exhausted
    }
    return jumps;
}
```
**Key insight:** This is BFS by levels, implemented with two pointers. `curEnd` marks the boundary of the current jump's reach; hitting it forces another jump.

---

### 3. Gas Station 🟡
> Circular route of gas stations. `gas[i]` fuel available, `cost[i]` to reach the next. Find the starting index, or `-1`.

#### 💬 Two insights that solve it instantly

**Insight 1:** If total gas < total cost, no solution exists. Period.

**Insight 2:** If you start at `A` and run dry at station `B`, then **no station between A and B can work either**. Why? Because you arrived at each of those with non-negative fuel — starting there means arriving with *zero*, which is worse. So skip past `B` entirely.

```
   gas  = [1, 2, 3, 4, 5]
   cost = [3, 4, 5, 1, 2]
   diff = [-2, -2, -2, 3, 3]

   start=0  tank: -2  ❌ fail at 0 → restart at 1
   start=1  tank: -2  ❌ fail at 1 → restart at 2
   start=2  tank: -2  ❌ fail at 2 → restart at 3
   start=3  tank: 3, then 3+3=6   ✅ survives to the end

   total = -2-2-2+3+3 = 0  ≥ 0  →  a solution exists
   ANSWER = 3
```

```cpp
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int total = 0, tank = 0, start = 0;
    for (int i = 0; i < (int)gas.size(); ++i) {
        int diff = gas[i] - cost[i];
        total += diff;
        tank += diff;
        if (tank < 0) { start = i + 1; tank = 0; }  // ⭐ restart AFTER the failure
    }
    return total >= 0 ? start : -1;
}
```
**Complexity:** O(n) single pass — no need to try each start.

---

### 4. Task Scheduler 🟡
```cpp
int leastInterval(vector<char>& tasks, int n) {
    int cnt[26] = {};
    for (char c : tasks) cnt[c - 'A']++;
    int maxCount = *max_element(cnt, cnt + 26);
    int numMax = count(cnt, cnt + 26, maxCount);
    return max((int)tasks.size(), (maxCount - 1) * (n + 1) + numMax);
}
```

```
   The most frequent task sets the skeleton. With A×3 and n=2:

   A _ _ | A _ _ | A
   └──3──┘└──3──┘  └─ the last block needs no idle time

   (maxCount-1) blocks of size (n+1), plus numMax tasks at the end.
   If other tasks can fill every gap, the answer is just tasks.size().
```

---

### 5. Partition Labels 🟡
```cpp
vector<int> partitionLabels(string s) {
    int last[26] = {};
    for (int i = 0; i < (int)s.size(); ++i) last[s[i] - 'a'] = i;

    vector<int> out;
    int start = 0, end = 0;
    for (int i = 0; i < (int)s.size(); ++i) {
        end = max(end, last[s[i] - 'a']);          // ⭐ must extend to cover this char
        if (i == end) { out.push_back(end - start + 1); start = i + 1; }
    }
    return out;
}
```

```
   s = "ababcbacadefegde..."

   Scan and keep stretching the window until every character
   inside it has ALL its occurrences inside it too:

   a b a b c b a c a | d e f e g d e | ...
   └──── i == end ───┘ └──── cut ────┘
        the 'a' at index 8 is the last 'a', so we can close here
```

---

### 6. Non-overlapping Intervals 🟡
```cpp
int eraseOverlapIntervals(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end(), [](auto& a, auto& b){ return a[1] < b[1]; });  // ⭐ by END
    int end = INT_MIN, keep = 0;
    for (auto& x : iv) if (x[0] >= end) { end = x[1]; ++keep; }
    return iv.size() - keep;
}
```

#### 💬 Why sort by END, not start?
Sorting by end time is what makes greedy provably optimal. Keeping the interval that **finishes earliest** leaves the maximum possible room for everything after it.

```
   Sorted by START:              Sorted by END:
   ├─────────────────┤           ├──┤
      ├──┤                          ├──┤
        ├──┤                           ├──┤

   Greedy takes the long one     Greedy takes all three ✅
   → only 1 interval ❌
```

---

### 7. Minimum Number of Arrows 🟡
```cpp
int findMinArrowShots(vector<vector<int>>& pts) {
    sort(pts.begin(), pts.end(), [](auto& a, auto& b){ return a[1] < b[1]; });
    int arrows = 0;
    long long end = LLONG_MIN;
    for (auto& p : pts) if (p[0] > end) { ++arrows; end = p[1]; }
    return arrows;
}
```

---

### 8. Two City Scheduling 🟡
```cpp
int twoCitySchedCost(vector<vector<int>>& costs) {
    sort(costs.begin(), costs.end(), [](auto& a, auto& b) {
        return a[0] - a[1] < b[0] - b[1];          // ⭐ sort by SAVINGS from choosing A
    });
    int total = 0, n = costs.size() / 2;
    for (int i = 0; i < (int)costs.size(); ++i)
        total += (i < n) ? costs[i][0] : costs[i][1];
    return total;
}
```
**Key insight:** Send everyone to city B, then compute how much each person would *save* by switching to A. Pick the `n` biggest savings. Sorting by `costA - costB` does exactly that.

---

### 9. Boats to Save People 🟡
```cpp
int numRescueBoats(vector<int>& p, int limit) {
    sort(p.begin(), p.end());
    int l = 0, r = p.size() - 1, boats = 0;
    while (l <= r) {
        if (p[l] + p[r] <= limit) ++l;             // lightest can join the heaviest
        --r;                                        // the heaviest always goes
        ++boats;
    }
    return boats;
}
```
**Why it's safe:** The heaviest person needs a boat no matter what. The only question is whether anyone can share — and if anyone can, the lightest person can.

---

### 10. Candy 🔴
```cpp
int candy(vector<int>& ratings) {
    int n = ratings.size();
    vector<int> c(n, 1);
    for (int i = 1; i < n; ++i)                    // ⭐ pass 1: left-to-right
        if (ratings[i] > ratings[i-1]) c[i] = c[i-1] + 1;
    for (int i = n - 2; i >= 0; --i)               // ⭐ pass 2: right-to-left
        if (ratings[i] > ratings[i+1]) c[i] = max(c[i], c[i+1] + 1);
    return accumulate(c.begin(), c.end(), 0);
}
```

```
   ratings = [1, 0, 2]

   start:     [1, 1, 1]
   L→R pass:  [1, 1, 2]    only fixes "higher than my LEFT neighbour"
   R→L pass:  [2, 1, 2]    now also fixes "higher than my RIGHT neighbour"
                ▲
              max() keeps both constraints satisfied

   ANSWER = 5
```
**Key insight:** Two constraints pull in opposite directions, so satisfy each in its own sweep and combine with `max`. This two-pass pattern is broadly reusable.

---

### 11. Queue Reconstruction by Height 🟡
```cpp
vector<vector<int>> reconstructQueue(vector<vector<int>>& people) {
    sort(people.begin(), people.end(), [](auto& a, auto& b) {
        return a[0] == b[0] ? a[1] < b[1] : a[0] > b[0];   // tall first, then by k
    });
    vector<vector<int>> out;
    for (auto& p : people) out.insert(out.begin() + p[1], p);   // ⭐ insert at index k
    return out;
}
```
**Key insight:** Place tall people first. Shorter people inserted later are invisible to them, so every already-placed person's `k` value stays correct.

---

### 12. Merge Triplets to Form Target 🟡
```cpp
bool mergeTriplets(vector<vector<int>>& triplets, vector<int>& target) {
    bool got[3] = {false, false, false};
    for (auto& t : triplets) {
        if (t[0] > target[0] || t[1] > target[1] || t[2] > target[2]) continue;  // ⭐ toxic
        for (int i = 0; i < 3; ++i) if (t[i] == target[i]) got[i] = true;
    }
    return got[0] && got[1] && got[2];
}
```
**Key insight:** Since merging takes the max, any triplet exceeding the target in *any* position poisons the result forever. Filter those out, then just check each position is achievable.

---

### 13. Valid Parenthesis String 🟡
```cpp
bool checkValidString(string s) {
    int lo = 0, hi = 0;                            // ⭐ RANGE of possible open counts
    for (char c : s) {
        if (c == '(')      { ++lo; ++hi; }
        else if (c == ')') { --lo; --hi; }
        else               { --lo; ++hi; }         // '*' could be ')', '(' or ''
        if (hi < 0) return false;                  // too many ')' even at best
        lo = max(lo, 0);                           // can't go below zero
    }
    return lo == 0;
}
```
**Key insight:** Instead of trying all interpretations of `*`, track the **range** of possible open-paren counts. Elegant and O(n).

---

### 14. Minimum Deletions to Make Character Frequencies Unique 🟡
```cpp
int minDeletions(string s) {
    int cnt[26] = {};
    for (char c : s) cnt[c - 'a']++;
    unordered_set<int> used;
    int deletions = 0;
    for (int f : cnt) {
        while (f > 0 && used.count(f)) { --f; ++deletions; }   // step down until free
        if (f > 0) used.insert(f);
    }
    return deletions;
}
```

---

### 15. Hand of Straights 🟡
```cpp
bool isNStraightHand(vector<int>& hand, int groupSize) {
    if (hand.size() % groupSize) return false;
    map<int,int> cnt;                              // ordered!
    for (int x : hand) cnt[x]++;
    while (!cnt.empty()) {
        int start = cnt.begin()->first;            // ⭐ smallest card MUST start a group
        for (int i = 0; i < groupSize; ++i) {
            auto it = cnt.find(start + i);
            if (it == cnt.end()) return false;
            if (--it->second == 0) cnt.erase(it);
        }
    }
    return true;
}
```

---

## Part 2 — Backtracking

### 🧠 The backtracking mindset

#### 💬 What backtracking is
Backtracking is **systematic trial and error with an undo button**. You explore a decision tree depth-first: make a choice, recurse deeper, and when you return, **undo** that choice so you can try the next one.

The "undo" is what makes it backtracking rather than plain recursion. It lets you reuse the same data structure for every branch instead of copying state everywhere.

```
   THE TEMPLATE — memorize this shape

   void backtrack(State& st, Path& path) {
       if (isComplete(st)) { record(path); return; }

       for (choice : availableChoices(st)) {
           if (!isValid(choice, st)) continue;    // ⭐ PRUNE — the key to speed

           make(choice, st);  path.push(choice);  //    CHOOSE
           backtrack(st, path);                   //    EXPLORE
           path.pop();  undo(choice, st);         //    UN-CHOOSE  ⭐
       }
   }
```

#### The decision tree, visualized

```
   Generating subsets of [1, 2, 3] — at each element, include or exclude:

                          []
                    ┌──────┴──────┐
                exclude 1      include 1
                    │              │
                   []            [1]
              ┌─────┴─────┐  ┌─────┴─────┐
             []          [2] [1]        [1,2]
            ┌─┴─┐      ┌─┴─┐ ┌─┴─┐     ┌──┴──┐
           [] [3]   [2] [2,3] [1] [1,3] [1,2] [1,2,3]

   8 leaves = 2³ subsets ✅
```

⚠️ **Pruning is everything.** Without it, backtracking is just brute force with extra steps. The `if (!isValid) continue` line is often the difference between 0.1 seconds and never finishing.

---

### 16. Subsets 🟡
```cpp
vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> out;
    vector<int> cur;
    function<void(int)> bt = [&](int i) {
        if (i == (int)nums.size()) { out.push_back(cur); return; }
        bt(i + 1);                                 // exclude nums[i]
        cur.push_back(nums[i]);
        bt(i + 1);                                 // include nums[i]
        cur.pop_back();                            // ⭐ undo
    };
    bt(0);
    return out;
}
```
**Complexity:** O(2ⁿ · n).

---

### 17. Subsets II (with duplicates) 🟡
```cpp
vector<vector<int>> subsetsWithDup(vector<int>& nums) {
    sort(nums.begin(), nums.end());                // ⭐ duplicates must be adjacent
    vector<vector<int>> out;
    vector<int> cur;
    function<void(int)> bt = [&](int start) {
        out.push_back(cur);
        for (int i = start; i < (int)nums.size(); ++i) {
            if (i > start && nums[i] == nums[i-1]) continue;   // ⭐ skip dup at same level
            cur.push_back(nums[i]);
            bt(i + 1);
            cur.pop_back();
        }
    };
    bt(0);
    return out;
}
```

```
   Why "i > start" and not "i > 0"?

   nums = [1, 2, 2]

   At the SAME level, picking the 2nd '2' after skipping the 1st
   would duplicate the branch → skip it.

   But going DEEPER (start advanced past the first 2), picking
   the second '2' is legitimate — that's the subset [2,2].

   level 0:  start=0, i=0 → [1]
                     i=1 → [2]
                     i=2 → SKIP (i>start and nums[2]==nums[1])
   level 1:  start=2, i=2 → [2,2] ✅ allowed
```

---

### 18. Permutations 🟡
```cpp
vector<vector<int>> permute(vector<int>& nums) {
    vector<vector<int>> out;
    function<void(int)> bt = [&](int start) {
        if (start == (int)nums.size()) { out.push_back(nums); return; }
        for (int i = start; i < (int)nums.size(); ++i) {
            swap(nums[start], nums[i]);            // ⭐ swap-based: no extra space
            bt(start + 1);
            swap(nums[start], nums[i]);            // undo
        }
    };
    bt(0);
    return out;
}
```

---

### 19. Permutations II (with duplicates) 🟡
```cpp
vector<vector<int>> permuteUnique(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    int n = nums.size();
    vector<vector<int>> out;
    vector<int> cur;
    vector<bool> used(n, false);
    function<void()> bt = [&]() {
        if ((int)cur.size() == n) { out.push_back(cur); return; }
        for (int i = 0; i < n; ++i) {
            if (used[i]) continue;
            // ⭐ only use a duplicate if its identical predecessor is already used
            if (i > 0 && nums[i] == nums[i-1] && !used[i-1]) continue;
            used[i] = true;  cur.push_back(nums[i]);
            bt();
            cur.pop_back();  used[i] = false;
        }
    };
    bt();
    return out;
}
```
**Key insight:** The `!used[i-1]` condition enforces that identical values are always consumed left-to-right, so each distinct arrangement is generated exactly once.

---

### 20. Combinations 🟡
```cpp
vector<vector<int>> combine(int n, int k) {
    vector<vector<int>> out;
    vector<int> cur;
    function<void(int)> bt = [&](int start) {
        if ((int)cur.size() == k) { out.push_back(cur); return; }
        // ⭐ PRUNE: stop if not enough numbers remain to reach size k
        for (int i = start; i <= n - (k - cur.size()) + 1; ++i) {
            cur.push_back(i);
            bt(i + 1);
            cur.pop_back();
        }
    };
    bt(1);
    return out;
}
```

---

### 21. Combination Sum 🟡
```cpp
vector<vector<int>> combinationSum(vector<int>& c, int target) {
    sort(c.begin(), c.end());
    vector<vector<int>> out;
    vector<int> cur;
    function<void(int,int)> bt = [&](int start, int rem) {
        if (rem == 0) { out.push_back(cur); return; }
        for (int i = start; i < (int)c.size(); ++i) {
            if (c[i] > rem) break;                 // ⭐ sorted → all later are too big
            cur.push_back(c[i]);
            bt(i, rem - c[i]);                     // ⭐ `i` not `i+1` → reuse allowed
            cur.pop_back();
        }
    };
    bt(0, target);
    return out;
}
```

---

### 22. Combination Sum II (each used once) 🟡
```cpp
vector<vector<int>> combinationSum2(vector<int>& c, int target) {
    sort(c.begin(), c.end());
    vector<vector<int>> out;
    vector<int> cur;
    function<void(int,int)> bt = [&](int start, int rem) {
        if (rem == 0) { out.push_back(cur); return; }
        for (int i = start; i < (int)c.size(); ++i) {
            if (c[i] > rem) break;
            if (i > start && c[i] == c[i-1]) continue;   // skip duplicates
            cur.push_back(c[i]);
            bt(i + 1, rem - c[i]);                       // ⭐ i+1 → each used once
            cur.pop_back();
        }
    };
    bt(0, target);
    return out;
}
```

---

### 23. Combination Sum III 🟡
```cpp
vector<vector<int>> combinationSum3(int k, int n) {
    vector<vector<int>> out;
    vector<int> cur;
    function<void(int,int)> bt = [&](int start, int rem) {
        if ((int)cur.size() == k) { if (rem == 0) out.push_back(cur); return; }
        for (int i = start; i <= 9; ++i) {
            if (i > rem) break;
            cur.push_back(i);
            bt(i + 1, rem - i);
            cur.pop_back();
        }
    };
    bt(1, n);
    return out;
}
```

---

### 24. Letter Combinations of a Phone Number 🟡
```cpp
vector<string> letterCombinations(string digits) {
    if (digits.empty()) return {};
    const string map[] = {"", "", "abc", "def", "ghi", "jkl",
                          "mno", "pqrs", "tuv", "wxyz"};
    vector<string> out;
    string cur;
    function<void(int)> bt = [&](int i) {
        if (i == (int)digits.size()) { out.push_back(cur); return; }
        for (char c : map[digits[i] - '0']) {
            cur.push_back(c);
            bt(i + 1);
            cur.pop_back();
        }
    };
    bt(0);
    return out;
}
```

---

### 25. Generate Parentheses 🟡
```cpp
vector<string> generateParenthesis(int n) {
    vector<string> out;
    string cur;
    function<void(int,int)> bt = [&](int open, int close) {
        if ((int)cur.size() == 2 * n) { out.push_back(cur); return; }
        if (open < n)     { cur += '('; bt(open + 1, close); cur.pop_back(); }
        if (close < open) { cur += ')'; bt(open, close + 1); cur.pop_back(); }
        //  ⭐ close < open is the whole validity rule
    };
    bt(0, 0);
    return out;
}
```

```
   n = 2, the pruned decision tree:

                    ""
                    │  open<2 ✅
                   "("
              ┌─────┴─────┐
        open<2 ✅      close<open ✅
           "(("           "()"
             │        ┌────┴────┐
      close<open ✅  open<2 ✅  close<open ❌ (0<0 false)
           "(()"        "()("
             │            │
           "(())" ✅    "()()"  ✅

   The two conditions prune every invalid branch BEFORE building it.
```

---

### 26. Word Search 🟡
```cpp
bool exist(vector<vector<char>>& b, string word) {
    int R = b.size(), C = b[0].size();
    function<bool(int,int,int)> dfs = [&](int r, int c, int i) -> bool {
        if (i == (int)word.size()) return true;
        if (r < 0 || r >= R || c < 0 || c >= C || b[r][c] != word[i]) return false;
        char tmp = b[r][c];
        b[r][c] = '#';                             // ⭐ mark
        bool found = dfs(r-1,c,i+1) || dfs(r+1,c,i+1)
                  || dfs(r,c-1,i+1) || dfs(r,c+1,i+1);
        b[r][c] = tmp;                             // ⭐ UNMARK — this is backtracking
        return found;
    };
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c) if (dfs(r, c, 0)) return true;
    return false;
}
```
⚠️ **Compare to Number of Islands**, where you mark and *never* unmark. The difference: island counting wants each cell visited once globally; word search needs the cell available to *other paths*.

---

### 27. Palindrome Partitioning 🟡
```cpp
vector<vector<string>> partition(string s) {
    int n = s.size();
    vector<vector<bool>> pal(n, vector<bool>(n, false));
    for (int i = n - 1; i >= 0; --i)               // precompute palindromes
        for (int j = i; j < n; ++j)
            pal[i][j] = (s[i] == s[j]) && (j - i < 2 || pal[i+1][j-1]);

    vector<vector<string>> out;
    vector<string> cur;
    function<void(int)> bt = [&](int start) {
        if (start == n) { out.push_back(cur); return; }
        for (int end = start; end < n; ++end) {
            if (!pal[start][end]) continue;        // ⭐ prune non-palindromes
            cur.push_back(s.substr(start, end - start + 1));
            bt(end + 1);
            cur.pop_back();
        }
    };
    bt(0);
    return out;
}
```

---

### 28. N-Queens 🔴
```cpp
vector<vector<string>> solveNQueens(int n) {
    vector<vector<string>> out;
    vector<int> pos(n);                            // pos[row] = column
    vector<bool> col(n, false), d1(2*n, false), d2(2*n, false);

    function<void(int)> bt = [&](int r) {
        if (r == n) {
            vector<string> board(n, string(n, '.'));
            for (int i = 0; i < n; ++i) board[i][pos[i]] = 'Q';
            out.push_back(board);
            return;
        }
        for (int c = 0; c < n; ++c) {
            // ⭐ O(1) conflict check via three boolean arrays
            if (col[c] || d1[r + c] || d2[r - c + n]) continue;
            col[c] = d1[r + c] = d2[r - c + n] = true;
            pos[r] = c;
            bt(r + 1);
            col[c] = d1[r + c] = d2[r - c + n] = false;   // undo
        }
    };
    bt(0);
    return out;
}
```

```
   Why r+c and r-c identify diagonals:

        c=0  c=1  c=2  c=3          r+c  (anti-diagonal ↗)
   r=0 [ 0 ][ 1 ][ 2 ][ 3 ]         every cell on the same ↗
   r=1 [ 1 ][ 2 ][ 3 ][ 4 ]         diagonal has the SAME r+c
   r=2 [ 2 ][ 3 ][ 4 ][ 5 ]
   r=3 [ 3 ][ 4 ][ 5 ][ 6 ]

        c=0  c=1  c=2  c=3          r-c  (main diagonal ↘)
   r=0 [ 0 ][-1 ][-2 ][-3 ]         same ↘ diagonal → same r-c
   r=1 [ 1 ][ 0 ][-1 ][-2 ]         (+n to keep the index non-negative)
   r=2 [ 2 ][ 1 ][ 0 ][-1 ]
   r=3 [ 3 ][ 2 ][ 1 ][ 0 ]
```

---

### 29. Sudoku Solver 🔴
```cpp
bool solveSudoku(vector<vector<char>>& b) {
    for (int r = 0; r < 9; ++r)
        for (int c = 0; c < 9; ++c) {
            if (b[r][c] != '.') continue;
            for (char d = '1'; d <= '9'; ++d) {
                if (!isValid(b, r, c, d)) continue;
                b[r][c] = d;
                if (solveSudoku(b)) return true;   // ⭐ solved downstream → done
                b[r][c] = '.';                     // undo and try the next digit
            }
            return false;                          // ⭐ no digit works → backtrack
        }
    return true;                                   // no empty cells left
}
bool isValid(vector<vector<char>>& b, int r, int c, char d) {
    for (int i = 0; i < 9; ++i) {
        if (b[r][i] == d || b[i][c] == d) return false;
        if (b[3*(r/3) + i/3][3*(c/3) + i%3] == d) return false;   // the 3×3 box
    }
    return true;
}
```

---

### 30. Restore IP Addresses 🟡
```cpp
vector<string> restoreIpAddresses(string s) {
    vector<string> out;
    vector<string> parts;
    function<void(int)> bt = [&](int start) {
        if (parts.size() == 4) {
            if (start == (int)s.size())
                out.push_back(parts[0]+"."+parts[1]+"."+parts[2]+"."+parts[3]);
            return;
        }
        for (int len = 1; len <= 3 && start + len <= (int)s.size(); ++len) {
            string seg = s.substr(start, len);
            if (seg.size() > 1 && seg[0] == '0') break;   // ⭐ no leading zeros
            if (stoi(seg) > 255) break;                    // ⭐ max octet
            parts.push_back(seg);
            bt(start + len);
            parts.pop_back();
        }
    };
    bt(0);
    return out;
}
```

---

### 31. Word Break II 🔴
```cpp
vector<string> wordBreak(string s, vector<string>& dict) {
    unordered_set<string> d(dict.begin(), dict.end());
    unordered_map<int, vector<string>> memo;       // ⭐ memoize by start index

    function<vector<string>(int)> bt = [&](int start) -> vector<string> {
        if (memo.count(start)) return memo[start];
        vector<string> res;
        if (start == (int)s.size()) { res.push_back(""); return res; }

        for (int end = start + 1; end <= (int)s.size(); ++end) {
            string word = s.substr(start, end - start);
            if (!d.count(word)) continue;
            for (auto& rest : bt(end))
                res.push_back(word + (rest.empty() ? "" : " " + rest));
        }
        return memo[start] = res;
    };
    return bt(0);
}
```
**Key insight:** Pure backtracking is exponential on inputs like `"aaaaaaa..."` with dict `{"a","aa","aaa"}`. Memoizing the *list of results* per start index makes it tractable.

---

### 32. Beautiful Arrangement 🟡
```cpp
int countArrangement(int n) {
    vector<bool> used(n + 1, false);
    function<int(int)> bt = [&](int pos) -> int {
        if (pos > n) return 1;
        int count = 0;
        for (int i = 1; i <= n; ++i) {
            if (used[i]) continue;
            if (i % pos && pos % i) continue;      // ⭐ divisibility constraint
            used[i] = true;
            count += bt(pos + 1);
            used[i] = false;
        }
        return count;
    };
    return bt(1);
}
```

---

## Part 3 — Bit Manipulation

### 🧠 The essential tricks

```cpp
x & 1                      // is x odd?
x >> 1                     // divide by 2
x << 1                     // multiply by 2
x & (x - 1)                // ⭐ clear the LOWEST set bit
x & (-x)                   // ⭐ isolate the LOWEST set bit
x | (x + 1)                // set the lowest zero bit
(x & (x - 1)) == 0         // is x a power of two? (for x > 0)
__builtin_popcount(x)      // count set bits
__builtin_ctz(x)           // count trailing zeros
```

```
   WHY x & (x-1) clears the lowest set bit:

   x     = 1011000
   x - 1 = 1010111    ← borrowing flips the lowest 1 and everything below it
   ─────────────────
   AND   = 1010000    ← the lowest 1 is gone ✅

   WHY x & (-x) isolates it:

   x     = 1011000
   -x    = 0101000    ← two's complement = ~x + 1
   ─────────────────
   AND   = 0001000    ← only the lowest 1 survives ✅
```

```
   XOR PROPERTIES — the foundation of many tricks

   a ^ a = 0        anything XORed with itself vanishes
   a ^ 0 = a        XOR with zero is identity
   commutative and associative → order doesn't matter

   ⭐ Consequence: XOR everything together and pairs cancel out,
     leaving only the unpaired element.
```

---

### 33. Single Number 🟢
```cpp
int singleNumber(vector<int>& nums) {
    int r = 0;
    for (int x : nums) r ^= x;                     // pairs cancel, the loner survives
    return r;
}
```

---

### 34. Single Number II (others appear 3×) 🟡
```cpp
int singleNumber(vector<int>& nums) {
    int ones = 0, twos = 0;
    for (int x : nums) {
        ones = (ones ^ x) & ~twos;                 // ⭐ a 2-bit counter, per bit position
        twos = (twos ^ x) & ~ones;
    }
    return ones;
}
```
**Key insight:** `ones` and `twos` together form a mod-3 counter for every bit position simultaneously. When a bit reaches count 3, both are cleared.

---

### 35. Single Number III (two loners) 🟡
```cpp
vector<int> singleNumber(vector<int>& nums) {
    long long xorAll = 0;
    for (int x : nums) xorAll ^= x;                // = a ^ b (the two unique numbers)

    int lowbit = xorAll & (-xorAll);               // ⭐ a bit where a and b DIFFER
    int a = 0, b = 0;
    for (int x : nums) {
        if (x & lowbit) a ^= x;                    // partition into two groups
        else            b ^= x;
    }
    return {a, b};
}
```
**Key insight:** Any set bit in `a ^ b` is a position where they differ. Splitting on that bit puts `a` and `b` in different groups, and each group reduces to the simple single-number problem.

---

### 36. Number of 1 Bits 🟢
```cpp
int hammingWeight(uint32_t n) {
    int count = 0;
    while (n) { n &= (n - 1); ++count; }           // ⭐ loops once PER SET BIT
    return count;
}
```

---

### 37. Counting Bits 🟢
```cpp
vector<int> countBits(int n) {
    vector<int> dp(n + 1, 0);
    for (int i = 1; i <= n; ++i)
        dp[i] = dp[i >> 1] + (i & 1);              // ⭐ i's bits = (i/2)'s bits + last bit
    return dp;
}
```

---

### 38. Reverse Bits 🟢
```cpp
uint32_t reverseBits(uint32_t n) {
    uint32_t r = 0;
    for (int i = 0; i < 32; ++i) {
        r = (r << 1) | (n & 1);                    // shift result left, pull n's low bit
        n >>= 1;
    }
    return r;
}
```

---

### 39. Missing Number 🟢
```cpp
int missingNumber(vector<int>& nums) {
    int r = nums.size();
    for (int i = 0; i < (int)nums.size(); ++i) r ^= i ^ nums[i];
    return r;
}
```

---

### 40. Sum of Two Integers (no + operator) 🟡
```cpp
int getSum(int a, int b) {
    while (b) {
        unsigned carry = (unsigned)(a & b) << 1;   // ⭐ AND finds carry positions
        a = a ^ b;                                 // ⭐ XOR is addition without carry
        b = carry;
    }
    return a;
}
```

```
   5 + 3:
     a=101  b=011   →  xor=110  carry=(001)<<1=010
     a=110  b=010   →  xor=100  carry=(010)<<1=100
     a=100  b=100   →  xor=000  carry=(100)<<1=1000
     a=000  b=1000  →  xor=1000 carry=0
     a=1000 = 8 ✅
```

---

### 41. Maximum XOR of Two Numbers 🟡
```cpp
int findMaximumXOR(vector<int>& nums) {
    int mx = 0, mask = 0;
    for (int bit = 31; bit >= 0; --bit) {          // ⭐ greedy, high bit first
        mask |= (1 << bit);
        unordered_set<int> prefixes;
        for (int x : nums) prefixes.insert(x & mask);

        int candidate = mx | (1 << bit);           // can we achieve this bit?
        for (int p : prefixes)
            if (prefixes.count(p ^ candidate)) { mx = candidate; break; }
    }
    return mx;
}
```
**Key insight:** Build the answer bit by bit from the top. `a ^ b == c` implies `a ^ c == b`, so checking whether a needed partner exists is a hash-set lookup.

---

### 42. Subsets via Bitmask 🟡
```cpp
vector<vector<int>> subsets(vector<int>& nums) {
    int n = nums.size();
    vector<vector<int>> out;
    for (int mask = 0; mask < (1 << n); ++mask) {  // ⭐ every integer IS a subset
        vector<int> cur;
        for (int i = 0; i < n; ++i)
            if (mask & (1 << i)) cur.push_back(nums[i]);
        out.push_back(cur);
    }
    return out;
}
```

---

## Part 4 — Math

### 43. Pow(x, n) — Fast Exponentiation 🟡
```cpp
double myPow(double x, int n) {
    long long N = n;
    if (N < 0) { x = 1 / x; N = -N; }              // ⭐ long long avoids INT_MIN overflow
    double result = 1;
    while (N) {
        if (N & 1) result *= x;                    // this bit is set → multiply in
        x *= x;                                     // square the base
        N >>= 1;
    }
    return result;
}
```

```
   x¹³ where 13 = 1101 in binary

   13 = 8 + 4 + 1   →   x¹³ = x⁸ · x⁴ · x¹

   bit   base       take?   result
   1     x          ✅      x
   0     x²         ❌      x
   1     x⁴         ✅      x⁵
   1     x⁸         ✅      x¹³ ✅

   O(log n) multiplications instead of O(n)
```

---

### 44. Sqrt(x) 🟢
```cpp
int mySqrt(int x) {
    long long lo = 0, hi = x;
    while (lo <= hi) {
        long long mid = lo + (hi - lo) / 2;
        if (mid * mid <= x) lo = mid + 1;
        else hi = mid - 1;
    }
    return hi;                                     // ⭐ hi lands on the floor
}
```

---

### 45. Count Primes (Sieve of Eratosthenes) 🟡
```cpp
int countPrimes(int n) {
    if (n < 3) return 0;
    vector<bool> composite(n, false);
    int count = 0;
    for (int i = 2; i < n; ++i) {
        if (composite[i]) continue;
        ++count;
        for (long long j = (long long)i * i; j < n; j += i)   // ⭐ start at i², not 2i
            composite[j] = true;
    }
    return count;
}
```

```
   Sieve for n = 20:

   2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19
   ✓  ✓  ✗  ✓  ✗  ✓  ✗  ✗  ✗  ✓  ✗  ✓  ✗  ✗  ✗  ✓  ✗  ✓

   Start at i² because anything smaller (2i, 3i, ...) was
   already crossed off by a smaller prime factor.
```
**Complexity:** O(n log log n).

---

### Bonus: GCD / LCM
```cpp
int gcd(int a, int b) { return b ? gcd(b, a % b) : a; }        // Euclid
long long lcm(int a, int b) { return (long long)a / gcd(a, b) * b; }
//                                            ⭐ divide FIRST to avoid overflow
```

---

### Bonus: Modular Arithmetic
```cpp
const int MOD = 1e9 + 7;
long long addMod(long long a, long long b) { return (a + b) % MOD; }
long long mulMod(long long a, long long b) { return a % MOD * (b % MOD) % MOD; }

long long powMod(long long b, long long e, long long m) {
    long long r = 1; b %= m;
    while (e) { if (e & 1) r = r * b % m; b = b * b % m; e >>= 1; }
    return r;
}
// Modular inverse when m is prime (Fermat's little theorem):
long long inv(long long a) { return powMod(a, MOD - 2, MOD); }
```

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║        GREEDY · BACKTRACKING · BITS · MATH — PATTERN RECALL       ║
╠═══════════════════════════════════════════════════════════════════╣
║ GREEDY — must be PROVEN, not assumed                              ║
║   exchange argument · greedy stays ahead                          ║
║   ⭐ intervals: sort by END for max-count problems                 ║
║   ⭐ two opposing constraints → TWO PASSES + max() (candy)         ║
║   ⭐ "restart after failure" (gas station) → O(n) not O(n²)        ║
║   FAILS on: 0/1 knapsack, coin change with odd denominations      ║
╠═══════════════════════════════════════════════════════════════════╣
║ BACKTRACKING — choose → explore → UN-CHOOSE                       ║
║   ⭐ pruning is what makes it fast, not the recursion              ║
║   duplicates → SORT, then skip `i > start && a[i]==a[i-1]`        ║
║   combinations: bt(i+1) no reuse · bt(i) allows reuse             ║
║   N-Queens: r+c and r-c identify the two diagonals                ║
║   word search UNMARKS · island counting does NOT                  ║
║   exponential blowup → add memoization (word break II)            ║
╠═══════════════════════════════════════════════════════════════════╣
║ BITS                                                              ║
║   x & (x-1)  clears the lowest set bit  → popcount loop           ║
║   x & (-x)   isolates the lowest set bit → partitioning trick     ║
║   XOR cancels pairs → single number family                        ║
║   two loners → split the array on a differing bit                 ║
║   subsets → iterate mask from 0 to 2ⁿ-1                           ║
╠═══════════════════════════════════════════════════════════════════╣
║ MATH                                                              ║
║   fast pow → square the base, consume exponent bits, O(log n)     ║
║   sieve → start crossing off at i², O(n log log n)                ║
║   lcm → DIVIDE before multiplying to avoid overflow               ║
║   modular inverse (prime m) → powMod(a, m-2, m)                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Master Problem Index →](INDEX.md)
