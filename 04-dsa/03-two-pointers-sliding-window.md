# 👉👈 Two Pointers & Sliding Window — 40 Problems

> Two techniques that turn O(n²) into O(n). The distinction: **two pointers** usually converge from opposite ends on sorted data; **sliding window** maintains a contiguous range with an invariant.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 The Templates

### Two pointers — opposite ends
```cpp
int l = 0, r = n - 1;
while (l < r) {
    if (condition(a[l], a[r])) return {l, r};
    if (needBigger)  ++l;                          // sorted: move left up
    else             --r;                          // sorted: move right down
}
```
**Requires sorted input.** Each comparison eliminates an entire row or column of the O(n²) pair space.

### Sliding window — the universal template
```cpp
int left = 0;
Counter window;
for (int right = 0; right < n; ++right) {
    window.add(a[right]);                          // ── 1. EXPAND

    while (windowIsInvalid()) {                    // ── 2. SHRINK
        window.remove(a[left]);
        ++left;
    }

    best = max(best, right - left + 1);            // ── 3. RECORD
}
```

```
   WHICH VARIANT?

   "LONGEST subarray with property P"
     → shrink while INVALID, record after the while loop
   
   "SHORTEST subarray with property P"
     → shrink while VALID, record INSIDE the while loop
   
   "COUNT subarrays with property P"
     → for each right, (right - left + 1) subarrays end at right
   
   "at most K" vs "exactly K"
     → exactly(K) = atMost(K) - atMost(K-1)   ⭐ very useful reduction
```

⚠️ **Sliding window only works when the property is monotonic** — growing the window can only push it in one direction. That's why "subarray sum = k" with **negative** numbers needs prefix sums instead.

---

## A. Two Pointers — Opposite Ends

### 1. Two Sum II (sorted input) 🟡
```cpp
vector<int> twoSum(vector<int>& a, int t) {
    int l = 0, r = a.size() - 1;
    while (l < r) {
        int s = a[l] + a[r];
        if (s == t) return {l + 1, r + 1};
        if (s < t) ++l; else --r;
    }
    return {};
}
```
**Complexity:** O(n) / O(1) — better than the hash map version for sorted input.

---

### 2. 3Sum 🟡
```cpp
vector<vector<int>> threeSum(vector<int>& a) {
    sort(a.begin(), a.end());
    int n = a.size();
    vector<vector<int>> out;
    for (int i = 0; i < n - 2; ++i) {
        if (a[i] > 0) break;                       // ⭐ no triplet can sum to 0
        if (i > 0 && a[i] == a[i-1]) continue;     // skip duplicate anchors

        int l = i + 1, r = n - 1;
        while (l < r) {
            int s = a[i] + a[l] + a[r];
            if (s < 0) ++l;
            else if (s > 0) --r;
            else {
                out.push_back({a[i], a[l], a[r]});
                while (l < r && a[l] == a[l+1]) ++l;   // skip duplicates
                while (l < r && a[r] == a[r-1]) --r;
                ++l; --r;
            }
        }
    }
    return out;
}
```
**Complexity:** O(n²) / O(1).
**Key insight:** Sort, fix one element, two-pointer the rest. The three duplicate-skipping steps are what most people get wrong.

---

### 3. 3Sum Closest 🟡
```cpp
int threeSumClosest(vector<int>& a, int t) {
    sort(a.begin(), a.end());
    int n = a.size(), best = a[0] + a[1] + a[2];
    for (int i = 0; i < n - 2; ++i) {
        int l = i + 1, r = n - 1;
        while (l < r) {
            int s = a[i] + a[l] + a[r];
            if (abs(s - t) < abs(best - t)) best = s;
            if (s == t) return t;
            if (s < t) ++l; else --r;
        }
    }
    return best;
}
```

---

### 4. 3Sum Smaller 🟡
```cpp
int threeSumSmaller(vector<int>& a, int t) {
    sort(a.begin(), a.end());
    int n = a.size(), cnt = 0;
    for (int i = 0; i < n - 2; ++i) {
        int l = i + 1, r = n - 1;
        while (l < r) {
            if (a[i] + a[l] + a[r] < t) { cnt += r - l; ++l; }   // ⭐ all r' in (l,r] work
            else --r;
        }
    }
    return cnt;
}
```
**Key insight:** If `a[i]+a[l]+a[r] < t`, then every index between `l+1` and `r` also works with `l` — that's `r - l` triplets counted at once.

