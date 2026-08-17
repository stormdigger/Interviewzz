# ⛰️ Heaps & Intervals — 30 Problems

> Two patterns that pair naturally. Heaps answer "what's the current extreme?" in O(log n). Intervals are almost always "sort, then sweep."

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 Heap Fundamentals

```cpp
priority_queue<int> maxHeap;                                     // default: MAX
priority_queue<int, vector<int>, greater<int>> minHeap;          // MIN
priority_queue<pair<int,int>> pq;                                // sorts by .first

// Custom comparator — ⚠️ the logic is INVERTED from a sort comparator
auto cmp = [](const Task& a, const Task& b) { return a.pri > b.pri; };  // → MIN-heap
priority_queue<Task, vector<Task>, decltype(cmp)> pq(cmp);

pq.push(x);  pq.top();  pq.pop();  pq.size();  pq.empty();
// ⚠️ No way to remove an arbitrary element, and no way to update a key.
//    Use "lazy deletion" (push a new entry, skip stale ones on pop).
```

```
   COMPLEXITY
     build from n elements  O(n)      ← heapify, not n × log n
     push / pop             O(log n)
     top                    O(1)
     find arbitrary         O(n)

   ⭐ THE COUNTERINTUITIVE RULE
     K LARGEST  → use a MIN-heap of size k (pop the smallest)
     K SMALLEST → use a MAX-heap of size k (pop the largest)
     Complexity O(n log k), beating O(n log n) sorting when k << n.
```

### When to use a heap

```
   "top k" / "k-th largest"            → bounded heap of size k
   "merge k sorted things"             → heap of the k heads
   "running/streaming median"          → TWO heaps
   "schedule by priority"              → max-heap
   "always process the cheapest next"  → min-heap (Dijkstra, Prim, Huffman)
   "minimum resources over intervals"  → min-heap of end times
```

---

## A. Top-K

### 1. Kth Largest Element in an Array 🟡
```cpp
// Approach 1: min-heap of size k — O(n log k)
int findKthLargest(vector<int>& a, int k) {
    priority_queue<int, vector<int>, greater<int>> pq;
    for (int x : a) {
        pq.push(x);
        if ((int)pq.size() > k) pq.pop();          // drop the smallest
    }
    return pq.top();
}

// Approach 2: quickselect — O(n) average, O(1) space  ⭐ mention this
int quickselect(vector<int>& a, int k) {
    nth_element(a.begin(), a.begin() + (a.size() - k), a.end());
    return a[a.size() - k];
}
```
🎤 Discuss the trade: heap is O(n log k) and works on a **stream**; quickselect is O(n) average but O(n²) worst case, needs the whole array in memory, and mutates it.

---

### 2. Top K Frequent Elements 🟡
```cpp
vector<int> topKFrequent(vector<int>& a, int k) {
    unordered_map<int,int> cnt;
    for (int x : a) cnt[x]++;

    // bucket sort — O(n), beats the heap here
    vector<vector<int>> bucket(a.size() + 1);
    for (auto& [v, c] : cnt) bucket[c].push_back(v);

    vector<int> out;
    for (int f = a.size(); f >= 1 && (int)out.size() < k; --f)
        for (int v : bucket[f]) { out.push_back(v); if ((int)out.size() == k) break; }
    return out;
}
```

---

### 3. K Closest Points to Origin 🟡
```cpp
vector<vector<int>> kClosest(vector<vector<int>>& pts, int k) {
    priority_queue<pair<int, int>> pq;             // max-heap of {dist², index}
    for (int i = 0; i < (int)pts.size(); ++i) {
        int d = pts[i][0]*pts[i][0] + pts[i][1]*pts[i][1];   // ⭐ no sqrt needed
        pq.push({d, i});
        if ((int)pq.size() > k) pq.pop();
    }
    vector<vector<int>> out;
    while (!pq.empty()) { out.push_back(pts[pq.top().second]); pq.pop(); }
    return out;
}
```
**Key insight:** Compare squared distances — `sqrt` is monotonic, so it changes nothing and costs time.

---

