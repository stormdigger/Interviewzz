# #️⃣ Hashing & Sets — 30 Problems

> The hash map is the single most valuable data structure in interviews. It converts "search for something" from O(n) to O(1), which is the core move in a majority of optimizations.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 When to Reach for a Hash Map

```
   Signal in the problem                     → Structure
   ─────────────────────                       ─────────
   "have I seen this before?"                → unordered_set
   "how many times does X appear?"           → unordered_map<T,int>
   "group things that share a property"      → unordered_map<key, vector<T>>
   "find a complement/pair"                  → unordered_map<value, index>
   "O(1) insert + delete + random"           → map + vector combo
   "need order too"                          → map (ordered) or map + linked list
   "cache with eviction"                     → map + doubly linked list (LRU)
```

**The core trade:** O(n) space to make lookups O(1). Almost always worth it in an interview.

⚠️ **C++ specifics you must know:**
```cpp
mp[key]                 // ⚠️ INSERTS a default-constructed value if absent
mp.count(key)           // safe existence check
mp.find(key)            // returns iterator; compare with mp.end()
mp.at(key)              // throws if absent
auto [it, inserted] = mp.insert({k, v});   // doesn't overwrite
mp.reserve(n);          // ⭐ avoids rehashing — meaningful speedup
```

---

## A. Counting & Frequency