---

### 5. 4Sum 🟡
```cpp
vector<vector<int>> fourSum(vector<int>& a, int t) {
    sort(a.begin(), a.end());
    int n = a.size();
    vector<vector<int>> out;
    for (int i = 0; i < n - 3; ++i) {
        if (i > 0 && a[i] == a[i-1]) continue;
        for (int j = i + 1; j < n - 2; ++j) {
            if (j > i + 1 && a[j] == a[j-1]) continue;
            int l = j + 1, r = n - 1;
            while (l < r) {
                long long s = (long long)a[i] + a[j] + a[l] + a[r];   // ⭐ overflow
                if (s < t) ++l;
                else if (s > t) --r;
                else {
                    out.push_back({a[i], a[j], a[l], a[r]});
                    while (l < r && a[l] == a[l+1]) ++l;
                    while (l < r && a[r] == a[r-1]) --r;
                    ++l; --r;
                }
            }
        }
    }
    return out;
}
```
**Complexity:** O(n³). Generalizes to kSum with recursion + two pointers at the base.

---

### 6. Container With Most Water 🟡
```cpp
int maxArea(vector<int>& h) {
    int l = 0, r = h.size() - 1, best = 0;
    while (l < r) {
        best = max(best, min(h[l], h[r]) * (r - l));
        if (h[l] < h[r]) ++l; else --r;            // ⭐ move the SHORTER side
    }
    return best;
}
```
**Key insight:** Moving the taller side can never help — the area is capped by the shorter side, and width only decreases. So moving the shorter side is the only move that could improve things. That's the exchange argument proving correctness.

---

### 7. Trapping Rain Water 🔴
> Given an elevation map, compute how much water it traps after raining.

#### 💬 Think of it like this
Forget the whole array for a moment and think about **one single position**. How much water sits on top of it?

Water at position `i` is bounded by the tallest wall to its left and the tallest wall to its right. Whichever of those is *shorter* determines the water level — water spills over the lower side. So:

```
   water[i] = min(tallest on left, tallest on right) − height[i]
```

That's the whole problem. The only question is how to know both maxima efficiently.

The naive way computes both for every position: O(n²). Precomputing two arrays makes it O(n) time but O(n) space.

**The two-pointer insight is subtler and gets you to O(1) space.** Walk in from both ends. At each step, compare the two current heights — and here's the trick: *you only need to know one of the two maxima with certainty.*

If `h[left] < h[right]`, then whatever the true right-maximum is, it's **at least** `h[right]`, which is already taller than `h[left]`. So the left side is definitely the binding constraint, and you can compute the water at `left` using only `leftMax` — without ever knowing the exact right maximum.

#### 📊 Watching it work on `[0,1,0,2,1,0,1,3]`

```
   height:  0  1  0  2  1  0  1  3
            ▁  ▃  ▁  ▆  ▃  ▁  ▃  █

   Visualized with water (~):

            ·  ·  ·  ·  ·  ·  ·  █
            ·  ·  ·  ▆  ~  ~  ~  █     ← water sits here
            ·  ▃  ~  ▆  ▃  ~  ▃  █
            ▁  ▃  ▁  ▆  ▃  ▁  ▃  █

   POSITION-BY-POSITION
   ┌───────┬────────┬─────────┬──────────┬───────────────────┐
   │ index │ height │ leftMax │ rightMax │ water = min−h     │
   ├───────┼────────┼─────────┼──────────┼───────────────────┤
   │   0   │   0    │    0    │    3     │ min(0,3)−0 = 0    │
   │   1   │   1    │    1    │    3     │ min(1,3)−1 = 0    │
   │   2   │   0    │    1    │    3     │ min(1,3)−0 = ⭐1   │
   │   3   │   2    │    2    │    3     │ min(2,3)−2 = 0    │
   │   4   │   1    │    2    │    3     │ min(2,3)−1 = ⭐1   │
   │   5   │   0    │    2    │    3     │ min(2,3)−0 = ⭐2   │
   │   6   │   1    │    2    │    3     │ min(2,3)−1 = ⭐1   │
   │   7   │   3    │    3    │    3     │ min(3,3)−3 = 0    │
   └───────┴────────┴─────────┴──────────┴───────────────────┘
                                            TOTAL = 5
```