### 4. Kth Largest Element in a Stream 🟢
```cpp
class KthLargest {
    priority_queue<int, vector<int>, greater<int>> pq;
    int k;
public:
    KthLargest(int k, vector<int>& a) : k(k) {
        for (int x : a) add(x);
    }
    int add(int x) {
        pq.push(x);
        if ((int)pq.size() > k) pq.pop();
        return pq.top();
    }
};
```
**Key insight:** This is where the heap genuinely beats quickselect — it maintains the answer incrementally on an unbounded stream.

---

### 5. Sort Characters By Frequency 🟡
```cpp
string frequencySort(string s) {
    unordered_map<char,int> cnt;
    for (char c : s) cnt[c]++;
    priority_queue<pair<int,char>> pq;             // max-heap by count
    for (auto& [c, f] : cnt) pq.push({f, c});
    string out;
    while (!pq.empty()) { auto [f, c] = pq.top(); pq.pop(); out += string(f, c); }
    return out;
}
```

---

### 6. Kth Smallest Element in a Sorted Matrix 🟡
```cpp
int kthSmallest(vector<vector<int>>& m, int k) {
    int n = m.size();
    // Binary search on the VALUE — O(n log(max-min))  ⭐ better than the heap
    int lo = m[0][0], hi = m[n-1][n-1];
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        int cnt = 0, j = n - 1;
        for (int i = 0; i < n; ++i) {              // count elements <= mid, O(n)
            while (j >= 0 && m[i][j] > mid) --j;
            cnt += j + 1;
        }
        if (cnt < k) lo = mid + 1; else hi = mid;
    }
    return lo;
}
```
**Key insight:** Binary search on the answer value, counting elements ≤ mid by walking the staircase. Avoids the O(k log n) heap approach entirely.

---

### 7. Find K Pairs with Smallest Sums 🟡
```cpp
vector<vector<int>> kSmallestPairs(vector<int>& a, vector<int>& b, int k) {
    auto cmp = [&](pair<int,int> x, pair<int,int> y) {
        return a[x.first] + b[x.second] > a[y.first] + b[y.second];
    };
    priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);
    for (int i = 0; i < min((int)a.size(), k); ++i) pq.push({i, 0});   // ⭐ seed column 0

    vector<vector<int>> out;
    while (k-- && !pq.empty()) {
        auto [i, j] = pq.top(); pq.pop();
        out.push_back({a[i], b[j]});
        if (j + 1 < (int)b.size()) pq.push({i, j + 1});   // only advance in b
    }
    return out;
}
```
**Key insight:** Think of it as a sorted matrix — the heap holds one frontier cell per row.

---

## B. Two Heaps

### 8. Find Median from Data Stream 🔴
> Numbers arrive one at a time. Report the median at any point, efficiently.

#### 💬 Think of it like this
Sorting on every query is O(n log n) each time. Way too slow.

But notice what the median actually *is*: the boundary between the smaller half and the larger half. You don't need the numbers sorted — you only need to know **what sits at that boundary**.

So split the data into two piles:
- A **max-heap** holding the smaller half — its top is the *largest* of the small numbers
- A **min-heap** holding the larger half — its top is the *smallest* of the large numbers

Those two tops are the values sitting right at the middle. If the piles are equal, the median is their average; if one has an extra element, its top *is* the median.

#### 📊 The structure

```
        SMALLER HALF                    LARGER HALF
        max-heap (lo)                   min-heap (hi)

        ┌─────────────┐                 ┌─────────────┐
        │      3      │◀── tops ──▶     │      4      │
        │    /   \    │   are the       │    /   \    │
        │   1     2   │   middle        │   6     5   │
        └─────────────┘                 └─────────────┘
              ▲                               ▲
        largest of the                  smallest of the
        small numbers                   large numbers

   ⭐ INVARIANTS
     1. Every element in `lo` ≤ every element in `hi`
     2. sizes are equal, OR `lo` has exactly one more
```

#### The insertion trick