### 1. Two Sum 🟢
> (Covered in [Arrays](01-arrays-strings.md#1-two-sum-) — the canonical hash map problem.)

```cpp
vector<int> twoSum(vector<int>& a, int t) {
    unordered_map<int,int> seen;
    for (int i = 0; i < (int)a.size(); ++i) {
        auto it = seen.find(t - a[i]);
        if (it != seen.end()) return {it->second, i};
        seen[a[i]] = i;
    }
    return {};
}
```

---

### 2. Valid Anagram 🟢
```cpp
bool isAnagram(string s, string t) {
    if (s.size() != t.size()) return false;
    int c[26] = {};
    for (char x : s) c[x - 'a']++;
    for (char x : t) if (--c[x - 'a'] < 0) return false;
    return true;
}
```
**Key insight:** For a fixed small alphabet, an array beats a hash map — no hashing overhead, perfect cache locality.

---

### 3. Group Anagrams 🟡
```cpp
vector<vector<string>> groupAnagrams(vector<string>& v) {
    unordered_map<string, vector<string>> g;
    for (auto& s : v) {
        array<int,26> c{};
        for (char x : s) c[x - 'a']++;
        string key;
        for (int i = 0; i < 26; ++i) { key += '#'; key += to_string(c[i]); }
        g[key].push_back(s);
    }
    vector<vector<string>> out;
    for (auto& [_, vec] : g) out.push_back(move(vec));
    return out;
}
```
**Complexity:** O(N·L). The `#` separator prevents `[1,11]` and `[11,1]` colliding.

---

### 4. Top K Frequent Elements 🟡
```cpp
vector<int> topKFrequent(vector<int>& a, int k) {
    unordered_map<int,int> cnt;
    for (int x : a) cnt[x]++;
    vector<vector<int>> bucket(a.size() + 1);
    for (auto& [v, c] : cnt) bucket[c].push_back(v);

    vector<int> out;
    for (int f = a.size(); f >= 1 && (int)out.size() < k; --f)
        for (int v : bucket[f]) { out.push_back(v); if ((int)out.size() == k) break; }
    return out;
}
```
**Complexity:** O(n) via bucket sort — frequency is bounded by n, so no comparison sort is needed.

---

### 5. Top K Frequent Words 🟡
```cpp
vector<string> topKFrequent(vector<string>& w, int k) {
    unordered_map<string,int> cnt;
    for (auto& s : w) cnt[s]++;

    // min-heap of size k; ties broken lexicographically (REVERSED, since
    // we pop the "worst" element)
    auto cmp = [](const pair<string,int>& a, const pair<string,int>& b) {
        if (a.second != b.second) return a.second > b.second;
        return a.first < b.first;
    };
    priority_queue<pair<string,int>, vector<pair<string,int>>, decltype(cmp)> pq(cmp);

    for (auto& p : cnt) { pq.push(p); if ((int)pq.size() > k) pq.pop(); }

    vector<string> out;
    while (!pq.empty()) { out.push_back(pq.top().first); pq.pop(); }
    reverse(out.begin(), out.end());
    return out;
}
```
⚠️ **The comparator is reversed** relative to intuition, because a min-heap pops the element you want to discard.

---

### 6. First Unique Character 🟢
```cpp
int firstUniqChar(string s) {
    int c[26] = {};
    for (char x : s) c[x - 'a']++;
    for (int i = 0; i < (int)s.size(); ++i) if (c[s[i] - 'a'] == 1) return i;
    return -1;
}
```

---

### 7. Sort Characters By Frequency 🟡
```cpp
string frequencySort(string s) {
    unordered_map<char,int> cnt;
    for (char c : s) cnt[c]++;
    vector<string> bucket(s.size() + 1);
    for (auto& [c, f] : cnt) bucket[f] += string(f, c);
    string out;
    for (int f = s.size(); f >= 1; --f) out += bucket[f];
    return out;
}
```

---

### 8. Find All Anagrams in a String 🟡
```cpp
vector<int> findAnagrams(string s, string p) {
    if (s.size() < p.size()) return {};
    int need[26] = {}, win[26] = {};
    for (char c : p) need[c - 'a']++;

    vector<int> out;
    for (int i = 0; i < (int)s.size(); ++i) {
        win[s[i] - 'a']++;
        if (i >= (int)p.size()) win[s[i - p.size()] - 'a']--;   // slide
        if (i >= (int)p.size() - 1 && equal(need, need + 26, win))
            out.push_back(i - p.size() + 1);
    }
    return out;
}
```
**Complexity:** O(26n). Track a `matched` counter instead of comparing arrays for true O(n).

---

### 9. Ransom Note 🟢
```cpp
bool canConstruct(string note, string mag) {
    int c[26] = {};
    for (char x : mag) c[x - 'a']++;
    for (char x : note) if (--c[x - 'a'] < 0) return false;
    return true;
}
```

---

### 10. Isomorphic Strings 🟢
```cpp
bool isIsomorphic(string s, string t) {
    int m1[256] = {}, m2[256] = {};                // 0 = unmapped
    for (int i = 0; i < (int)s.size(); ++i) {
        if (m1[(unsigned char)s[i]] != m2[(unsigned char)t[i]]) return false;
        m1[(unsigned char)s[i]] = m2[(unsigned char)t[i]] = i + 1;   // ⭐ 1-indexed
    }
    return true;
}
```
**Key insight:** Storing `i+1` avoids needing a separate "seen" flag, and requiring **both** maps enforces a bijection. One map alone would accept `"ab" → "aa"`.

---

### 11. Word Pattern 🟢
```cpp
bool wordPattern(string p, string s) {
    vector<string> words;
    istringstream iss(s);
    string w;
    while (iss >> w) words.push_back(w);
    if (words.size() != p.size()) return false;

    unordered_map<char,int> c2i;
    unordered_map<string,int> w2i;
    for (int i = 0; i < (int)p.size(); ++i) {
        if (c2i[p[i]] != w2i[words[i]]) return false;
        c2i[p[i]] = w2i[words[i]] = i + 1;
    }
    return true;
}
```

---

## B. Existence & Deduplication

### 12. Contains Duplicate II (within k distance) 🟢
```cpp
bool containsNearbyDuplicate(vector<int>& a, int k) {
    unordered_map<int,int> last;
    for (int i = 0; i < (int)a.size(); ++i) {
        auto it = last.find(a[i]);
        if (it != last.end() && i - it->second <= k) return true;
        last[a[i]] = i;
    }
    return true, false;
}
```

---

### 13. Contains Duplicate III (value and index window) 🔴
```cpp
bool containsNearbyAlmostDuplicate(vector<int>& a, int k, int t) {
    if (t < 0 || k <= 0) return false;
    set<long long> win;                            // ordered set = the window
    for (int i = 0; i < (int)a.size(); ++i) {
        auto it = win.lower_bound((long long)a[i] - t);
        if (it != win.end() && *it <= (long long)a[i] + t) return true;
        win.insert(a[i]);
        if ((int)win.size() > k) win.erase(a[i - k]);   // maintain window size
    }
    return false;
}
```
**Complexity:** O(n log k). Bucketing gives O(n).
**Key insight:** An ordered set lets you ask "is there any value within ±t?" in O(log k) — a plain hash map cannot answer range queries.

---

### 14. Longest Consecutive Sequence 🟡
> Find the length of the longest run of consecutive integers, in O(n). The array is unsorted.

#### 💬 Think of it like this
Sorting would make this trivial, but sorting is O(n log n) and we want O(n).

Put everything in a set, so membership checks are O(1). Now for any number you could walk upward — is `x+1` present? `x+2`? — counting the run.

But doing that from *every* number re-walks the same sequences repeatedly. For the run 1,2,3,4 you'd walk it from 1, then again from 2, then from 3.

**The fix is one line: only start walking from the beginning of a run.** A number `x` starts a run only if `x-1` is *not* in the set. If `x-1` exists, you're standing in the middle of a sequence someone else will walk, so skip it.

#### 📊 Tracing `[100, 4, 200, 1, 3, 2]`

```
   set = {100, 4, 200, 1, 3, 2}

   ┌─────────────────────────────────────────────────────────────┐
   │ x=100:  is 99 in the set?  NO  → ⭐ this STARTS a run        │
   │         walk: 101? no.  run length = 1                      │
   ├─────────────────────────────────────────────────────────────┤
   │ x=4:    is 3 in the set?   YES → ⚠️ SKIP (mid-sequence)      │
   ├─────────────────────────────────────────────────────────────┤
   │ x=200:  is 199 in the set? NO  → starts a run               │
   │         walk: 201? no.  run length = 1                      │
   ├─────────────────────────────────────────────────────────────┤
   │ x=1:    is 0 in the set?   NO  → ⭐ starts a run             │
   │         walk: 2? yes. 3? yes. 4? yes. 5? no.                │
   │         run length = 4  ⭐ BEST                              │
   ├─────────────────────────────────────────────────────────────┤
   │ x=3:    is 2 in the set?   YES → SKIP                       │
   │ x=2:    is 1 in the set?   YES → SKIP                       │
   └─────────────────────────────────────────────────────────────┘

   ANSWER = 4   (the run 1,2,3,4)
```

#### Why this is genuinely O(n), despite the nested loop

```
   ⭐ The inner while-loop only ever executes for numbers that
     START a run. And a run of length L is walked exactly ONCE,
     costing L steps.

     Since every number belongs to exactly one run, the total
     inner work summed across the whole outer loop is at most n.

   Total = n (outer) + n (all inner walks combined) = O(n)

   ⚠️ Remove the `if (s.count(x-1)) continue;` guard and it
     becomes O(n²) — that single line is what makes the
     complexity work.
```

```cpp
int longestConsecutive(vector<int>& a) {
    unordered_set<int> s(a.begin(), a.end());
    int best = 0;
    for (int x : s) {
        if (s.count(x - 1)) continue;              // ⭐ only start from sequence heads
        int len = 1;
        while (s.count(x + len)) ++len;
        best = max(best, len);
    }
    return best;
}
```
**Complexity:** O(n) — despite the nested loop.
**Key insight:** The `if (s.count(x-1)) continue` guard means each sequence is walked exactly once, from its smallest element. Total inner work across all iterations is O(n).

---

### 15. Happy Number 🟢
```cpp
int sq(int n) { int s = 0; while (n) { int d = n % 10; s += d * d; n /= 10; } return s; }

bool isHappy(int n) {
    int slow = n, fast = n;                        // Floyd — O(1) space
    do { slow = sq(slow); fast = sq(sq(fast)); } while (slow != fast);
    return slow == 1;
}
```

---

### 16. Intersection of Two Arrays 🟢
```cpp
vector<int> intersection(vector<int>& a, vector<int>& b) {
    unordered_set<int> sa(a.begin(), a.end()), out;
    for (int x : b) if (sa.count(x)) out.insert(x);
    return vector<int>(out.begin(), out.end());
}
```

---

### 17. Intersection of Two Arrays II (with multiplicity) 🟢
```cpp
vector<int> intersect(vector<int>& a, vector<int>& b) {
    unordered_map<int,int> cnt;
    for (int x : a) cnt[x]++;
    vector<int> out;
    for (int x : b) { auto it = cnt.find(x); if (it != cnt.end() && it->second > 0) { out.push_back(x); it->second--; } }
    return out;
}
```
🎤 **Follow-up:** if the arrays are sorted, use two pointers with O(1) space. If one array is huge and on disk, hash the smaller one and stream the larger.

---

### 18. Valid Sudoku 🟡
```cpp
bool isValidSudoku(vector<vector<char>>& b) {
    unordered_set<string> seen;
    for (int i = 0; i < 9; ++i)
        for (int j = 0; j < 9; ++j) {
            if (b[i][j] == '.') continue;
            char c = b[i][j];
            string r = "r" + to_string(i) + c;
            string col = "c" + to_string(j) + c;
            string box = "b" + to_string(i/3) + to_string(j/3) + c;
            if (!seen.insert(r).second || !seen.insert(col).second
                || !seen.insert(box).second) return false;
        }
    return true;
}
```
**Key insight:** Encoding all three constraints as strings in one set is clean; the bitmask version (in Arrays §53) is faster.

---

## C. Design Problems

### 19. LRU Cache 🟡
> `get` and `put` both O(1). Evict the least recently used item when full.

#### 💬 Think of it like this
You need two things simultaneously, and no single data structure gives you both:

- **Fast lookup by key** → that's a hash map
- **Fast "which item was used longest ago?" and fast reordering** → that's an ordered list

So use both, pointing at the same nodes.

The list is ordered by recency: most-recently-used at the front, least-recently-used at the back. Every access moves that node to the front. When you're full, you evict from the back — which is O(1) because you know exactly where it is.

The list must be **doubly** linked. With a singly linked list, removing a node from the middle requires finding its predecessor, which is O(n). With a `prev` pointer, unlinking is O(1).

#### 📊 The structure

```
   HASH MAP                    DOUBLY LINKED LIST
   key → node pointer          (ordered by recency)

   ┌─────────┐
   │ "a" ────┼──────┐   head ⇄ [c] ⇄ [b] ⇄ [a] ⇄ tail
   │ "b" ────┼───┐  │           ▲               ▲
   │ "c" ────┼┐  │  │       most recent    least recent
   └─────────┘│  │  │                      ⭐ evict from here
              └──┴──┘
        pointers into the same nodes

   ⭐ The map gives O(1) "where is this node?"
     The list gives O(1) "reorder" and O(1) "who's oldest?"
```

#### Watching a sequence with capacity 2

```
   put(1,A):  head ⇄ [1] ⇄ tail                      map {1}
   put(2,B):  head ⇄ [2] ⇄ [1] ⇄ tail                map {1,2}

   get(1):    found. ⭐ MOVE TO FRONT.
              head ⇄ [1] ⇄ [2] ⇄ tail                map {1,2}

   put(3,C):  ⚠️ at capacity → evict from the TAIL, which is [2]
              head ⇄ [3] ⇄ [1] ⇄ tail                map {1,3}
                                 ▲
                          key 2 removed from BOTH
                          the list and the map

   ⭐ Note that get(1) is what saved key 1 from eviction.
     That's the whole point of "least recently USED" rather
     than "least recently inserted."
```

#### Why sentinel nodes matter

```
   Without sentinels, every insert and remove needs null checks:
     "is this the head? is this the tail? is the list empty?"

   ⭐ With permanent dummy head and tail nodes that never hold
     data, every real node ALWAYS has both a prev and a next.
     The unlink and insert operations become three lines with
     no branching.

   This is a general technique worth remembering — it applies
   to most linked-structure problems.
```

```cpp
class LRUCache {
    struct Node { int k, v; Node *prev, *next; };
    int cap;
    unordered_map<int, Node*> mp;
    Node *head, *tail;                             // sentinels

    void remove(Node* n) { n->prev->next = n->next; n->next->prev = n->prev; }
    void addFront(Node* n) {
        n->next = head->next; n->prev = head;
        head->next->prev = n; head->next = n;
    }
public:
    LRUCache(int c) : cap(c) {
        head = new Node(); tail = new Node();
        head->next = tail; tail->prev = head;
    }
    int get(int k) {
        auto it = mp.find(k);
        if (it == mp.end()) return -1;
        remove(it->second); addFront(it->second);  // mark as recently used
        return it->second->v;
    }
    void put(int k, int v) {
        auto it = mp.find(k);
        if (it != mp.end()) {
            it->second->v = v;
            remove(it->second); addFront(it->second);
            return;
        }
        if ((int)mp.size() == cap) {
            Node* lru = tail->prev;                // evict from the tail
            remove(lru); mp.erase(lru->k); delete lru;
        }
        Node* n = new Node{k, v, nullptr, nullptr};
        mp[k] = n; addFront(n);
    }
};
```
**Key insight:** Hash map gives O(1) lookup; the doubly linked list gives O(1) reorder and eviction. Neither alone suffices. Sentinel head/tail nodes eliminate every null check.

⭐ In C++ you can also use `list<pair<int,int>>` plus `unordered_map<int, list<...>::iterator>` with `splice()`.

---

### 20. LFU Cache 🔴
```cpp
class LFUCache {
    int cap, minFreq = 0;
    unordered_map<int, pair<int,int>> kv;                       // key -> {value, freq}
    unordered_map<int, list<int>> freqList;                     // freq -> keys (LRU order)
    unordered_map<int, list<int>::iterator> pos;                // key -> iterator

    void touch(int k) {
        int f = kv[k].second;
        freqList[f].erase(pos[k]);
        if (freqList[f].empty()) { freqList.erase(f); if (minFreq == f) ++minFreq; }
        ++kv[k].second;
        freqList[f + 1].push_front(k);
        pos[k] = freqList[f + 1].begin();
    }
public:
    LFUCache(int c) : cap(c) {}
    int get(int k) {
        if (!kv.count(k) || cap == 0) return -1;
        touch(k);
        return kv[k].first;
    }
    void put(int k, int v) {
        if (cap == 0) return;
        if (kv.count(k)) { kv[k].first = v; touch(k); return; }
        if ((int)kv.size() == cap) {
            int evict = freqList[minFreq].back();               // least frequent, then LRU
            freqList[minFreq].pop_back();
            if (freqList[minFreq].empty()) freqList.erase(minFreq);
            kv.erase(evict); pos.erase(evict);
        }
        kv[k] = {v, 1};
        freqList[1].push_front(k);
        pos[k] = freqList[1].begin();
        minFreq = 1;                                            // ⭐ reset
    }
};
```
**Key insight:** Three maps. `minFreq` only ever increases by 1 in `touch` and resets to 1 on insert, so tracking it is O(1).

---

### 21. Insert Delete GetRandom O(1) 🟡
```cpp
class RandomizedSet {
    vector<int> v;
    unordered_map<int,int> idx;                    // value -> index in v
public:
    bool insert(int x) {
        if (idx.count(x)) return false;
        idx[x] = v.size();
        v.push_back(x);
        return true;
    }
    bool remove(int x) {
        auto it = idx.find(x);
        if (it == idx.end()) return false;
        int i = it->second, last = v.back();
        v[i] = last; idx[last] = i;                // ⭐ swap with the last element
        v.pop_back(); idx.erase(it);
        return true;
    }
    int getRandom() { return v[rand() % v.size()]; }
};
```
**Key insight:** The vector gives O(1) random access; the map gives O(1) lookup. Deletion is O(1) by swapping the target with the last element — order doesn't matter for random selection.

---

### 22. Insert Delete GetRandom O(1) — Duplicates Allowed 🔴
```cpp
class RandomizedCollection {
    vector<int> v;
    unordered_map<int, unordered_set<int>> idx;    // value -> set of indices
public:
    bool insert(int x) {
        bool isNew = idx[x].empty();
        idx[x].insert(v.size());
        v.push_back(x);
        return isNew;
    }
    bool remove(int x) {
        auto it = idx.find(x);
        if (it == idx.end() || it->second.empty()) return false;
        int i = *it->second.begin();
        it->second.erase(i);

        int last = v.back();
        v[i] = last;
        if (i != (int)v.size() - 1) {              // ⭐ guard self-swap
            idx[last].erase(v.size() - 1);
            idx[last].insert(i);
        }
        v.pop_back();
        return true;
    }
    int getRandom() { return v[rand() % v.size()]; }
};
```
⚠️ The self-swap guard matters when removing the last element — otherwise you erase the index you just inserted.

---

### 23. Design HashMap 🟢
```cpp
class MyHashMap {
    static const int N = 10007;                    // prime bucket count
    vector<list<pair<int,int>>> b;
public:
    MyHashMap() : b(N) {}
    void put(int k, int v) {
        for (auto& p : b[k % N]) if (p.first == k) { p.second = v; return; }
        b[k % N].push_back({k, v});
    }
    int get(int k) {
        for (auto& p : b[k % N]) if (p.first == k) return p.second;
        return -1;
    }
    void remove(int k) {
        b[k % N].remove_if([k](auto& p){ return p.first == k; });
    }
};
```
🎤 **Follow-ups:** load factor and resizing; open addressing vs chaining; why a prime modulus reduces clustering.

---

### 24. Design Twitter 🟡
```cpp
class Twitter {
    int time = 0;
    unordered_map<int, vector<pair<int,int>>> tweets;      // user -> {time, tweetId}
    unordered_map<int, unordered_set<int>> follows;
public:
    void postTweet(int u, int t) { tweets[u].push_back({time++, t}); }

    vector<int> getNewsFeed(int u) {
        priority_queue<pair<int,int>> pq;                  // max-heap by time
        auto add = [&](int uid) {
            auto& v = tweets[uid];
            for (int i = max(0, (int)v.size() - 10); i < (int)v.size(); ++i)
                pq.push(v[i]);                             // ⭐ only the last 10 each
        };
        add(u);
        for (int f : follows[u]) if (f != u) add(f);

        vector<int> out;
        while (!pq.empty() && (int)out.size() < 10) { out.push_back(pq.top().second); pq.pop(); }
        return out;
    }
    void follow(int a, int b)   { follows[a].insert(b); }
    void unfollow(int a, int b) { follows[a].erase(b); }
};
```
🎤 This is a mini system-design question — see [Twitter case study](../05-system-design/03-case-studies-1.md) for fanout-on-write vs fanout-on-read at scale.

---

### 25. Design Underground System 🟡
```cpp
class UndergroundSystem {
    unordered_map<int, pair<string,int>> inTransit;                    // id -> {station, t}
    unordered_map<string, pair<long long,int>> stats;                  // route -> {sum, count}
public:
    void checkIn(int id, string s, int t) { inTransit[id] = {s, t}; }
    void checkOut(int id, string e, int t) {
        auto [s, st] = inTransit[id];
        inTransit.erase(id);
        auto& [sum, cnt] = stats[s + "->" + e];
        sum += t - st; ++cnt;
    }
    double getAverageTime(string s, string e) {
        auto& [sum, cnt] = stats[s + "->" + e];
        return (double)sum / cnt;
    }
};
```

---

### 26. Time Based Key-Value Store 🟡
```cpp
class TimeMap {
    unordered_map<string, vector<pair<int,string>>> m;   // timestamps are increasing
public:
    void set(string k, string v, int t) { m[k].push_back({t, v}); }
    string get(string k, int t) {
        auto it = m.find(k);
        if (it == m.end()) return "";
        auto& v = it->second;
        // largest timestamp <= t
        int lo = 0, hi = v.size();
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (v[mid].first <= t) lo = mid + 1; else hi = mid;
        }
        return lo == 0 ? "" : v[lo - 1].second;
    }
};
```
**Key insight:** Timestamps arrive in increasing order, so the vector is already sorted — binary search directly, no sorting needed.

---

### 27. Logger Rate Limiter 🟢
```cpp
class Logger {
    unordered_map<string,int> last;
public:
    bool shouldPrintMessage(int t, string msg) {
        auto it = last.find(msg);
        if (it != last.end() && t < it->second + 10) return false;
        last[msg] = t;
        return true;
    }
};
```
🎤 **Follow-up:** unbounded memory growth. Fix with a sliding-window purge or an LRU-bounded map.

---

### 28. Design Hit Counter 🟡
```cpp
class HitCounter {
    deque<pair<int,int>> q;                        // {timestamp, count}
    int total = 0;
    void evict(int t) {
        while (!q.empty() && q.front().first <= t - 300) { total -= q.front().second; q.pop_front(); }
    }
public:
    void hit(int t) {
        evict(t);
        if (!q.empty() && q.back().first == t) q.back().second++;
        else q.push_back({t, 1});
        ++total;
    }
    int getHits(int t) { evict(t); return total; }
};
```
**Key insight:** Aggregating by timestamp bounds memory at 300 entries regardless of hit rate.

---

## D. Advanced Hashing

### 29. Longest Substring Without Repeating Characters 🟡
```cpp
int lengthOfLongestSubstring(string s) {
    vector<int> last(128, -1);
    int left = 0, best = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        left = max(left, last[s[r]] + 1);          // ⭐ never move left backwards
        last[s[r]] = r;
        best = max(best, r - left + 1);
    }
    return best;
}
```
**Key insight:** `max` is essential — a repeat *before* the current window shouldn't drag `left` back.

---

### 30. Subarray Sum Equals K 🟡
```cpp
int subarraySum(vector<int>& a, int k) {
    unordered_map<long long,int> cnt{{0, 1}};
    long long sum = 0; int ans = 0;
    for (int x : a) { sum += x; ans += cnt[sum - k]; cnt[sum]++; }
    return ans;
}
```
⚠️ Note `cnt[sum - k]` uses `operator[]`, which inserts a zero entry. That's harmless here (it returns 0 correctly) but grows the map. `find` is cleaner in production code.

---

### 31. 4Sum II 🟡
```cpp
int fourSumCount(vector<int>& a, vector<int>& b, vector<int>& c, vector<int>& d) {
    unordered_map<int,int> ab;
    for (int x : a) for (int y : b) ab[x + y]++;   // O(n²)
    int cnt = 0;
    for (int x : c) for (int y : d) {
        auto it = ab.find(-(x + y));
        if (it != ab.end()) cnt += it->second;
    }
    return cnt;
}
```
**Complexity:** O(n²) instead of O(n⁴).
**Key insight:** **Meet in the middle** — split four loops into two pairs and join with a hash map. This technique generalizes to many exponential problems.

---

### 32. Copy List with Random Pointer 🟡
```cpp
Node* copyRandomList(Node* head) {
    if (!head) return nullptr;
    unordered_map<Node*, Node*> mp;
    for (Node* p = head; p; p = p->next) mp[p] = new Node(p->val);
    for (Node* p = head; p; p = p->next) {
        mp[p]->next = mp[p->next];                 // nullptr maps to nullptr
        mp[p]->random = mp[p->random];
    }
    return mp[head];
}
```
🎤 **Follow-up (O(1) space):** interleave copies into the original list, wire up randoms via `p->next->random = p->random->next`, then split the lists.

---

### 33. Word Break 🟡
```cpp
bool wordBreak(string s, vector<string>& dict) {
    unordered_set<string> d(dict.begin(), dict.end());
    int n = s.size();
    vector<bool> dp(n + 1, false);
    dp[0] = true;
    for (int i = 1; i <= n; ++i)
        for (int j = 0; j < i; ++j)
            if (dp[j] && d.count(s.substr(j, i - j))) { dp[i] = true; break; }
    return dp[n];
}
```
**Complexity:** O(n²·L). Hashing turns dictionary lookup into O(L).

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║                    HASHING — PATTERN RECALL                       ║
╠═══════════════════════════════════════════════════════════════════╣
║ "find a pair/complement"        → map value→index, check          ║
║                                   BEFORE inserting                ║
║ "group by property"             → map<canonicalKey, vector>       ║
║   anagram key = count array, NOT sorted string (O(L) vs O(L logL))║
║ "top k frequent"                → BUCKET SORT by frequency, O(n)  ║
║ "longest consecutive"           → set + only start from heads     ║
║ "O(1) insert/delete/random"     → vector + map, swap with last    ║
║ "LRU"                           → map + doubly linked list        ║
║ "O(n⁴) with 4 arrays"           → MEET IN THE MIDDLE, 2+2 → O(n²) ║
║ "range query on values"         → ordered set, NOT hash map       ║
╠═══════════════════════════════════════════════════════════════════╣
║ C++ TRAPS                                                         ║
║   mp[key] INSERTS on read — use find() or count()                 ║
║   small fixed alphabet → use int[26], not a hash map              ║
║   mp.reserve(n) avoids rehashing                                  ║
║   unordered_map is O(n) worst case (adversarial hash collisions)  ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Two Pointers & Sliding Window →](03-two-pointers-sliding-window.md)