#### Why the pointer comparison is safe

```
   At any moment we know leftMax (everything we've passed on
   the left) and rightMax (everything passed on the right).

   ┌─────────────────────────────────────────────────────────────┐
   │ IF h[left] < h[right]:                                      │
   │                                                             │
   │   The true right maximum is ≥ h[right] > h[left] ≥ leftMax  │
   │                                       ▲                     │
   │   ⭐ So min(leftMax, trueRightMax) = leftMax, GUARANTEED —   │
   │     regardless of what the exact right maximum turns out    │
   │     to be. We can safely compute water at `left` now.       │
   │                                                             │
   │ Symmetrically for the other side.                           │
   └─────────────────────────────────────────────────────────────┘

   ⭐ THIS IS THE KEY INSIGHT: you never need both exact maxima
     at the same time — only the one that's provably smaller.
```

```cpp
int trap(vector<int>& h) {
    int l = 0, r = h.size() - 1, lmax = 0, rmax = 0, water = 0;
    while (l < r) {
        if (h[l] < h[r]) {
            lmax = max(lmax, h[l]);
            water += lmax - h[l];                  // ⭐ lmax is the true bound here
            ++l;
        } else {
            rmax = max(rmax, h[r]);
            water += rmax - h[r];
            --r;
        }
    }
    return water;
}
```
**Complexity:** O(n) / O(1).
**Key insight:** Water at position `i` is `min(maxLeft, maxRight) - h[i]`. When `h[l] < h[r]`, we know some bar on the right is at least `h[r] > h[l]`, so `lmax` is definitively the binding constraint — no need to know `rmax` exactly.

---

### 8. Valid Palindrome 🟢
```cpp
bool isPalindrome(string s) {
    int i = 0, j = s.size() - 1;
    while (i < j) {
        while (i < j && !isalnum(s[i])) ++i;
        while (i < j && !isalnum(s[j])) --j;
        if (tolower(s[i++]) != tolower(s[j--])) return false;
    }
    return true;
}
```

---

### 9. Reverse String / Reverse Vowels 🟢
```cpp
string reverseVowels(string s) {
    auto isV = [](char c) { return string("aeiouAEIOU").find(c) != string::npos; };
    int i = 0, j = s.size() - 1;
    while (i < j) {
        while (i < j && !isV(s[i])) ++i;
        while (i < j && !isV(s[j])) --j;
        swap(s[i++], s[j--]);
    }
    return s;
}
```

---

### 10. Squares of a Sorted Array 🟢
```cpp
vector<int> sortedSquares(vector<int>& a) {
    int n = a.size(), l = 0, r = n - 1;
    vector<int> out(n);
    for (int k = n - 1; k >= 0; --k) {             // ⭐ fill from the BACK
        int ls = a[l] * a[l], rs = a[r] * a[r];
        if (ls > rs) { out[k] = ls; ++l; } else { out[k] = rs; --r; }
    }
    return out;
}
```
**Key insight:** The largest square is at one end or the other. Filling backwards makes it a single merge pass, O(n) instead of O(n log n).

---

### 11. Boats to Save People 🟡
```cpp
int numRescueBoats(vector<int>& p, int limit) {
    sort(p.begin(), p.end());
    int l = 0, r = p.size() - 1, boats = 0;
    while (l <= r) {
        if (p[l] + p[r] <= limit) ++l;             // lightest pairs with heaviest
        --r;
        ++boats;
    }
    return boats;
}
```
**Key insight:** Greedy — the heaviest person must go, so pair them with the lightest who fits.

---

### 12. Partition Labels 🟡
```cpp
vector<int> partitionLabels(string s) {
    int last[26] = {};
    for (int i = 0; i < (int)s.size(); ++i) last[s[i] - 'a'] = i;

    vector<int> out;
    int start = 0, end = 0;
    for (int i = 0; i < (int)s.size(); ++i) {
        end = max(end, last[s[i] - 'a']);          // extend to cover this char
        if (i == end) { out.push_back(end - start + 1); start = i + 1; }
    }
    return out;
}
```