```
   ⭐ Rather than deciding which heap a new number belongs to
     (which needs comparisons and branching), route it through
     BOTH:

   ┌─────────────────────────────────────────────────────────────┐
   │ 1. Push the new number into `lo` (the max-heap)             │
   │ 2. Immediately move lo's TOP into `hi`                      │
   │    ⭐ This guarantees invariant 1 automatically — whatever   │
   │      the largest of the small half now is, it moves up.     │
   │ 3. If `hi` is now bigger than `lo`, move hi's top back      │
   │    ⭐ This restores invariant 2.                             │
   └─────────────────────────────────────────────────────────────┘

   No comparisons, no branching on value. Three lines.
```

#### Watching `[5, 15, 1, 3]` arrive

```
   ┌───────────────────────────────────────────────────────────┐
   │ add 5:   lo=[5]        hi=[]        median = 5            │
   ├───────────────────────────────────────────────────────────┤
   │ add 15:  push to lo → lo=[15,5]                           │
   │          move top up → lo=[5]  hi=[15]                    │
   │          sizes equal → median = (5+15)/2 = 10             │
   ├───────────────────────────────────────────────────────────┤
   │ add 1:   push to lo → lo=[5,1]                            │
   │          move top up → lo=[1]  hi=[5,15]                  │
   │          hi bigger → move back → lo=[5,1]  hi=[15]        │
   │          ⭐ lo has one more → median = lo.top = 5          │
   ├───────────────────────────────────────────────────────────┤
   │ add 3:   push to lo → lo=[5,1,3]                          │
   │          move top up → lo=[3,1]  hi=[5,15]                │
   │          sizes equal → median = (3+5)/2 = 4               │
   └───────────────────────────────────────────────────────────┘

   Verify: sorted data is [1,3,5,15] → median = (3+5)/2 = 4 ✅
```

**Complexity:** O(log n) per insertion, O(1) per median query.

⚠️ **Where two heaps DON'T work:** the sliding-window median (§9), because heaps can't remove an arbitrary element — only the top. That's why the windowed version uses a `multiset` with a maintained middle iterator instead.

```cpp
class MedianFinder {
    priority_queue<int> lo;                                   // max-heap, lower half
    priority_queue<int, vector<int>, greater<int>> hi;        // min-heap, upper half
public:
    void addNum(int x) {
        lo.push(x);
        hi.push(lo.top()); lo.pop();               // ⭐ route through to keep order
        if (hi.size() > lo.size()) { lo.push(hi.top()); hi.pop(); }
    }
    double findMedian() {
        return lo.size() > hi.size() ? lo.top() : (lo.top() + hi.top()) / 2.0;
    }
};
```
```
   INVARIANT:  every element in `lo` ≤ every element in `hi`
               |lo| == |hi|  or  |lo| == |hi| + 1

   lo (max-heap)        hi (min-heap)
   [ 1  2  3 ]          [ 4  5  6 ]
          ▲                ▲
        top=3            top=4    → median = (3+4)/2
```
**Key insight:** Pushing into `lo` then immediately moving its top to `hi` guarantees the ordering invariant without any comparison logic.

---

### 9. Sliding Window Median 🔴
```cpp
vector<double> medianSlidingWindow(vector<int>& a, int k) {
    multiset<int> win(a.begin(), a.begin() + k);
    auto mid = next(win.begin(), k / 2);
    vector<double> out;
    for (int i = k; ; ++i) {
        out.push_back(k % 2 ? (double)*mid : ((double)*mid + *prev(mid)) / 2.0);
        if (i == (int)a.size()) break;
        win.insert(a[i]);
        if (a[i] < *mid) --mid;
        if (a[i - k] <= *mid) ++mid;
        win.erase(win.lower_bound(a[i - k]));      // ⭐ erase ONE, by iterator
    }
    return out;
}
```
**Key insight:** Two heaps can't delete arbitrary elements, so a `multiset` with a maintained middle iterator is the cleaner solution.

---

