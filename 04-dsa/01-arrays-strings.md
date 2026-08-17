# 📊 Arrays & Strings — 70 Problems

> The largest category and the most common in interviews. Master the four sub-patterns here — traversal tricks, prefix sums, in-place manipulation, and matrix operations — and a third of all interview problems become routine.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 📑 Sections

| Section | Problems | Focus |
|---|---|---|
| [A. Fundamentals](#a-fundamentals) | 1-12 | Basic traversal, searching, in-place |
| [B. Prefix Sum & Ranges](#b-prefix-sum--ranges) | 13-24 | Cumulative techniques |
| [C. Sorting-Based](#c-sorting-based) | 25-34 | Sort then solve |
| [D. In-Place Manipulation](#d-in-place-manipulation) | 35-44 | O(1) space tricks |
| [E. Matrix](#e-matrix) | 45-54 | 2D problems |
| [F. Strings](#f-strings) | 55-70 | Parsing, transformation, matching |

---

## A. Fundamentals

### 1. Two Sum 🟢
> Given `nums` and `target`, return indices of two numbers adding to target.

**Approach:** One-pass hash map. For each element, check if its complement was already seen.

```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int,int> seen;                 // value -> index
    for (int i = 0; i < (int)nums.size(); ++i) {
        int need = target - nums[i];
        auto it = seen.find(need);
        if (it != seen.end()) return {it->second, i};
        seen[nums[i]] = i;                       // insert AFTER checking
    }
    return {};
}
```
**Complexity:** O(n) time, O(n) space.
**Key insight:** Inserting after checking handles `target = 2*x` correctly — you can't pair an element with itself.

---

### 2. Best Time to Buy and Sell Stock 🟢
> One transaction. Maximize profit.

**Approach:** Track the minimum price so far; the best profit ending at `i` is `price[i] - minSoFar`.

```cpp
int maxProfit(vector<int>& p) {
    int lo = INT_MAX, best = 0;
    for (int x : p) {
        lo = min(lo, x);
        best = max(best, x - lo);
    }
    return best;
}
```
**Complexity:** O(n) / O(1).
**Key insight:** This is Kadane's algorithm on the difference array in disguise.

---

### 3. Contains Duplicate 🟢
```cpp
bool containsDuplicate(vector<int>& nums) {
    unordered_set<int> s(nums.begin(), nums.end());
    return s.size() != nums.size();
}
```
**Complexity:** O(n) / O(n). Sorting gives O(n log n) / O(1) if space matters.

---

### 4. Product of Array Except Self 🟡
> Return array where `out[i]` = product of all elements except `nums[i]`. No division, O(n).

#### 💬 Think of it like this
The obvious solution is to multiply everything and divide by `nums[i]` — but division is banned, and it breaks on zeros anyway.

So think about what "everything except me" actually means. Standing at position `i`, the product of everything except you is **everything to your left, times everything to your right.** Two independent halves.

So make two passes. On the first, sweep left to right and record the running product of everything before you. On the second, sweep right to left and multiply in the running product of everything after you. When both passes finish, each cell holds left × right — which is exactly the answer.

#### 📊 Watching it work on `[1, 2, 3, 4]`

```
   PASS 1 — sweep LEFT to RIGHT, storing "product of everything before me"

   index:      0      1      2      3
   nums:       1      2      3      4
              ┌──────┬──────┬──────┬──────┐
   out:       │  1   │  1   │  2   │  6   │
              └──────┴──────┴──────┴──────┘
                 ▲      ▲      ▲      ▲
             nothing   just   1×2    1×2×3
             before     1
              me

   PASS 2 — sweep RIGHT to LEFT, multiplying in "product of everything after me"

   suffix:     24     12     4      1
               ▲      ▲      ▲      ▲
             2×3×4   3×4     4    nothing after me

              ┌──────┬──────┬──────┬──────┐
   out:       │ 1×24 │ 1×12 │ 2×4  │ 6×1  │
              │ = 24 │ = 12 │ = 8  │ = 6  │
              └──────┴──────┴──────┴──────┘

   Check index 2:  everything except 3  =  1×2×4  =  8  ✅
```

```cpp
vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> out(n, 1);

    // PASS 1: out[i] = product of everything to the LEFT of i
    int pre = 1;
    for (int i = 0; i < n; ++i) {
        out[i] = pre;        // store BEFORE updating — nums[i] must be excluded
        pre *= nums[i];
    }

    // PASS 2: multiply in the product of everything to the RIGHT of i
    int suf = 1;
    for (int i = n - 1; i >= 0; --i) {
        out[i] *= suf;       // same ordering rule, mirrored
        suf *= nums[i];
    }

    return out;
}
```
**Complexity:** O(n) time, O(1) extra space (the output array doesn't count).
**Why it handles zeros for free:** a zero simply makes one of the two running products zero, which propagates correctly with no special case — unlike the division approach, which would need to count zeros and branch.

---

### 5. Maximum Subarray (Kadane) 🟡
> Contiguous subarray with the largest sum.

```cpp
int maxSubArray(vector<int>& nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < (int)nums.size(); ++i) {
        cur = max(nums[i], cur + nums[i]);       // extend or restart
        best = max(best, cur);
    }
    return best;
}
```
**Complexity:** O(n) / O(1).
**Key insight:** At each position, either extend the previous subarray or start fresh. If the running sum is negative, it can only hurt — drop it.

**Variant — return the indices:**
```cpp
int cur = nums[0], best = nums[0], s = 0, bl = 0, br = 0;
for (int i = 1; i < n; ++i) {
    if (nums[i] > cur + nums[i]) { cur = nums[i]; s = i; }
    else cur += nums[i];
    if (cur > best) { best = cur; bl = s; br = i; }
}
```

---

### 6. Maximum Product Subarray 🟡
> Largest product of a contiguous subarray.

**Approach:** Track both max and min — a negative number swaps them.

```cpp
int maxProduct(vector<int>& nums) {
    int mx = nums[0], mn = nums[0], best = nums[0];
    for (int i = 1; i < (int)nums.size(); ++i) {
        int x = nums[i];
        if (x < 0) swap(mx, mn);                 // negative flips the roles
        mx = max(x, mx * x);
        mn = min(x, mn * x);
        best = max(best, mx);
    }
    return best;
}
```
**Complexity:** O(n) / O(1).
**Key insight:** Unlike sums, the *minimum* matters — a large negative times a negative becomes a large positive.

---

### 7. Find Minimum in Rotated Sorted Array 🟡
```cpp
int findMin(vector<int>& a) {
    int lo = 0, hi = a.size() - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] > a[hi]) lo = mid + 1;        // min is to the right
        else hi = mid;                            // min is at mid or left
    }
    return a[lo];
}
```
**Complexity:** O(log n) / O(1).
**Key insight:** Compare with `a[hi]`, not `a[lo]` — comparing with `lo` is ambiguous when the array isn't rotated.

---

### 8. Search in Rotated Sorted Array 🟡
```cpp
int search(vector<int>& a, int t) {
    int lo = 0, hi = a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == t) return mid;

        if (a[lo] <= a[mid]) {                   // LEFT half is sorted
            if (a[lo] <= t && t < a[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {                                  // RIGHT half is sorted
            if (a[mid] < t && t <= a[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```
**Key insight:** One half is always sorted. Determine which, check if the target lies within it, and discard the other half.

---

### 9. Search in Rotated Sorted Array II (with duplicates) 🟡
```cpp
bool search(vector<int>& a, int t) {
    int lo = 0, hi = a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == t) return true;

        if (a[lo] == a[mid] && a[mid] == a[hi]) { ++lo; --hi; continue; }  // ⭐

        if (a[lo] <= a[mid]) {
            if (a[lo] <= t && t < a[mid]) hi = mid - 1; else lo = mid + 1;
        } else {
            if (a[mid] < t && t <= a[hi]) lo = mid + 1; else hi = mid - 1;
        }
    }
    return false;
}
```
**Complexity:** O(log n) average, **O(n) worst** (all duplicates).
**Key insight:** Duplicates break the "one half is sorted" guarantee — `[3,1,3,3,3]`. When ends match the middle, shrink linearly.

---

### 10. Find First and Last Position of Element 🟡
```cpp
vector<int> searchRange(vector<int>& a, int t) {
    auto lo = lower_bound(a.begin(), a.end(), t);
    if (lo == a.end() || *lo != t) return {-1, -1};
    auto hi = upper_bound(a.begin(), a.end(), t);
    return {int(lo - a.begin()), int(hi - a.begin()) - 1};
}
```
**Key insight:** `lower_bound` finds the first `>= t`; `upper_bound` finds the first `> t`. Write them by hand if asked.

---

### 11. Missing Number 🟢
```cpp
int missingNumber(vector<int>& a) {
    int n = a.size(), res = n;
    for (int i = 0; i < n; ++i) res ^= i ^ a[i];   // XOR cancels pairs
    return res;
}
// Alternative: sum formula n(n+1)/2 - actual  (⚠️ overflow risk for large n)
```
**Key insight:** XOR of `0..n` with all array elements leaves only the missing one, since `a ^ a = 0`.

---

### 12. Single Number 🟢
```cpp
int singleNumber(vector<int>& a) {
    int r = 0;
    for (int x : a) r ^= x;
    return r;
}
```

---

## B. Prefix Sum & Ranges

### 13. Range Sum Query — Immutable 🟢
```cpp
class NumArray {
    vector<long long> pre;
public:
    NumArray(vector<int>& a) : pre(a.size() + 1, 0) {
        for (int i = 0; i < (int)a.size(); ++i) pre[i+1] = pre[i] + a[i];
    }
    int sumRange(int l, int r) { return pre[r+1] - pre[l]; }
};
```
**Complexity:** O(n) build, O(1) query.

---

### 14. Subarray Sum Equals K 🟡
> Count subarrays summing to `k`. Values may be negative (so no sliding window).

#### 💬 Think of it like this
Imagine walking along the array keeping a running total — the *prefix sum*. After each step you know "the sum of everything from the start up to here."

Now here's the key move. The sum of any subarray from `l` to `r` is just:

```
   sum(l..r)  =  prefix[r]  −  prefix[l-1]
```

So if you're standing at position `r` and you want subarrays ending here that sum to `k`, you need to find earlier positions where the prefix was exactly `prefix[r] − k`. Every one of those is a valid subarray.

That turns the problem into: *"how many times have I seen the value `prefix[r] − k` before?"* — which is a hash map lookup.

#### 📊 Tracing `[1, 2, 3]` with `k = 3`

```
   We keep a map: prefix_sum → how many times we've seen it
   Seeded with {0: 1} — the "empty prefix", so subarrays
   starting at index 0 are counted correctly.

   ┌───────────────────────────────────────────────────────────┐
   │ start:  sum = 0   map = {0:1}   answer = 0                 │
   ├───────────────────────────────────────────────────────────┤
   │ i=0, x=1:  sum = 1                                        │
   │            looking for sum−k = 1−3 = −2   → not in map    │
   │            map = {0:1, 1:1}          answer = 0           │
   ├───────────────────────────────────────────────────────────┤
   │ i=1, x=2:  sum = 3                                        │
   │            looking for 3−3 = 0   → ⭐ found, count 1       │
   │            (that's the subarray [1,2])   answer = 1        │
   │            map = {0:1, 1:1, 3:1}                          │
   ├───────────────────────────────────────────────────────────┤
   │ i=2, x=3:  sum = 6                                        │
   │            looking for 6−3 = 3   → ⭐ found, count 1       │
   │            (that's the subarray [3])     answer = 2        │
   │            map = {0:1, 1:1, 3:1, 6:1}                     │
   └───────────────────────────────────────────────────────────┘

   ANSWER = 2   →   [1,2] and [3]  ✅
```

#### Why the `{0: 1}` seed is essential

```
   Without it, a subarray starting at index 0 is missed.

   In the trace above, at i=1 the running sum is exactly 3.
   The subarray [1,2] uses NOTHING before it — so the "prefix
   before it" is the empty prefix, whose sum is 0.

   ⭐ Seeding {0: 1} says "I have seen a prefix summing to 0,
     once — the empty one." That makes the arithmetic work.
```

⚠️ **Why sliding window fails here:** with negative numbers, extending the window doesn't monotonically increase the sum, so there's no valid shrink condition. Prefix sums don't care about sign.

```cpp
int subarraySum(vector<int>& a, int k) {
    unordered_map<long long,int> cnt{{0, 1}};    // ⭐ empty prefix counts once
    long long sum = 0; int ans = 0;
    for (int x : a) {
        sum += x;
        auto it = cnt.find(sum - k);
        if (it != cnt.end()) ans += it->second;
        cnt[sum]++;
    }
    return ans;
}
```
**Complexity:** O(n) / O(n).
**Key insight:** `sum(l..r) = pre[r] - pre[l-1]`. So for each `r`, count how many earlier prefixes equal `pre[r] - k`. The `{0,1}` seed handles subarrays starting at index 0.

⚠️ **Sliding window does NOT work here** because negatives mean growing the window doesn't monotonically grow the sum.

---

### 15. Continuous Subarray Sum (multiple of k) 🟡
```cpp
bool checkSubarraySum(vector<int>& a, int k) {
    unordered_map<int,int> first{{0, -1}};       // remainder -> earliest index
    int sum = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        sum = (sum + a[i]) % k;
        auto it = first.find(sum);
        if (it != first.end()) { if (i - it->second >= 2) return true; }
        else first[sum] = i;                      // keep the EARLIEST only
    }
    return false;
}
```
**Key insight:** Two prefixes with the same remainder mod k means the subarray between them is divisible by k. Store only the earliest index to maximize length.

---

### 16. Subarray Sums Divisible by K 🟡
```cpp
int subarraysDivByK(vector<int>& a, int k) {
    vector<int> cnt(k, 0);
    cnt[0] = 1;
    int sum = 0, ans = 0;
    for (int x : a) {
        sum = ((sum + x) % k + k) % k;            // ⭐ handle negatives
        ans += cnt[sum]++;
    }
    return ans;
}
```
**Key insight:** C++ `%` can return negative — normalize with `((x % k) + k) % k`.

---

### 17. Contiguous Array (equal 0s and 1s) 🟡
```cpp
int findMaxLength(vector<int>& a) {
    unordered_map<int,int> first{{0, -1}};
    int sum = 0, best = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        sum += (a[i] == 1 ? 1 : -1);              // ⭐ map 0 to -1
        auto it = first.find(sum);
        if (it != first.end()) best = max(best, i - it->second);
        else first[sum] = i;
    }
    return best;
}
```
**Key insight:** Mapping 0→−1 turns "equal counts" into "sum equals zero," which is the standard prefix-sum problem.

---

### 18. Maximum Size Subarray Sum Equals K 🟡
```cpp
int maxSubArrayLen(vector<int>& a, int k) {
    unordered_map<long long,int> first{{0, -1}};
    long long sum = 0; int best = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        sum += a[i];
        auto it = first.find(sum - k);
        if (it != first.end()) best = max(best, i - it->second);
        if (!first.count(sum)) first[sum] = i;    // earliest index only
    }
    return best;
}
```

---

### 19. Range Addition (difference array) 🟡
```cpp
vector<int> getModifiedArray(int n, vector<vector<int>>& updates) {
    vector<int> diff(n + 1, 0);
    for (auto& u : updates) { diff[u[0]] += u[2]; diff[u[1] + 1] -= u[2]; }
    vector<int> out(n);
    int run = 0;
    for (int i = 0; i < n; ++i) { run += diff[i]; out[i] = run; }
    return out;
}
```
**Complexity:** O(u + n) instead of O(u·n).
**Key insight:** The difference array turns range updates into two point updates.

---

### 20. Corporate Flight Bookings 🟡
```cpp
vector<int> corpFlightBookings(vector<vector<int>>& b, int n) {
    vector<int> d(n + 1, 0);
    for (auto& x : b) { d[x[0]-1] += x[2]; d[x[1]] -= x[2]; }
    for (int i = 1; i < n; ++i) d[i] += d[i-1];
    d.pop_back();
    return d;
}
```

---

### 21. Car Pooling 🟡
```cpp
bool carPooling(vector<vector<int>>& trips, int cap) {
    vector<int> d(1001, 0);
    for (auto& t : trips) { d[t[1]] += t[0]; d[t[2]] -= t[0]; }
    int cur = 0;
    for (int x : d) { cur += x; if (cur > cap) return false; }
    return true;
}
```

---

### 22. Find Pivot Index 🟢
```cpp
int pivotIndex(vector<int>& a) {
    long long total = accumulate(a.begin(), a.end(), 0LL), left = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        if (left == total - left - a[i]) return i;
        left += a[i];
    }
    return -1;
}
```

---

### 23. Range Sum Query 2D — Immutable 🟡
```cpp
class NumMatrix {
    vector<vector<long long>> pre;
public:
    NumMatrix(vector<vector<int>>& m) {
        int R = m.size(), C = R ? m[0].size() : 0;
        pre.assign(R + 1, vector<long long>(C + 1, 0));
        for (int i = 0; i < R; ++i)
            for (int j = 0; j < C; ++j)
                pre[i+1][j+1] = m[i][j] + pre[i][j+1] + pre[i+1][j] - pre[i][j];
    }
    int sumRegion(int r1, int c1, int r2, int c2) {
        return pre[r2+1][c2+1] - pre[r1][c2+1] - pre[r2+1][c1] + pre[r1][c1];
    }
};
```
**Key insight:** Inclusion-exclusion. Subtract the two overlapping strips, then add back the doubly-subtracted corner.

---

### 24. Maximum Sum Rectangle in a 2D Matrix 🔴
```cpp
int maxSumRectangle(vector<vector<int>>& m) {
    int R = m.size(), C = m[0].size(), best = INT_MIN;
    for (int top = 0; top < R; ++top) {
        vector<int> col(C, 0);
        for (int bot = top; bot < R; ++bot) {
            for (int c = 0; c < C; ++c) col[c] += m[bot][c];   // collapse rows
            // Kadane on the collapsed 1D array
            int cur = col[0], loc = col[0];
            for (int c = 1; c < C; ++c) {
                cur = max(col[c], cur + col[c]);
                loc = max(loc, cur);
            }
            best = max(best, loc);
        }
    }
    return best;
}
```
**Complexity:** O(R²·C).
**Key insight:** Fix the top and bottom rows, collapse the strip into a 1D array by summing columns, then run Kadane. This reduction from 2D to 1D is a reusable technique.

---

## C. Sorting-Based

### 25. Merge Sorted Array 🟢
> Merge `b` into `a` in place; `a` has trailing space.

```cpp
void merge(vector<int>& a, int m, vector<int>& b, int n) {
    int i = m - 1, j = n - 1, k = m + n - 1;
    while (j >= 0) {                              // ⭐ fill from the BACK
        a[k--] = (i >= 0 && a[i] > b[j]) ? a[i--] : b[j--];
    }
}
```
**Key insight:** Filling from the back avoids overwriting unprocessed elements — no extra space needed.

---

### 26. Sort Colors (Dutch National Flag) 🟡
> Sort an array of 0s, 1s, 2s in one pass.

```cpp
void sortColors(vector<int>& a) {
    int lo = 0, mid = 0, hi = a.size() - 1;
    while (mid <= hi) {
        if (a[mid] == 0) swap(a[lo++], a[mid++]);
        else if (a[mid] == 2) swap(a[mid], a[hi--]);   // ⭐ don't ++mid here
        else ++mid;
    }
}
```
**Key insight:** After swapping with `hi`, the incoming value is unexamined, so `mid` must not advance. After swapping with `lo`, the incoming value is already known to be 0 or 1.

---

### 27. Merge Intervals 🟡
```cpp
vector<vector<int>> merge(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end());
    vector<vector<int>> out;
    for (auto& c : iv) {
        if (out.empty() || out.back()[1] < c[0]) out.push_back(c);
        else out.back()[1] = max(out.back()[1], c[1]);
    }
    return out;
}
```

---

### 28. Insert Interval 🟡
```cpp
vector<vector<int>> insert(vector<vector<int>>& iv, vector<int> ni) {
    vector<vector<int>> out;
    int i = 0, n = iv.size();
    while (i < n && iv[i][1] < ni[0]) out.push_back(iv[i++]);       // before
    while (i < n && iv[i][0] <= ni[1]) {                             // overlapping
        ni[0] = min(ni[0], iv[i][0]);
        ni[1] = max(ni[1], iv[i][1]);
        ++i;
    }
    out.push_back(ni);
    while (i < n) out.push_back(iv[i++]);                            // after
    return out;
}
```
**Complexity:** O(n) — input is already sorted, no re-sort needed.

---

### 29. Non-overlapping Intervals 🟡
> Minimum removals to make intervals non-overlapping.

```cpp
int eraseOverlapIntervals(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end(), [](auto& a, auto& b){ return a[1] < b[1]; });  // ⭐ by END
    int end = INT_MIN, keep = 0;
    for (auto& x : iv) if (x[0] >= end) { end = x[1]; ++keep; }
    return iv.size() - keep;
}
```
**Key insight:** Classic activity selection. Sorting by **end** time is what makes greedy optimal — keeping the earliest-ending interval leaves the most room for the rest.

---

### 30. Meeting Rooms II 🟡
```cpp
int minMeetingRooms(vector<vector<int>>& iv) {
    vector<pair<int,int>> ev;
    for (auto& m : iv) { ev.push_back({m[0], 1}); ev.push_back({m[1], -1}); }
    sort(ev.begin(), ev.end());                   // -1 sorts before +1 at equal time
    int cur = 0, best = 0;
    for (auto& [t, d] : ev) { cur += d; best = max(best, cur); }
    return best;
}
```
**Key insight:** At equal timestamps, ends must process before starts (a room freed at 10 can be reused at 10). `pair` sorting gives this for free since `-1 < 1`.

---

### 31. Largest Number 🟡
```cpp
string largestNumber(vector<int>& a) {
    vector<string> s;
    for (int x : a) s.push_back(to_string(x));
    sort(s.begin(), s.end(), [](const string& x, const string& y){
        return x + y > y + x;                     // ⭐ custom order
    });
    if (s[0] == "0") return "0";                  // all zeros
    string out;
    for (auto& t : s) out += t;
    return out;
}
```
**Key insight:** Compare concatenations, not the numbers. "9" vs "34": `"934" > "349"`, so 9 comes first. This comparator is provably a valid strict weak ordering.

---

### 32. Sort Array by Parity 🟢
```cpp
vector<int> sortArrayByParity(vector<int>& a) {
    int i = 0, j = a.size() - 1;
    while (i < j) {
        if (a[i] % 2 > a[j] % 2) swap(a[i], a[j]);
        if (a[i] % 2 == 0) ++i;
        if (a[j] % 2 == 1) --j;
    }
    return a;
}
```

---

### 33. H-Index 🟡
```cpp
int hIndex(vector<int>& c) {
    int n = c.size();
    vector<int> bucket(n + 1, 0);
    for (int x : c) bucket[min(x, n)]++;          // counting sort
    int total = 0;
    for (int h = n; h >= 0; --h) {
        total += bucket[h];
        if (total >= h) return h;
    }
    return 0;
}
```
**Complexity:** O(n) with counting sort, vs O(n log n) sorting.

---

### 34. Top K Frequent Elements 🟡
```cpp
vector<int> topKFrequent(vector<int>& a, int k) {
    unordered_map<int,int> cnt;
    for (int x : a) cnt[x]++;

    int n = a.size();
    vector<vector<int>> bucket(n + 1);            // ⭐ bucket sort by frequency
    for (auto& [v, c] : cnt) bucket[c].push_back(v);

    vector<int> out;
    for (int f = n; f >= 1 && (int)out.size() < k; --f)
        for (int v : bucket[f]) {
            out.push_back(v);
            if ((int)out.size() == k) break;
        }
    return out;
}
```
**Complexity:** O(n) — bucket sort beats the O(n log k) heap approach because frequency is bounded by n.

---

## D. In-Place Manipulation

### 35. Remove Duplicates from Sorted Array 🟢
```cpp
int removeDuplicates(vector<int>& a) {
    if (a.empty()) return 0;
    int k = 1;
    for (int i = 1; i < (int)a.size(); ++i)
        if (a[i] != a[k-1]) a[k++] = a[i];
    return k;
}
```

---

### 36. Remove Duplicates from Sorted Array II (allow 2) 🟡
```cpp
int removeDuplicates(vector<int>& a) {
    int k = 0;
    for (int x : a)
        if (k < 2 || x != a[k-2]) a[k++] = x;     // ⭐ generalizes to k copies
    return k;
}
```
**Key insight:** Comparing against `a[k-2]` generalizes: for at most `m` copies, compare against `a[k-m]`.

---

### 37. Remove Element 🟢
```cpp
int removeElement(vector<int>& a, int val) {
    int k = 0;
    for (int x : a) if (x != val) a[k++] = x;
    return k;
}
```

---

### 38. Move Zeroes 🟢
```cpp
void moveZeroes(vector<int>& a) {
    int k = 0;
    for (int i = 0; i < (int)a.size(); ++i) if (a[i]) swap(a[k++], a[i]);
}
```
**Key insight:** Swapping (rather than assigning then zero-filling) keeps it to one pass.

---

### 39. Rotate Array 🟡
```cpp
void rotate(vector<int>& a, int k) {
    int n = a.size();
    k %= n;                                       // ⭐ k can exceed n
    reverse(a.begin(), a.end());
    reverse(a.begin(), a.begin() + k);
    reverse(a.begin() + k, a.end());
}
```
**Key insight:** Reverse all, then reverse each part. Three reversals, O(1) space.

```
   [1,2,3,4,5,6,7], k=3
   reverse all      → [7,6,5,4,3,2,1]
   reverse first 3  → [5,6,7,4,3,2,1]
   reverse rest     → [5,6,7,1,2,3,4] ✅
```

---

### 40. First Missing Positive 🔴
> Find the smallest missing positive integer in O(n) time, O(1) space.

#### 💬 Think of it like this
Start with an observation that makes the problem tractable. With `n` numbers, the answer **must** be somewhere in `1..n+1`. If the array happened to contain exactly 1 through n, the answer is n+1. Otherwise one of 1..n is missing. Nothing outside that range can possibly be the answer.

So you only care about values 1 through n — everything else (negatives, zeros, huge numbers) is noise.

Now, you'd like a hash set to check membership. But you're not allowed extra space. **So use the array itself as the hash set.** Put the value `v` at index `v-1`. Then "is 3 present?" becomes "is `a[2] == 3`?" — an O(1) check with no extra memory.

The placement pass is a series of swaps: look at the current slot, and if its value belongs somewhere else, swap it there. Repeat until the current slot holds something that belongs (or is out of range), then move on.

#### 📊 Watching the placement on `[3, 4, -1, 1]`

```
   Goal: put value v at index v-1

   ┌─────────────────────────────────────────────────────────────┐
   │ i=0:  a = [3, 4, -1, 1]                                     │
   │       a[0]=3 → belongs at index 2. Swap a[0] ↔ a[2].        │
   │       a = [-1, 4, 3, 1]                                     │
   │       ⭐ don't advance i — the incoming value is unexamined  │
   ├─────────────────────────────────────────────────────────────┤
   │ i=0:  a[0] = -1 → out of range, ignore. Advance.            │
   ├─────────────────────────────────────────────────────────────┤
   │ i=1:  a[1]=4 → belongs at index 3. Swap a[1] ↔ a[3].        │
   │       a = [-1, 1, 3, 4]                                     │
   ├─────────────────────────────────────────────────────────────┤
   │ i=1:  a[1]=1 → belongs at index 0. Swap a[1] ↔ a[0].        │
   │       a = [1, -1, 3, 4]                                     │
   ├─────────────────────────────────────────────────────────────┤
   │ i=1:  a[1] = -1 → out of range. Advance.                    │
   │ i=2:  a[2] = 3, and 3 belongs at index 2. ✅ Already home.   │
   │ i=3:  a[3] = 4, belongs at index 3. ✅ Already home.         │
   └─────────────────────────────────────────────────────────────┘

   FINAL SCAN — first index where a[i] != i+1

   index:      0     1     2     3
              ┌─────┬─────┬─────┬─────┐
   a:         │  1  │ -1  │  3  │  4  │
              └─────┴─────┴─────┴─────┘
   expected:     1     2     3     4
                       ▲
                  ⭐ MISMATCH at index 1 → answer is 2
```

#### Why this is O(n) despite the nested loop

```
   The `while` inside the `for` looks like it could be O(n²).
   It isn't.

   ⭐ Every swap places at least one value in its FINAL correct
     position, and a value once placed is never moved again.
     There are only n values, so there can be at most n swaps
     across the ENTIRE run.

   Total work = n iterations + at most n swaps = O(n).
```

```cpp
int firstMissingPositive(vector<int>& a) {
    int n = a.size();
    for (int i = 0; i < n; ++i) {
        while (a[i] > 0 && a[i] <= n && a[a[i] - 1] != a[i])
            swap(a[i], a[a[i] - 1]);              // cyclic sort placement
    }
    for (int i = 0; i < n; ++i) if (a[i] != i + 1) return i + 1;
    return n + 1;
}
```
**Complexity:** O(n) / O(1).
**Key insight:** The answer must be in `[1, n+1]`. Use the array itself as a hash table by placing value `v` at index `v-1`. Each swap places one value permanently, so total swaps ≤ n.

---

### 41. Find All Duplicates in an Array 🟡
```cpp
vector<int> findDuplicates(vector<int>& a) {
    vector<int> out;
    for (int x : a) {
        int i = abs(x) - 1;
        if (a[i] < 0) out.push_back(abs(x));      // seen before
        else a[i] = -a[i];                         // mark as seen
    }
    return out;
}
```
**Key insight:** Use the sign bit at index `v-1` as a "seen" marker — O(1) extra space.

---

### 42. Find All Numbers Disappeared in an Array 🟢
```cpp
vector<int> findDisappearedNumbers(vector<int>& a) {
    for (int x : a) a[abs(x) - 1] = -abs(a[abs(x) - 1]);
    vector<int> out;
    for (int i = 0; i < (int)a.size(); ++i) if (a[i] > 0) out.push_back(i + 1);
    return out;
}
```

---

### 43. Set Matrix Zeroes 🟡
```cpp
void setZeroes(vector<vector<int>>& m) {
    int R = m.size(), C = m[0].size();
    bool firstCol = false;

    for (int i = 0; i < R; ++i) {
        if (m[i][0] == 0) firstCol = true;        // ⭐ track column 0 separately
        for (int j = 1; j < C; ++j)
            if (m[i][j] == 0) { m[i][0] = 0; m[0][j] = 0; }   // use row/col 0 as flags
    }

    for (int i = R - 1; i >= 0; --i) {            // ⭐ backwards: flags stay intact
        for (int j = C - 1; j >= 1; --j)
            if (m[i][0] == 0 || m[0][j] == 0) m[i][j] = 0;
        if (firstCol) m[i][0] = 0;
    }
}
```
**Complexity:** O(R·C) / O(1).
**Key insight:** Store the flags in the matrix's own first row and column. Column 0 needs a separate boolean because `m[0][0]` is shared between the row-0 and column-0 flags.

---

### 44. Next Permutation 🟡
```cpp
void nextPermutation(vector<int>& a) {
    int n = a.size(), i = n - 2;
    while (i >= 0 && a[i] >= a[i+1]) --i;         // 1. find the pivot

    if (i >= 0) {
        int j = n - 1;
        while (a[j] <= a[i]) --j;                 // 2. rightmost element > pivot
        swap(a[i], a[j]);
    }
    reverse(a.begin() + i + 1, a.end());          // 3. make the suffix ascending
}
```
```
   [1,3,5,4,2]
    ↑ pivot (3, since 3 < 5)
   find rightmost > 3 → 4
   swap → [1,4,5,3,2]
   reverse suffix → [1,4,2,3,5] ✅
```
**Key insight:** The suffix after the pivot is always non-increasing, so reversing it produces the smallest arrangement.

---

## E. Matrix

### 45. Rotate Image (90° clockwise, in place) 🟡
```cpp
void rotate(vector<vector<int>>& m) {
    int n = m.size();
    for (int i = 0; i < n; ++i)                   // 1. transpose
        for (int j = i + 1; j < n; ++j)
            swap(m[i][j], m[j][i]);
    for (auto& row : m) reverse(row.begin(), row.end());   // 2. reverse each row
}
// Counter-clockwise: transpose, then reverse COLUMNS (reverse the row order).
```

---

### 46. Spiral Matrix 🟡
```cpp
vector<int> spiralOrder(vector<vector<int>>& m) {
    if (m.empty()) return {};
    int top = 0, bot = m.size() - 1, left = 0, right = m[0].size() - 1;
    vector<int> out;
    while (top <= bot && left <= right) {
        for (int j = left; j <= right; ++j) out.push_back(m[top][j]);
        ++top;
        for (int i = top; i <= bot; ++i) out.push_back(m[i][right]);
        --right;
        if (top <= bot) {                          // ⭐ re-check after shrinking
            for (int j = right; j >= left; --j) out.push_back(m[bot][j]);
            --bot;
        }
        if (left <= right) {
            for (int i = bot; i >= top; --i) out.push_back(m[i][left]);
            ++left;
        }
    }
    return out;
}
```
**Key insight:** The two re-checks prevent double-visiting on single-row or single-column remainders.

---

### 47. Spiral Matrix II (generate) 🟡
```cpp
vector<vector<int>> generateMatrix(int n) {
    vector<vector<int>> m(n, vector<int>(n));
    int v = 1, top = 0, bot = n - 1, left = 0, right = n - 1;
    while (v <= n * n) {
        for (int j = left; j <= right; ++j) m[top][j] = v++;
        ++top;
        for (int i = top; i <= bot; ++i) m[i][right] = v++;
        --right;
        for (int j = right; j >= left; --j) m[bot][j] = v++;
        --bot;
        for (int i = bot; i >= top; --i) m[i][left] = v++;
        ++left;
    }
    return m;
}
```

---

### 48. Search a 2D Matrix 🟡
> Rows sorted, first element of each row > last of the previous.

```cpp
bool searchMatrix(vector<vector<int>>& m, int t) {
    int R = m.size(), C = m[0].size();
    int lo = 0, hi = R * C - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        int v = m[mid / C][mid % C];              // ⭐ treat as a flat array
        if (v == t) return true;
        if (v < t) lo = mid + 1; else hi = mid - 1;
    }
    return false;
}
```

---

### 49. Search a 2D Matrix II 🟡
> Rows and columns each sorted, but rows don't chain.

```cpp
bool searchMatrix(vector<vector<int>>& m, int t) {
    int r = 0, c = m[0].size() - 1;               // ⭐ start top-RIGHT
    while (r < (int)m.size() && c >= 0) {
        if (m[r][c] == t) return true;
        if (m[r][c] > t) --c;                     // eliminate a column
        else ++r;                                  // eliminate a row
    }
    return false;
}
```
**Complexity:** O(R + C).
**Key insight:** The top-right corner is the only position where moving left strictly decreases and moving down strictly increases — each step eliminates an entire row or column.

---

### 50. Diagonal Traverse 🟡
```cpp
vector<int> findDiagonalOrder(vector<vector<int>>& m) {
    if (m.empty()) return {};
    int R = m.size(), C = m[0].size();
    vector<int> out;
    for (int d = 0; d < R + C - 1; ++d) {
        if (d % 2 == 0) {                          // up-right
            int r = min(d, R - 1), c = d - r;
            while (r >= 0 && c < C) out.push_back(m[r--][c++]);
        } else {                                   // down-left
            int c = min(d, C - 1), r = d - c;
            while (c >= 0 && r < R) out.push_back(m[r++][c--]);
        }
    }
    return out;
}
```
**Key insight:** All cells on a diagonal share `r + c = d`.

---

### 51. Game of Life (in place) 🟡
```cpp
void gameOfLife(vector<vector<int>>& b) {
    int R = b.size(), C = b[0].size();
    // Encode: bit0 = current, bit1 = next
    for (int i = 0; i < R; ++i)
        for (int j = 0; j < C; ++j) {
            int live = 0;
            for (int di = -1; di <= 1; ++di)
                for (int dj = -1; dj <= 1; ++dj) {
                    if (!di && !dj) continue;
                    int ni = i + di, nj = j + dj;
                    if (ni >= 0 && ni < R && nj >= 0 && nj < C)
                        live += b[ni][nj] & 1;     // ⭐ read the ORIGINAL bit
                }
            if ((b[i][j] & 1) && (live == 2 || live == 3)) b[i][j] |= 2;
            if (!(b[i][j] & 1) && live == 3)               b[i][j] |= 2;
        }
    for (auto& row : b) for (int& x : row) x >>= 1;
}
```
**Key insight:** Store two states in one integer using bits — the low bit holds the current state so neighbors read correctly, the high bit accumulates the next state.

---

### 52. Transpose Matrix 🟢
```cpp
vector<vector<int>> transpose(vector<vector<int>>& m) {
    int R = m.size(), C = m[0].size();
    vector<vector<int>> t(C, vector<int>(R));
    for (int i = 0; i < R; ++i) for (int j = 0; j < C; ++j) t[j][i] = m[i][j];
    return t;
}
```

---

### 53. Valid Sudoku 🟡
```cpp
bool isValidSudoku(vector<vector<char>>& b) {
    bool row[9][9] = {}, col[9][9] = {}, box[9][9] = {};
    for (int i = 0; i < 9; ++i)
        for (int j = 0; j < 9; ++j) {
            if (b[i][j] == '.') continue;
            int d = b[i][j] - '1';
            int k = (i / 3) * 3 + j / 3;          // ⭐ box index
            if (row[i][d] || col[j][d] || box[k][d]) return false;
            row[i][d] = col[j][d] = box[k][d] = true;
        }
    return true;
}
```

---

### 54. Spiral Matrix III / Matrix Diagonal Sum 🟢
```cpp
int diagonalSum(vector<vector<int>>& m) {
    int n = m.size(), s = 0;
    for (int i = 0; i < n; ++i) {
        s += m[i][i];
        if (i != n - 1 - i) s += m[i][n - 1 - i];  // avoid double-counting the center
    }
    return s;
}
```

---

## F. Strings

### 55. Valid Anagram 🟢
```cpp
bool isAnagram(string s, string t) {
    if (s.size() != t.size()) return false;
    int c[26] = {};
    for (char x : s) c[x - 'a']++;
    for (char x : t) if (--c[x - 'a'] < 0) return false;
    return true;
}
```

---

### 56. Group Anagrams 🟡
```cpp
vector<vector<string>> groupAnagrams(vector<string>& v) {
    unordered_map<string, vector<string>> g;
    for (auto& s : v) {
        array<int,26> c{};
        for (char x : s) c[x - 'a']++;
        string key;
        for (int i = 0; i < 26; ++i) { key += '#'; key += to_string(c[i]); }
        g[key].push_back(s);                      // ⭐ count-key: O(L) not O(L log L)
    }
    vector<vector<string>> out;
    for (auto& [k, vec] : g) out.push_back(move(vec));
    return out;
}
```
**Complexity:** O(N·L) with count keys vs O(N·L log L) with sorted keys.

---

### 57. Valid Palindrome 🟢
```cpp
bool isPalindrome(string s) {
    int i = 0, j = s.size() - 1;
    while (i < j) {
        while (i < j && !isalnum(s[i])) ++i;
        while (i < j && !isalnum(s[j])) --j;
        if (tolower(s[i]) != tolower(s[j])) return false;
        ++i; --j;
    }
    return true;
}
```

---

### 58. Valid Palindrome II (delete at most one) 🟢
```cpp
bool check(const string& s, int i, int j) {
    while (i < j) { if (s[i++] != s[j--]) return false; }
    return true;
}
bool validPalindrome(string s) {
    int i = 0, j = s.size() - 1;
    while (i < j) {
        if (s[i] != s[j])
            return check(s, i + 1, j) || check(s, i, j - 1);   // try both deletions
        ++i; --j;
    }
    return true;
}
```

---

### 59. Longest Common Prefix 🟢
```cpp
string longestCommonPrefix(vector<string>& v) {
    if (v.empty()) return "";
    string p = v[0];
    for (int i = 1; i < (int)v.size(); ++i) {
        while (v[i].compare(0, p.size(), p) != 0) {
            p.pop_back();
            if (p.empty()) return "";
        }
    }
    return p;
}
```

---

### 60. String to Integer (atoi) 🟡
```cpp
int myAtoi(string s) {
    int i = 0, n = s.size();
    while (i < n && s[i] == ' ') ++i;
    int sign = 1;
    if (i < n && (s[i] == '+' || s[i] == '-')) sign = (s[i++] == '-') ? -1 : 1;

    long long r = 0;
    while (i < n && isdigit(s[i])) {
        r = r * 10 + (s[i++] - '0');
        if (sign == 1 && r > INT_MAX) return INT_MAX;          // ⭐ clamp early
        if (sign == -1 && -r < INT_MIN) return INT_MIN;
    }
    return (int)(sign * r);
}
```
**Key insight:** Clamp inside the loop, before the accumulator itself overflows.

---

### 61. Implement strStr (KMP) 🟡
```cpp
int strStr(string h, string n) {
    if (n.empty()) return 0;
    int m = n.size();

    // LPS: lps[i] = length of the longest proper prefix that is also a suffix
    vector<int> lps(m, 0);
    for (int i = 1, len = 0; i < m; ) {
        if (n[i] == n[len]) lps[i++] = ++len;
        else if (len) len = lps[len - 1];         // ⭐ fall back
        else lps[i++] = 0;
    }

    for (int i = 0, j = 0; i < (int)h.size(); ) {
        if (h[i] == n[j]) { ++i; ++j; if (j == m) return i - m; }
        else if (j) j = lps[j - 1];               // shift without moving i
        else ++i;
    }
    return -1;
}
```
**Complexity:** O(n + m).
**Key insight:** The LPS array lets you skip re-comparing characters you already matched — `i` never moves backwards.

---

### 62. Longest Palindromic Substring 🟡
```cpp
string longestPalindrome(string s) {
    int n = s.size(), bl = 0, blen = 1;
    if (n < 2) return s;

    auto expand = [&](int l, int r) {
        while (l >= 0 && r < n && s[l] == s[r]) { --l; ++r; }
        int len = r - l - 1;
        if (len > blen) { blen = len; bl = l + 1; }
    };

    for (int i = 0; i < n; ++i) { expand(i, i); expand(i, i + 1); }  // odd + even
    return s.substr(bl, blen);
}
```
**Complexity:** O(n²) / O(1). Manacher's gives O(n) but is rarely required.
**Key insight:** Expand around all 2n−1 possible centers (n characters plus n−1 gaps).

---

### 63. Palindromic Substrings (count) 🟡
```cpp
int countSubstrings(string s) {
    int n = s.size(), cnt = 0;
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < n && s[l] == s[r]) { ++cnt; --l; ++r; }
    };
    for (int i = 0; i < n; ++i) { expand(i, i); expand(i, i + 1); }
    return cnt;
}
```

---

### 64. Reverse Words in a String 🟡
```cpp
string reverseWords(string s) {
    reverse(s.begin(), s.end());
    int n = s.size(), k = 0;
    for (int i = 0; i < n; ) {
        while (i < n && s[i] == ' ') ++i;
        if (i == n) break;
        if (k) s[k++] = ' ';
        int start = k;
        while (i < n && s[i] != ' ') s[k++] = s[i++];
        reverse(s.begin() + start, s.begin() + k);   // ⭐ un-reverse each word
    }
    s.resize(k);
    return s;
}
```
**Key insight:** Reverse everything, then reverse each word back. Same trick as rotating an array.

---

### 65. Zigzag Conversion 🟡
```cpp
string convert(string s, int rows) {
    if (rows == 1) return s;
    vector<string> r(rows);
    int cur = 0, dir = -1;
    for (char c : s) {
        r[cur] += c;
        if (cur == 0 || cur == rows - 1) dir = -dir;   // bounce
        cur += dir;
    }
    string out;
    for (auto& x : r) out += x;
    return out;
}
```

---

### 66. Compare Version Numbers 🟡
```cpp
int compareVersion(string a, string b) {
    int i = 0, j = 0;
    while (i < (int)a.size() || j < (int)b.size()) {
        long long x = 0, y = 0;
        while (i < (int)a.size() && a[i] != '.') x = x * 10 + (a[i++] - '0');
        while (j < (int)b.size() && b[j] != '.') y = y * 10 + (b[j++] - '0');
        if (x != y) return x < y ? -1 : 1;
        ++i; ++j;                                  // skip the dots
    }
    return 0;
}
```
**Key insight:** Missing revisions are treated as 0 — the loop condition handles unequal lengths naturally.

---

### 67. Multiply Strings 🟡
```cpp
string multiply(string a, string b) {
    if (a == "0" || b == "0") return "0";
    int m = a.size(), n = b.size();
    vector<int> r(m + n, 0);

    for (int i = m - 1; i >= 0; --i)
        for (int j = n - 1; j >= 0; --j) {
            int mul = (a[i] - '0') * (b[j] - '0');
            int p1 = i + j, p2 = i + j + 1;        // ⭐ positions
            int sum = mul + r[p2];
            r[p2] = sum % 10;
            r[p1] += sum / 10;                     // carry
        }

    string out;
    for (int x : r) if (!(out.empty() && x == 0)) out += ('0' + x);
    return out;
}
```
**Key insight:** Digits `i` and `j` contribute to result positions `i+j` and `i+j+1` — the standard grade-school layout.

---

### 68. Add Binary / Add Strings 🟢
```cpp
string addStrings(string a, string b) {
    string out;
    int i = a.size() - 1, j = b.size() - 1, carry = 0;
    while (i >= 0 || j >= 0 || carry) {
        int s = carry;
        if (i >= 0) s += a[i--] - '0';
        if (j >= 0) s += b[j--] - '0';
        out += ('0' + s % 10);
        carry = s / 10;
    }
    reverse(out.begin(), out.end());
    return out;
}
```

---

### 69. Text Justification 🔴
```cpp
vector<string> fullJustify(vector<string>& w, int W) {
    vector<string> out;
    int i = 0, n = w.size();
    while (i < n) {
        int j = i, len = 0;
        while (j < n && len + (int)w[j].size() + (j - i) <= W) len += w[j++].size();

        int words = j - i, spaces = W - len;
        string line;
        if (words == 1 || j == n) {                // left-justify last line
            for (int k = i; k < j; ++k) { line += w[k]; if (k + 1 < j) line += ' '; }
            line += string(W - line.size(), ' ');
        } else {
            int base = spaces / (words - 1), extra = spaces % (words - 1);
            for (int k = i; k < j; ++k) {
                line += w[k];
                if (k + 1 < j) line += string(base + (k - i < extra ? 1 : 0), ' ');
            }
        }
        out.push_back(line);
        i = j;
    }
    return out;
}
```
**Key insight:** Extra spaces go to the leftmost gaps — that's the `(k - i < extra)` term.

---

### 70. Encode and Decode Strings 🟡
```cpp
string encode(vector<string>& v) {
    string out;
    for (auto& s : v) out += to_string(s.size()) + "#" + s;   // ⭐ length prefix
    return out;
}
vector<string> decode(string s) {
    vector<string> out;
    int i = 0;
    while (i < (int)s.size()) {
        int j = s.find('#', i);
        int len = stoi(s.substr(i, j - i));
        out.push_back(s.substr(j + 1, len));
        i = j + 1 + len;
    }
    return out;
}
```
**Key insight:** Length-prefixing is delimiter-free and works for any content, including strings containing `#`. Any pure-delimiter scheme is breakable.

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║              ARRAYS & STRINGS — PATTERN RECALL                    ║
╠═══════════════════════════════════════════════════════════════════╣
║ "subarray sum = k" with NEGATIVES  → prefix sum + hashmap         ║
║ "subarray sum = k" all POSITIVE    → sliding window               ║
║ "range updates"                    → difference array             ║
║ "max subarray"                     → Kadane                       ║
║ "max product subarray"             → track BOTH max and min       ║
║ "except self"                      → prefix × suffix, two sweeps  ║
║ "values in 1..n, find missing/dup" → cyclic sort or sign marking  ║
║ "rotated sorted array"             → binary search, one half      ║
║                                      is always sorted             ║
║ "in-place O(1) space"              → reverse tricks, sign bits,   ║
║                                      use the matrix's own row 0   ║
║ "sorted 2D, rows+cols"             → start from the TOP-RIGHT     ║
║ "palindromic substring"            → expand around 2n-1 centers   ║
║ "anagram grouping"                 → count array as the key       ║
║ "serialize strings"                → length prefix, not delimiter ║
╠═══════════════════════════════════════════════════════════════════╣
║ ALWAYS: check empty · check overflow (use long long) ·            ║
║   mid = lo + (hi-lo)/2 · normalize negative % with ((x%k)+k)%k    ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Hashing →](02-hashing.md)