---

## B. Two Pointers — Same Direction

### 13. Remove Duplicates from Sorted Array 🟢
```cpp
int removeDuplicates(vector<int>& a) {
    int k = 0;
    for (int x : a) if (k == 0 || x != a[k-1]) a[k++] = x;
    return k;
}
```

---

### 14. Move Zeroes 🟢
```cpp
void moveZeroes(vector<int>& a) {
    int k = 0;
    for (int i = 0; i < (int)a.size(); ++i) if (a[i]) swap(a[k++], a[i]);
}
```

---

### 15. Sort Colors 🟡
```cpp
void sortColors(vector<int>& a) {
    int lo = 0, mid = 0, hi = a.size() - 1;
    while (mid <= hi) {
        if (a[mid] == 0) swap(a[lo++], a[mid++]);
        else if (a[mid] == 2) swap(a[mid], a[hi--]);
        else ++mid;
    }
}
```

---

### 16. Backspace String Compare 🟢
```cpp
bool backspaceCompare(string s, string t) {
    int i = s.size() - 1, j = t.size() - 1;
    while (true) {
        int skip = 0;
        while (i >= 0 && (s[i] == '#' || skip)) { skip += s[i] == '#' ? 1 : -1; --i; }
        skip = 0;
        while (j >= 0 && (t[j] == '#' || skip)) { skip += t[j] == '#' ? 1 : -1; --j; }

        if (i < 0 || j < 0) return i < 0 && j < 0;
        if (s[i--] != t[j--]) return false;
    }
}
```
**Key insight:** Scan from the **right** — backspaces affect what came before, so right-to-left resolves them in one pass with O(1) space.

---

### 17. Is Subsequence 🟢
```cpp
bool isSubsequence(string s, string t) {
    int i = 0;
    for (char c : t) if (i < (int)s.size() && s[i] == c) ++i;
    return i == (int)s.size();
}
```
🎤 **Follow-up (many queries):** precompute `next[i][c]` = the next occurrence of character `c` at or after index `i` in `t`, giving O(|s|) per query.

---

### 18. Merge Sorted Array 🟢
```cpp
void merge(vector<int>& a, int m, vector<int>& b, int n) {
    int i = m - 1, j = n - 1, k = m + n - 1;
    while (j >= 0) a[k--] = (i >= 0 && a[i] > b[j]) ? a[i--] : b[j--];
}
```

---

### 19. Interval List Intersections 🟡
```cpp
vector<vector<int>> intervalIntersection(vector<vector<int>>& A, vector<vector<int>>& B) {
    vector<vector<int>> out;
    int i = 0, j = 0;
    while (i < (int)A.size() && j < (int)B.size()) {
        int lo = max(A[i][0], B[j][0]);
        int hi = min(A[i][1], B[j][1]);
        if (lo <= hi) out.push_back({lo, hi});
        if (A[i][1] < B[j][1]) ++i; else ++j;      // advance the one that ends first
    }
    return out;
}
```

---

## C. Sliding Window — Variable Size

### 20. Longest Substring Without Repeating Characters 🟡
```cpp
int lengthOfLongestSubstring(string s) {
    vector<int> last(128, -1);
    int left = 0, best = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        left = max(left, last[s[r]] + 1);
        last[s[r]] = r;
        best = max(best, r - left + 1);
    }
    return best;
}
```

---

### 21. Longest Repeating Character Replacement 🟡
> Change at most `k` characters to make the longest uniform substring.

```cpp
int characterReplacement(string s, int k) {
    int cnt[26] = {}, left = 0, maxCount = 0, best = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        maxCount = max(maxCount, ++cnt[s[r] - 'A']);
        // window is invalid if replacements needed > k
        while (r - left + 1 - maxCount > k) --cnt[s[left++] - 'A'];
        best = max(best, r - left + 1);
    }
    return best;
}
```
**Key insight:** `maxCount` is never decremented, which looks like a bug but isn't. The window only ever grows when a *better* `maxCount` appears, so a stale value can't produce a larger answer than the true one. The result is still correct and the code stays O(n).

---

### 22. Minimum Window Substring 🔴
> Find the shortest substring of `s` containing every character of `t` (including duplicates).

#### 💬 Think of it like this
Picture a stretchy window over the string with a left and right edge.