### 10. IPO (maximize capital) 🔴
```cpp
int findMaximizedCapital(int k, int w, vector<int>& profits, vector<int>& capital) {
    int n = profits.size();
    vector<pair<int,int>> proj(n);
    for (int i = 0; i < n; ++i) proj[i] = {capital[i], profits[i]};
    sort(proj.begin(), proj.end());                // by capital required

    priority_queue<int> avail;                     // max-heap of affordable profits
    int i = 0;
    while (k--) {
        while (i < n && proj[i].first <= w) avail.push(proj[i++].second);
        if (avail.empty()) break;
        w += avail.top(); avail.pop();             // ⭐ greedily take the best affordable
    }
    return w;
}
```
**Key insight:** Two structures — a sorted list gates by affordability, a heap picks the best among what's currently affordable.

---

## C. Merging & Scheduling

### 11. Merge k Sorted Lists 🔴
```cpp
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
```
**Complexity:** O(N log k).

---

### 12. Merge k Sorted Arrays / Smallest Range 🔴
```cpp
vector<int> smallestRange(vector<vector<int>>& lists) {
    using T = tuple<int,int,int>;                  // {value, listIdx, elemIdx}
    priority_queue<T, vector<T>, greater<T>> pq;
    int hi = INT_MIN;
    for (int i = 0; i < (int)lists.size(); ++i) {
        pq.push({lists[i][0], i, 0});
        hi = max(hi, lists[i][0]);
    }

    int bl = 0, br = INT_MAX;
    while (true) {
        auto [v, li, ei] = pq.top(); pq.pop();
        if (hi - v < br - bl) { bl = v; br = hi; }
        if (ei + 1 == (int)lists[li].size()) break;   // ⭐ a list is exhausted
        pq.push({lists[li][ei+1], li, ei+1});
        hi = max(hi, lists[li][ei+1]);
    }
    return {bl, br};
}
```
**Key insight:** The heap holds one element from each list. The range is `[heap min, running max]` — advancing the minimum is the only move that can shrink it.

---

### 13. Task Scheduler 🟡
```cpp
int leastInterval(vector<char>& tasks, int n) {
    int cnt[26] = {};
    for (char c : tasks) cnt[c - 'A']++;
    int maxCount = *max_element(cnt, cnt + 26);
    int numMax = count(cnt, cnt + 26, maxCount);

    // Frame the most frequent task; fill the gaps
    return max((int)tasks.size(), (maxCount - 1) * (n + 1) + numMax);
}
```
```
   With A appearing 3 times and n = 2:
   A _ _ A _ _ A
   └─(n+1)─┘└─(n+1)─┘  ← (maxCount-1) frames
                    └─ plus numMax tasks in the last slot

   max(...) with tasks.size() handles the case where there are
   enough other tasks to fill every idle slot.
```

---

### 14. Reorganize String 🟡
```cpp
string reorganizeString(string s) {
    int cnt[26] = {};
    for (char c : s) cnt[c - 'a']++;
    priority_queue<pair<int,char>> pq;
    for (int i = 0; i < 26; ++i) if (cnt[i]) pq.push({cnt[i], 'a' + i});

    string out;
    while (pq.size() >= 2) {
        auto [c1, ch1] = pq.top(); pq.pop();       // ⭐ take the TWO most frequent
        auto [c2, ch2] = pq.top(); pq.pop();
        out += ch1; out += ch2;
        if (c1 > 1) pq.push({c1 - 1, ch1});
        if (c2 > 1) pq.push({c2 - 1, ch2});
    }
    if (!pq.empty()) {
        if (pq.top().first > 1) return "";         // impossible
        out += pq.top().second;
    }
    return out;
}
```
**Key insight:** Always emit the two currently most frequent characters — they can't be adjacent to themselves, and greedily reducing the largest count keeps the problem feasible.

---

### 15. Rearrange String k Distance Apart 🔴
```cpp
string rearrangeString(string s, int k) {
    if (k <= 1) return s;
    unordered_map<char,int> cnt;
    for (char c : s) cnt[c]++;
    priority_queue<pair<int,char>> pq;
    for (auto& [c, f] : cnt) pq.push({f, c});

    string out;
    queue<pair<int,char>> cooldown;                // ⭐ can't reuse for k steps
    while (!pq.empty()) {
        auto [f, c] = pq.top(); pq.pop();
        out += c;
        cooldown.push({f - 1, c});
        if ((int)cooldown.size() >= k) {
            auto [nf, nc] = cooldown.front(); cooldown.pop();
            if (nf > 0) pq.push({nf, nc});
        }
    }
    return out.size() == s.size() ? out : "";
}
```

---

### 16. Minimum Cost to Connect Sticks 🟡
```cpp
int connectSticks(vector<int>& sticks) {
    priority_queue<int, vector<int>, greater<int>> pq(sticks.begin(), sticks.end());
    int cost = 0;
    while (pq.size() > 1) {
        int a = pq.top(); pq.pop();
        int b = pq.top(); pq.pop();
        cost += a + b;
        pq.push(a + b);
    }
    return cost;
}
```
**Key insight:** Huffman coding. Always merging the two smallest minimizes total cost, because the earlier a stick is merged, the more times its length is counted.

---

### 17. Last Stone Weight 🟢
```cpp
int lastStoneWeight(vector<int>& stones) {
    priority_queue<int> pq(stones.begin(), stones.end());
    while (pq.size() > 1) {
        int a = pq.top(); pq.pop();
        int b = pq.top(); pq.pop();
        if (a != b) pq.push(a - b);
    }
    return pq.empty() ? 0 : pq.top();
}
```

---

### 18. Ugly Number II 🟡
```cpp
int nthUglyNumber(int n) {
    vector<int> ugly(n);
    ugly[0] = 1;
    int i2 = 0, i3 = 0, i5 = 0;
    for (int i = 1; i < n; ++i) {
        int next = min({ugly[i2] * 2, ugly[i3] * 3, ugly[i5] * 5});
        ugly[i] = next;
        if (next == ugly[i2] * 2) ++i2;            // ⭐ advance ALL matching pointers
        if (next == ugly[i3] * 3) ++i3;            //    (handles duplicates like 6)
        if (next == ugly[i5] * 5) ++i5;
    }
    return ugly[n-1];
}
```
**Complexity:** O(n) with three pointers, vs O(n log n) with a heap plus a dedup set.

---

## D. Intervals — Merge & Sweep

### 19. Merge Intervals 🟡
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

### 20. Insert Interval 🟡
```cpp
vector<vector<int>> insert(vector<vector<int>>& iv, vector<int> ni) {
    vector<vector<int>> out;
    int i = 0, n = iv.size();
    while (i < n && iv[i][1] < ni[0]) out.push_back(iv[i++]);
    while (i < n && iv[i][0] <= ni[1]) {
        ni[0] = min(ni[0], iv[i][0]);
        ni[1] = max(ni[1], iv[i][1]);
        ++i;
    }
    out.push_back(ni);
    while (i < n) out.push_back(iv[i++]);
    return out;
}
```
**Complexity:** O(n) — no sort needed since the input is sorted.

---

### 21. Meeting Rooms 🟢
```cpp
bool canAttendMeetings(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end());
    for (int i = 1; i < (int)iv.size(); ++i)
        if (iv[i][0] < iv[i-1][1]) return false;
    return true;
}
```

---

### 22. Meeting Rooms II 🟡
```cpp
// Approach 1: min-heap of END times
int minMeetingRooms(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end());
    priority_queue<int, vector<int>, greater<int>> ends;
    for (auto& m : iv) {
        if (!ends.empty() && ends.top() <= m[0]) ends.pop();   // ⭐ reuse a room
        ends.push(m[1]);
    }
    return ends.size();
}

// Approach 2: sweep line — often cleaner
int minMeetingRooms2(vector<vector<int>>& iv) {
    vector<pair<int,int>> ev;
    for (auto& m : iv) { ev.push_back({m[0], 1}); ev.push_back({m[1], -1}); }
    sort(ev.begin(), ev.end());                    // -1 before +1 at the same time
    int cur = 0, best = 0;
    for (auto& [t, d] : ev) { cur += d; best = max(best, cur); }
    return best;
}
```
**Key insight:** The heap holds the end time of every currently-occupied room; its size *is* the room count. The sweep-line version generalizes better to weighted problems.