**Expand right** until the window contains everything you need. **Then shrink from the left** as far as you can while still being valid, recording the size each time. When it becomes invalid, go back to expanding.

That's the "shortest" variant of the sliding window template — you record *inside* the shrink loop, not after it, because you want the smallest valid window at each stopping point.

The clever part is checking validity in O(1) rather than comparing two frequency maps every step. Keep a counter called `missing` — how many required characters you still need. Increment and decrement it only when crossing the boundary between "have enough" and "don't have enough."

#### 📊 Tracing `s = "ADOBECODEBANC"`, `t = "ABC"`

```
   need = {A:1, B:1, C:1}, missing = 3

   ┌──────────────────────────────────────────────────────────────┐
   │ EXPAND until valid                                           │
   │                                                              │
   │  A D O B E C O D E B A N C                                   │
   │  └─────────┘                                                 │
   │  A(missing 2) D O B(missing 1) E C(missing 0) ⭐ VALID        │
   │  window "ADOBEC", length 6  → best so far                    │
   ├──────────────────────────────────────────────────────────────┤
   │ SHRINK from the left while still valid                       │
   │                                                              │
   │  A D O B E C O D E B A N C                                   │
   │    └───────┘                                                 │
   │  drop A → missing becomes 1 → ⚠️ INVALID, stop shrinking      │
   │  best is still "ADOBEC" (6)                                  │
   ├──────────────────────────────────────────────────────────────┤
   │ EXPAND again until valid                                     │
   │                                                              │
   │  A D O B E C O D E B A N C                                   │
   │    └───────────────────┘                                     │
   │  ... reach the second A → valid again                        │
   │  window "DOBECODEBA", length 10 → worse, don't record        │
   ├──────────────────────────────────────────────────────────────┤
   │ SHRINK                                                       │
   │  A D O B E C O D E B A N C                                   │
   │            └───────────┘                                     │
   │  shrink to "CODEBA", length 6 → tie                          │
   │  shrink further → drops C → invalid                          │
   ├──────────────────────────────────────────────────────────────┤
   │ EXPAND to the final C                                        │
   │  A D O B E C O D E B A N C                                   │
   │                    └─────┘                                   │
   │  window "BANC", length 4  ⭐ NEW BEST                         │
   └──────────────────────────────────────────────────────────────┘

   ANSWER = "BANC"
```

#### The `missing` counter trick

```
   need[c] starts as the required count for each character in t,
   and 0 for everything else.

   ⭐ ON EXPAND:  need[c]--  ... and IF it was still POSITIVE
                  before decrementing, that character was one we
                  genuinely still needed → missing--

     This means surplus characters push need[c] NEGATIVE, and
     correctly do NOT decrease `missing`.

   ⭐ ON SHRINK:  need[c]++  ... and IF it becomes POSITIVE,
                  we've just given up a character we needed
                  → missing++

     Surplus characters going from -2 to -1 stay non-positive,
     so `missing` is untouched and the window stays valid.

   ⭐ Result: validity is a single integer comparison
     (missing == 0) instead of comparing 128 counters.
```

```cpp
string minWindow(string s, string t) {
    if (s.size() < t.size()) return "";
    int need[128] = {};
    for (char c : t) need[c]++;

    int missing = t.size(), left = 0, bl = 0, blen = INT_MAX;
    for (int r = 0; r < (int)s.size(); ++r) {
        if (need[s[r]]-- > 0) --missing;           // ⭐ only counts chars we need

        while (missing == 0) {                     // valid → try to shrink
            if (r - left + 1 < blen) { blen = r - left + 1; bl = left; }
            if (++need[s[left++]] > 0) ++missing;
        }
    }
    return blen == INT_MAX ? "" : s.substr(bl, blen);
}
```
**Complexity:** O(|s| + |t|).
**Key insight:** `need[c]` can go negative (surplus characters). `missing` only changes when crossing zero, so the check is O(1) rather than comparing full frequency maps.

---