---

### 23. Non-overlapping Intervals 🟡
```cpp
int eraseOverlapIntervals(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end(), [](auto& a, auto& b){ return a[1] < b[1]; });
    int end = INT_MIN, keep = 0;
    for (auto& x : iv) if (x[0] >= end) { end = x[1]; ++keep; }
    return iv.size() - keep;
}
```
⚠️ **Sort by END, not start.** Activity selection is only optimal when you always keep the earliest-finishing interval, which leaves maximum room for the rest.

---

### 24. Minimum Number of Arrows to Burst Balloons 🟡
```cpp
int findMinArrowShots(vector<vector<int>>& pts) {
    sort(pts.begin(), pts.end(), [](auto& a, auto& b){ return a[1] < b[1]; });
    int arrows = 0;
    long long end = LLONG_MIN;
    for (auto& p : pts) if (p[0] > end) { ++arrows; end = p[1]; }
    return arrows;
}
```
**Key insight:** Same greedy as activity selection — shoot at the earliest end point, popping every balloon that overlaps it.

---

### 25. Interval List Intersections 🟡
```cpp
vector<vector<int>> intervalIntersection(vector<vector<int>>& A, vector<vector<int>>& B) {
    vector<vector<int>> out;
    int i = 0, j = 0;
    while (i < (int)A.size() && j < (int)B.size()) {
        int lo = max(A[i][0], B[j][0]);
        int hi = min(A[i][1], B[j][1]);
        if (lo <= hi) out.push_back({lo, hi});
        if (A[i][1] < B[j][1]) ++i; else ++j;
    }
    return out;
}
```

---

### 26. Employee Free Time 🔴
```cpp
vector<Interval> employeeFreeTime(vector<vector<Interval>>& schedule) {
    vector<Interval> all;
    for (auto& emp : schedule) for (auto& iv : emp) all.push_back(iv);
    sort(all.begin(), all.end(), [](auto& a, auto& b){ return a.start < b.start; });

    vector<Interval> out;
    int end = all[0].end;
    for (int i = 1; i < (int)all.size(); ++i) {
        if (all[i].start > end) out.push_back({end, all[i].start});   // ⭐ a gap
        end = max(end, all[i].end);
    }
    return out;
}
```
**Key insight:** Flatten, merge, and report the gaps between merged intervals.

---

### 27. My Calendar I 🟡
```cpp
class MyCalendar {
    map<int,int> book;                             // start -> end, kept sorted
public:
    bool book_(int start, int end) {
        auto it = book.lower_bound(start);
        if (it != book.end() && it->first < end) return false;         // next overlaps
        if (it != book.begin() && prev(it)->second > start) return false;  // prev overlaps
        book[start] = end;
        return true;
    }
};
```
**Complexity:** O(log n) per booking.
**Key insight:** An ordered map gives you the neighbors of a proposed interval in O(log n) — only those two can possibly overlap.

---

### 28. My Calendar II (allow double booking) 🟡
```cpp
class MyCalendarTwo {
    map<int,int> delta;                            // sweep-line difference map
public:
    bool book(int start, int end) {
        ++delta[start]; --delta[end];
        int active = 0;
        for (auto& [t, d] : delta) {
            active += d;
            if (active >= 3) {                     // ⭐ triple booking → undo
                --delta[start]; ++delta[end];
                if (delta[start] == 0) delta.erase(start);
                if (delta[end] == 0) delta.erase(end);
                return false;
            }
        }
        return true;
    }
};
```

---

### 29. My Calendar III (max concurrent) 🔴
```cpp
class MyCalendarThree {
    map<int,int> delta;
    int best = 0;
public:
    int book(int start, int end) {
        ++delta[start]; --delta[end];
        int active = 0;
        for (auto& [t, d] : delta) { active += d; best = max(best, active); }
        return best;
    }
};
```
**Key insight:** The difference-map sweep is the general tool for "how many things overlap at once."

---