### 23. Permutation in String 🟡
```cpp
bool checkInclusion(string p, string s) {
    if (p.size() > s.size()) return false;
    int need[26] = {}, win[26] = {};
    for (char c : p) need[c - 'a']++;

    for (int i = 0; i < (int)s.size(); ++i) {
        win[s[i] - 'a']++;
        if (i >= (int)p.size()) win[s[i - p.size()] - 'a']--;
        if (i >= (int)p.size() - 1 && equal(need, need + 26, win)) return true;
    }
    return false;
}
```

---

### 24. Find All Anagrams in a String 🟡
```cpp
vector<int> findAnagrams(string s, string p) {
    vector<int> out;
    if (s.size() < p.size()) return out;
    int need[26] = {}, win[26] = {};
    for (char c : p) need[c - 'a']++;
    for (int i = 0; i < (int)s.size(); ++i) {
        win[s[i] - 'a']++;
        if (i >= (int)p.size()) win[s[i - p.size()] - 'a']--;
        if (i >= (int)p.size() - 1 && equal(need, need + 26, win))
            out.push_back(i - p.size() + 1);
    }
    return out;
}
```

---

### 25. Longest Substring with At Most K Distinct Characters 🟡
```cpp
int lengthOfLongestSubstringKDistinct(string s, int k) {
    unordered_map<char,int> cnt;
    int left = 0, best = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        cnt[s[r]]++;
        while ((int)cnt.size() > k) {
            if (--cnt[s[left]] == 0) cnt.erase(s[left]);   // ⭐ erase at zero
            ++left;
        }
        best = max(best, r - left + 1);
    }
    return best;
}
```
⚠️ You must **erase** the key at zero, or `cnt.size()` overcounts distinct characters.

---

### 26. Fruit Into Baskets (at most 2 distinct) 🟡
```cpp
int totalFruit(vector<int>& f) {
    unordered_map<int,int> cnt;
    int left = 0, best = 0;
    for (int r = 0; r < (int)f.size(); ++r) {
        cnt[f[r]]++;
        while ((int)cnt.size() > 2) if (--cnt[f[left++]] == 0) cnt.erase(f[left-1]);
        best = max(best, r - left + 1);
    }
    return best;
}
```

---

### 27. Minimum Size Subarray Sum 🟡
> Shortest subarray with sum ≥ target. All positive.

```cpp
int minSubArrayLen(int target, vector<int>& a) {
    int left = 0, best = INT_MAX;
    long long sum = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        sum += a[r];
        while (sum >= target) {                    // ⭐ shrink while VALID
            best = min(best, r - left + 1);
            sum -= a[left++];
        }
    }
    return best == INT_MAX ? 0 : best;
}
```
**Key insight:** This is the "shortest" variant — record **inside** the shrink loop, not after it.

---

### 28. Max Consecutive Ones III 🟡
```cpp
int longestOnes(vector<int>& a, int k) {
    int left = 0, zeros = 0, best = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        if (a[r] == 0) ++zeros;
        while (zeros > k) if (a[left++] == 0) --zeros;
        best = max(best, r - left + 1);
    }
    return best;
}
```

---

### 29. Subarrays with K Different Integers 🔴
```cpp
int atMost(vector<int>& a, int k) {
    unordered_map<int,int> cnt;
    int left = 0, res = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        if (cnt[a[r]]++ == 0) --k;
        while (k < 0) if (--cnt[a[left++]] == 0) ++k;
        res += r - left + 1;                       // ⭐ all subarrays ending at r
    }
    return res;
}
int subarraysWithKDistinct(vector<int>& a, int k) {
    return atMost(a, k) - atMost(a, k - 1);        // ⭐⭐ the key reduction
}
```
**Key insight:** `exactly(K) = atMost(K) - atMost(K-1)`. "Exactly" isn't directly slideable because the window isn't monotonic, but "at most" is. Memorize this reduction — it appears constantly.

---

### 30. Count Number of Nice Subarrays 🟡
```cpp
int atMost(vector<int>& a, int k) {
    int left = 0, odd = 0, res = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        odd += a[r] & 1;
        while (odd > k) odd -= a[left++] & 1;
        res += r - left + 1;
    }
    return res;
}
int numberOfSubarrays(vector<int>& a, int k) { return atMost(a, k) - atMost(a, k - 1); }
```

---