### 30. Data Stream as Disjoint Intervals 🔴
```cpp
class SummaryRanges {
    map<int,int> iv;                               // start -> end
public:
    void addNum(int x) {
        auto it = iv.upper_bound(x);               // first interval starting after x
        auto pre = (it == iv.begin()) ? iv.end() : prev(it);

        if (pre != iv.end() && pre->second >= x) return;              // already covered
        int start = x, end = x;
        if (pre != iv.end() && pre->second == x - 1) { start = pre->first; iv.erase(pre); }
        if (it != iv.end() && it->first == end + 1) { end = it->second; iv.erase(it); }
        iv[start] = end;                           // ⭐ merges left and/or right
    }
    vector<vector<int>> getIntervals() {
        vector<vector<int>> out;
        for (auto& [s, e] : iv) out.push_back({s, e});
        return out;
    }
}
```
**Key insight:** Each insertion touches at most its two neighbors — check for adjacency on both sides and merge.

---

### Bonus: Car Pooling 🟡
```cpp
bool carPooling(vector<vector<int>>& trips, int cap) {
    map<int,int> delta;
    for (auto& t : trips) { delta[t[1]] += t[0]; delta[t[2]] -= t[0]; }
    int cur = 0;
    for (auto& [t, d] : delta) { cur += d; if (cur > cap) return false; }
    return true;
}
```

---

### Bonus: The Skyline Problem 🔴
```cpp
vector<vector<int>> getSkyline(vector<vector<int>>& b) {
    vector<pair<int,int>> ev;
    for (auto& x : b) {
        ev.push_back({x[0], -x[2]});               // ⭐ negative height = START
        ev.push_back({x[1],  x[2]});               //    positive = END
    }
    sort(ev.begin(), ev.end());
    // Sorting rules that fall out of this encoding:
    //   at the same x, starts (negative) come first  → taller start wins
    //   at the same x, ends sort ascending           → shorter end first

    multiset<int> heights{0};                      // 0 = ground level
    vector<vector<int>> out;
    int prev = 0;
    for (auto& [x, h] : ev) {
        if (h < 0) heights.insert(-h);             // building starts
        else heights.erase(heights.find(h));       // ⭐ erase ONE occurrence

        int cur = *heights.rbegin();               // current max height
        if (cur != prev) { out.push_back({x, cur}); prev = cur; }
    }
    return out;
}
```
**Complexity:** O(n log n).
**Key insight:** The negative-height encoding makes a plain sort produce exactly the right event ordering — a genuinely elegant trick. `multiset` supports removing an arbitrary height, which a heap cannot.

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║              HEAPS & INTERVALS — PATTERN RECALL                   ║
╠═══════════════════════════════════════════════════════════════════╣
║ HEAPS                                                             ║
║   ⭐ K LARGEST → MIN-heap of size k (O(n log k))                   ║
║   ⭐ K SMALLEST → MAX-heap of size k                               ║
║   streaming median → TWO heaps, |lo| == |hi| or |hi|+1            ║
║   merge k sorted → heap of the k current heads                    ║
║   Huffman / min merge cost → always merge the two smallest        ║
║   ⚠️ heap CANNOT delete arbitrary elements or update keys          ║
║      → use multiset, or lazy deletion (skip stale entries)        ║
║   ⚠️ C++ comparator is INVERTED vs a sort comparator               ║
║   quickselect: O(n) avg for k-th, but needs the whole array       ║
╠═══════════════════════════════════════════════════════════════════╣
║ INTERVALS — always SORT first, but sort by WHAT?                  ║
║   merge / insert / intersect  → sort by START                     ║
║   ⭐ max non-overlapping / min arrows → sort by END                ║
║      (activity selection: earliest finish leaves the most room)   ║
║   min rooms / max concurrent → SWEEP LINE                         ║
║      events {start:+1, end:-1}, sort, running sum, track max      ║
║      at equal times, ENDS (-1) must come before STARTS (+1)       ║
║   dynamic insertion/query → ordered map, check only 2 neighbors   ║
║   skyline → encode starts as NEGATIVE height so sort ordering     ║
║             falls out automatically; multiset for the max         ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Graphs →](08-graphs.md)