### 31. Subarray Product Less Than K 🟡
```cpp
int numSubarrayProductLessThanK(vector<int>& a, int k) {
    if (k <= 1) return 0;
    long long prod = 1;
    int left = 0, res = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        prod *= a[r];
        while (prod >= k) prod /= a[left++];
        res += r - left + 1;
    }
    return res;
}
```

---

### 32. Longest Subarray of 1's After Deleting One Element 🟡
```cpp
int longestSubarray(vector<int>& a) {
    int left = 0, zeros = 0, best = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        if (a[r] == 0) ++zeros;
        while (zeros > 1) if (a[left++] == 0) --zeros;
        best = max(best, r - left);                // ⭐ -1 for the deleted element
    }
    return best;
}
```

---

### 33. Minimum Operations to Reduce X to Zero 🟡
```cpp
int minOperations(vector<int>& a, int x) {
    long long total = accumulate(a.begin(), a.end(), 0LL);
    long long target = total - x;                  // ⭐ invert the problem
    if (target < 0) return -1;
    if (target == 0) return a.size();

    long long sum = 0;
    int left = 0, best = -1;
    for (int r = 0; r < (int)a.size(); ++r) {
        sum += a[r];
        while (sum > target) sum -= a[left++];
        if (sum == target) best = max(best, r - left + 1);
    }
    return best == -1 ? -1 : a.size() - best;
}
```
**Key insight:** "Remove from both ends summing to x" is equivalent to "find the **longest middle subarray** summing to `total - x`." Inverting the problem turns it into a standard window.

---

### 34. Maximum Points from Cards 🟡
```cpp
int maxScore(vector<int>& a, int k) {
    int n = a.size();
    int windowSize = n - k;                        // the part we DON'T take
    long long total = accumulate(a.begin(), a.end(), 0LL);
    if (windowSize == 0) return total;

    long long sum = 0, minWindow = LLONG_MAX;
    for (int i = 0; i < n; ++i) {
        sum += a[i];
        if (i >= windowSize) sum -= a[i - windowSize];
        if (i >= windowSize - 1) minWindow = min(minWindow, sum);
    }
    return total - minWindow;
}
```
**Key insight:** Same inversion — maximize the ends by minimizing the fixed-size middle window.

---

## D. Sliding Window — Fixed Size

### 35. Maximum Average Subarray I 🟢
```cpp
double findMaxAverage(vector<int>& a, int k) {
    long long sum = 0;
    for (int i = 0; i < k; ++i) sum += a[i];
    long long best = sum;
    for (int i = k; i < (int)a.size(); ++i) {
        sum += a[i] - a[i - k];
        best = max(best, sum);
    }
    return (double)best / k;
}
```

---

### 36. Sliding Window Maximum 🔴
```cpp
vector<int> maxSlidingWindow(vector<int>& a, int k) {
    deque<int> dq;                                 // indices, values DECREASING
    vector<int> out;
    for (int i = 0; i < (int)a.size(); ++i) {
        if (!dq.empty() && dq.front() <= i - k) dq.pop_front();       // out of window
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();   // ⭐ dominated
        dq.push_back(i);
        if (i >= k - 1) out.push_back(a[dq.front()]);
    }
    return out;
}
```
**Complexity:** O(n) — each index is pushed and popped once.
**Key insight:** A **monotonic deque**. If `a[j] <= a[i]` and `j < i`, then `a[j]` can never be a maximum again — `a[i]` is both larger and stays in the window longer. So it's discarded permanently.

---

### 37. Sliding Window Median 🔴
```cpp
vector<double> medianSlidingWindow(vector<int>& a, int k) {
    multiset<int> win(a.begin(), a.begin() + k);
    auto mid = next(win.begin(), k / 2);
    vector<double> out;

    for (int i = k; ; ++i) {
        out.push_back(k % 2 ? (double)*mid : ((double)*mid + *prev(mid)) / 2.0);
        if (i == (int)a.size()) break;

        win.insert(a[i]);
        if (a[i] < *mid) --mid;                    // ⭐ maintain the mid iterator
        if (a[i - k] <= *mid) ++mid;
        win.erase(win.lower_bound(a[i - k]));      // ⭐ erase ONE occurrence
    }
    return out;
}
```
⚠️ `win.erase(value)` removes **all** matching elements. Use `erase(iterator)` to remove exactly one.

---

### 38. Repeated DNA Sequences 🟡
```cpp
vector<string> findRepeatedDnaSequences(string s) {
    if (s.size() < 10) return {};
    unordered_map<int,int> seen;                   // rolling hash -> count
    unordered_map<char,int> code{{'A',0},{'C',1},{'G',2},{'T',3}};

    vector<string> out;
    int h = 0, mask = (1 << 20) - 1;               // 10 chars × 2 bits
    for (int i = 0; i < (int)s.size(); ++i) {
        h = ((h << 2) | code[s[i]]) & mask;        // ⭐ rolling: shift in, mask out
        if (i >= 9 && ++seen[h] == 2) out.push_back(s.substr(i - 9, 10));
    }
    return out;
}
```
**Key insight:** Two bits per base packs a 10-character window into a 20-bit integer — the rolling update is O(1) instead of O(10) hashing.

---

### 39. Minimum Window Subsequence 🔴
```cpp
string minWindow(string s, string t) {
    int n = s.size(), m = t.size(), bl = -1, blen = INT_MAX;
    int i = 0;
    while (i < n) {
        int j = 0, k = i;
        while (k < n) {                            // forward: match t greedily
            if (s[k] == t[j]) if (++j == m) break;
            ++k;
        }
        if (k == n) break;                         // no more matches

        int end = k;
        --j;
        while (j >= 0) {                           // ⭐ backward: shrink to minimal
            if (s[k] == t[j]) --j;
            --k;
        }
        ++k;
        if (end - k + 1 < blen) { blen = end - k + 1; bl = k; }
        i = k + 1;                                 // restart just after this window
    }
    return bl == -1 ? "" : s.substr(bl, blen);
}
```
**Complexity:** O(n·m) worst case.
**Key insight:** Unlike minimum window *substring*, order matters here, so a frequency window doesn't apply. The forward-then-backward two-phase scan finds the minimal window ending at each match.

---

### 40. Longest Substring with At Least K Repeating Characters 🟡
```cpp
int longestSubstring(string s, int k) {
    int best = 0;
    // Try each possible count of distinct characters — makes the window monotonic
    for (int unique = 1; unique <= 26; ++unique) {
        int cnt[26] = {}, left = 0, distinct = 0, atLeastK = 0;
        for (int r = 0; r < (int)s.size(); ++r) {
            if (cnt[s[r]-'a']++ == 0) ++distinct;
            if (cnt[s[r]-'a'] == k) ++atLeastK;

            while (distinct > unique) {
                if (cnt[s[left]-'a'] == k) --atLeastK;
                if (--cnt[s[left]-'a'] == 0) --distinct;
                ++left;
            }
            if (distinct == unique && atLeastK == unique)
                best = max(best, r - left + 1);
        }
    }
    return best;
}
```
**Key insight:** Plain sliding window fails because the property isn't monotonic. Fixing the number of distinct characters restores monotonicity. The divide-and-conquer alternative — split on any character appearing fewer than k times — is also worth knowing.

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║          TWO POINTERS & SLIDING WINDOW — PATTERN RECALL           ║
╠═══════════════════════════════════════════════════════════════════╣
║ TWO POINTERS (opposite ends) — requires SORTED input              ║
║   pair/triplet sum · container water · trapping rain              ║
║   ⭐ move the pointer that CAN'T improve if left alone             ║
║                                                                   ║
║ SLIDING WINDOW template: EXPAND → while-invalid SHRINK → RECORD   ║
║   LONGEST  → record AFTER the while loop                          ║
║   SHORTEST → record INSIDE the while loop                         ║
║   COUNT    → res += (right - left + 1) per step                   ║
║   EXACTLY K → atMost(K) − atMost(K−1)   ⭐⭐ memorize               ║
║                                                                   ║
║ ⚠️ WINDOW FAILS WITH NEGATIVES → use prefix sum + hashmap          ║
║ ⚠️ Non-monotonic property → fix a parameter (e.g. distinct count)  ║
║                              to restore monotonicity              ║
║                                                                   ║
║ MONOTONIC DEQUE → sliding window max/min in O(n)                  ║
║ MULTISET → sliding window median (erase by ITERATOR not value)    ║
║ INVERSION TRICK → "remove from ends" = "keep the middle window"   ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Linked Lists →](04-linked-lists.md)
