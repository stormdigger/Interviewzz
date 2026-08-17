# 🕸️ Graphs — 50 Problems

> Graph problems look intimidating but reduce to a small set of algorithms. The real skill is **recognizing that a problem is a graph problem** — grids, dependencies, word transformations, and state machines are all graphs in disguise.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 Representations & Templates

```cpp
// ADJACENCY LIST — the default. O(V+E) space.
vector<vector<int>> adj(n);
for (auto& e : edges) { adj[e[0]].push_back(e[1]); adj[e[1]].push_back(e[0]); }

// Weighted
vector<vector<pair<int,int>>> adj(n);              // {neighbor, weight}

// ADJACENCY MATRIX — O(V²) space. Only for dense graphs or V ≤ ~500.
vector<vector<int>> mat(n, vector<int>(n, 0));

// GRID as an implicit graph — ⭐ extremely common
const int dr[] = {-1, 1, 0, 0}, dc[] = {0, 0, -1, 1};
```

### The algorithm selector

```
                        What is being asked?
                                │
   ┌──────────────┬─────────────┼──────────────┬─────────────────┐
   ▼              ▼             ▼              ▼                 ▼
 SHORTEST      CONNECTIVITY  ORDERING      ALL PATHS         MST
 PATH          COMPONENTS    DEPENDENCIES  / CYCLES
   │              │             │              │                 │
   ▼              ▼             ▼              ▼                 ▼
 unweighted    DFS/BFS        Topological     DFS +           Kruskal(DSU)
   → BFS       or DSU         sort            backtracking     or Prim(heap)
 weighted ≥0                  (Kahn or DFS)
   → Dijkstra  DSU also for
 negative      DYNAMIC
   → Bellman-  connectivity
     Ford
 all pairs
   → Floyd-Warshall
 {0,1} weights
   → 0-1 BFS (deque)
```

---

## A. Grid Traversal

### 1. Number of Islands 🟡
> Given a grid of `'1'` (land) and `'0'` (water), count how many separate islands there are. Land connects horizontally and vertically.

#### 💬 Think of it like this
Imagine the grid is a real map and you have a bucket of black paint. You walk the map cell by cell. The moment you step on a piece of land you've never seen before, you shout **"new island!"** and increment your counter. Then — before moving on — you pour paint over that *entire* island: you flood from where you stand to every connected land cell, painting each one black so you can never count it again.

By the time you finish scanning the whole map, every island has been painted exactly once, and your counter holds the answer.

The "pouring paint" step is the recursion. It spreads outward in all four directions, and each cell it touches gets marked so the spread doesn't bounce back.

#### 📊 Watching it work

```
   START                    Scan finds (0,0) = land → count = 1
   ┌───┬───┬───┬───┐        Flood fill spreads from there:
   │ 1 │ 1 │ 0 │ 0 │
   ├───┼───┼───┼───┤          (0,0)→(0,1)→(1,1)   all become 0
   │ 0 │ 1 │ 0 │ 1 │
   ├───┼───┼───┼───┤        ┌───┬───┬───┬───┐
   │ 0 │ 0 │ 0 │ 1 │        │ 0 │ 0 │ 0 │ 0 │   island 1 erased
   └───┴───┴───┴───┘        ├───┼───┼───┼───┤
                            │ 0 │ 0 │ 0 │ 1 │
   Keep scanning...         ├───┼───┼───┼───┤
   find (1,3) = land        │ 0 │ 0 │ 0 │ 1 │
   → count = 2              └───┴───┴───┴───┘
   flood (1,3) and (2,3)
                            ┌───┬───┬───┬───┐
                            │ 0 │ 0 │ 0 │ 0 │   grid fully erased
                            ├───┼───┼───┼───┤
                            │ 0 │ 0 │ 0 │ 0 │   ANSWER = 2
                            ├───┼───┼───┼───┤
                            │ 0 │ 0 │ 0 │ 0 │
                            └───┴───┴───┴───┘
```

#### How the flood spreads from one cell

```
            (r-1, c)
               ▲
               │
   (r, c-1) ◀──●──▶ (r, c+1)      each arrow is a recursive call
               │                   each call repeats the same 4 arrows
               ▼                   until it hits water or the edge
            (r+1, c)
```

```cpp
int numIslands(vector<vector<char>>& g) {
    int R = g.size(), C = g[0].size(), count = 0;

    // "Pour paint" — erase every land cell connected to (r,c)
    function<void(int,int)> sink = [&](int r, int c) {
        // Stop if we walked off the map, or hit water/already-painted land
        if (r < 0 || r >= R || c < 0 || c >= C || g[r][c] != '1') return;

        g[r][c] = '0';          // ⭐ paint it — this is our "visited" marker
                                //    (no separate visited array needed)

        sink(r-1, c);           // spread up
        sink(r+1, c);           // spread down
        sink(r, c-1);           // spread left
        sink(r, c+1);           // spread right
    };

    // Walk every cell; each unpainted land cell starts a NEW island
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c)
            if (g[r][c] == '1') { ++count; sink(r, c); }

    return count;
}
```

**Why is this O(R·C) and not worse?** Every cell is painted at most once. Once painted it returns immediately on any future visit. So the total work across *all* flood fills is bounded by the number of cells.

⚠️ **Real-world gotcha:** recursion depth can reach R·C. On a 1000×1000 all-land grid that's a million stack frames and your program crashes. In production or for large inputs, convert to BFS with a queue, or DFS with an explicit stack.

---

### 2. Max Area of Island 🟡
```cpp
int maxAreaOfIsland(vector<vector<int>>& g) {
    int R = g.size(), C = g[0].size(), best = 0;
    function<int(int,int)> area = [&](int r, int c) -> int {
        if (r < 0 || r >= R || c < 0 || c >= C || g[r][c] != 1) return 0;
        g[r][c] = 0;
        return 1 + area(r-1,c) + area(r+1,c) + area(r,c-1) + area(r,c+1);
    };
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c) best = max(best, area(r, c));
    return best;
}
```

---

### 3. Rotting Oranges 🟡
> Grid of `0` (empty), `1` (fresh orange), `2` (rotten). Every minute, a rotten orange rots its 4 neighbours. Return the minutes until nothing fresh remains, or `-1` if impossible.

#### 💬 Think of it like this
This is rot **spreading like a wave**. At minute 0, several oranges are already rotten — maybe five of them, scattered around the box. At minute 1, all five simultaneously infect their neighbours. At minute 2, that whole new ring infects *its* neighbours.

The critical realization: you do **not** run a separate simulation from each rotten orange. You put *all* of them into the queue at the start, and let one single wave expand from all of them at once. This is called **multi-source BFS**, and it answers "distance to the *nearest* source" for every cell in one pass.

The minute counter is just "how many rings has the wave expanded." That's why we process the queue one full level at a time.

#### 📊 The wave expanding

```
   minute 0              minute 1              minute 2
   ┌───┬───┬───┐        ┌───┬───┬───┐        ┌───┬───┬───┐
   │ 2 │ 1 │ 1 │        │ 2 │ 2 │ 1 │        │ 2 │ 2 │ 2 │
   ├───┼───┼───┤        ├───┼───┼───┤        ├───┼───┼───┤
   │ 1 │ 1 │ 0 │        │ 2 │ 1 │ 0 │        │ 2 │ 2 │ 0 │
   ├───┼───┼───┤        ├───┼───┼───┤        ├───┼───┼───┤
   │ 0 │ 1 │ 1 │        │ 0 │ 1 │ 1 │        │ 0 │ 2 │ 1 │
   └───┴───┴───┘        └───┴───┴───┘        └───┴───┴───┘
   fresh = 6            fresh = 4            fresh = 2

                          minute 3              minute 4
                        ┌───┬───┬───┐        ┌───┬───┬───┐
                        │ 2 │ 2 │ 2 │        │ 2 │ 2 │ 2 │
                        ├───┼───┼───┤        ├───┼───┼───┤
                        │ 2 │ 2 │ 0 │        │ 2 │ 2 │ 0 │
                        ├───┼───┼───┤        ├───┼───┼───┤
                        │ 0 │ 2 │ 2 │        │ 0 │ 2 │ 2 │
                        └───┴───┴───┘        └───┴───┴───┘
                        fresh = 0  →  ANSWER = 4
```

#### Why `int sz = q.size()` matters

```
   Queue at the start of minute 1:  [ A ][ B ][ C ]
                                     └────sz=3────┘

   We process EXACTLY these 3, and the new cells they add
   go behind them — they belong to minute 2, not minute 1.

   After processing:  [ D ][ E ][ F ][ G ]   ← next level
                       └───── minute 2 ─────┘

   Without freezing sz, the loop would run into the newly
   added cells and the minute counter would be meaningless.
```

```cpp
int orangesRotting(vector<vector<int>>& g) {
    int R = g.size(), C = g[0].size(), fresh = 0;
    queue<pair<int,int>> q;
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c) {
            if (g[r][c] == 2) q.push({r, c});      // ⭐ MULTI-SOURCE BFS
            else if (g[r][c] == 1) ++fresh;
        }

    int minutes = 0;
    while (!q.empty() && fresh) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            auto [r, c] = q.front(); q.pop();
            for (int d = 0; d < 4; ++d) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr < 0 || nr >= R || nc < 0 || nc >= C || g[nr][nc] != 1) continue;
                g[nr][nc] = 2; --fresh;
                q.push({nr, nc});
            }
        }
        ++minutes;
    }
    return fresh ? -1 : minutes;
}
```
**Key insight:** **Multi-source BFS** — seed the queue with *all* starting points. This computes the distance from the nearest source to every cell in one pass, rather than running BFS per source.

---

### 4. 01 Matrix (distance to nearest 0) 🟡
```cpp
vector<vector<int>> updateMatrix(vector<vector<int>>& m) {
    int R = m.size(), C = m[0].size();
    vector<vector<int>> dist(R, vector<int>(C, -1));
    queue<pair<int,int>> q;
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c)
            if (m[r][c] == 0) { dist[r][c] = 0; q.push({r, c}); }

    while (!q.empty()) {
        auto [r, c] = q.front(); q.pop();
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= R || nc < 0 || nc >= C || dist[nr][nc] != -1) continue;
            dist[nr][nc] = dist[r][c] + 1;
            q.push({nr, nc});
        }
    }
    return dist;
}
```

---

### 5. Surrounded Regions 🟡
```cpp
void solve(vector<vector<char>>& b) {
    if (b.empty()) return;
    int R = b.size(), C = b[0].size();
    function<void(int,int)> mark = [&](int r, int c) {
        if (r < 0 || r >= R || c < 0 || c >= C || b[r][c] != 'O') return;
        b[r][c] = '#';                             // ⭐ safe: connected to a border
        mark(r-1,c); mark(r+1,c); mark(r,c-1); mark(r,c+1);
    };
    for (int r = 0; r < R; ++r) { mark(r, 0); mark(r, C-1); }
    for (int c = 0; c < C; ++c) { mark(0, c); mark(R-1, c); }

    for (auto& row : b) for (char& ch : row)
        ch = (ch == '#') ? 'O' : 'X';
}
```
**Key insight:** Invert the problem — instead of finding surrounded regions, find the *unsurrounded* ones by starting from the borders.

---

### 6. Pacific Atlantic Water Flow 🟡
```cpp
vector<vector<int>> pacificAtlantic(vector<vector<int>>& h) {
    int R = h.size(), C = h[0].size();
    vector<vector<bool>> pac(R, vector<bool>(C, false)), atl = pac;

    function<void(int,int,vector<vector<bool>>&)> dfs =
        [&](int r, int c, vector<vector<bool>>& vis) {
        vis[r][c] = true;
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
            if (vis[nr][nc] || h[nr][nc] < h[r][c]) continue;   // ⭐ flow UPHILL
            dfs(nr, nc, vis);
        }
    };

    for (int r = 0; r < R; ++r) { dfs(r, 0, pac); dfs(r, C-1, atl); }
    for (int c = 0; c < C; ++c) { dfs(0, c, pac); dfs(R-1, c, atl); }

    vector<vector<int>> out;
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c)
            if (pac[r][c] && atl[r][c]) out.push_back({r, c});
    return out;
}
```
**Key insight:** Reverse the flow direction. Instead of asking "can this cell reach the ocean?" for every cell, ask "which cells can the ocean reach going uphill?" — two traversals instead of R·C.

---

### 7. Word Search 🟡
```cpp
bool exist(vector<vector<char>>& b, string word) {
    int R = b.size(), C = b[0].size();
    function<bool(int,int,int)> dfs = [&](int r, int c, int i) -> bool {
        if (i == (int)word.size()) return true;
        if (r < 0 || r >= R || c < 0 || c >= C || b[r][c] != word[i]) return false;
        char tmp = b[r][c];
        b[r][c] = '#';                             // ⭐ mark, recurse, UNMARK
        bool found = dfs(r-1,c,i+1) || dfs(r+1,c,i+1)
                  || dfs(r,c-1,i+1) || dfs(r,c+1,i+1);
        b[r][c] = tmp;                             // backtrack
        return found;
    };
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c) if (dfs(r, c, 0)) return true;
    return false;
}
```
**Key insight:** Unlike island-counting, this is **backtracking** — the mark must be undone so other paths can reuse the cell.

---

### 8. Word Search II (Trie + backtracking) 🔴
```cpp
struct TrieNode {
    TrieNode* child[26] = {};
    string word;                                   // ⭐ store the full word at the end
};

vector<string> findWords(vector<vector<char>>& b, vector<string>& words) {
    TrieNode* root = new TrieNode();
    for (auto& w : words) {
        TrieNode* cur = root;
        for (char c : w) {
            if (!cur->child[c-'a']) cur->child[c-'a'] = new TrieNode();
            cur = cur->child[c-'a'];
        }
        cur->word = w;
    }

    int R = b.size(), C = b[0].size();
    vector<string> out;
    function<void(int,int,TrieNode*)> dfs = [&](int r, int c, TrieNode* node) {
        if (r < 0 || r >= R || c < 0 || c >= C || b[r][c] == '#') return;
        char ch = b[r][c];
        TrieNode* nxt = node->child[ch-'a'];
        if (!nxt) return;                          // ⭐ prune: no word has this prefix

        if (!nxt->word.empty()) { out.push_back(nxt->word); nxt->word.clear(); }  // dedupe

        b[r][c] = '#';
        for (int d = 0; d < 4; ++d) dfs(r+dr[d], c+dc[d], nxt);
        b[r][c] = ch;
    };
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c) dfs(r, c, root);
    return out;
}
```
**Key insight:** Searching each word independently is O(words × R × C × 4^L). The trie searches **all words simultaneously** and prunes the instant no word shares the current prefix.

---

### 9. Number of Distinct Islands 🟡
```cpp
int numDistinctIslands(vector<vector<int>>& g) {
    int R = g.size(), C = g[0].size();
    unordered_set<string> shapes;
    function<void(int,int,string&,char)> dfs = [&](int r, int c, string& path, char dir) {
        if (r < 0 || r >= R || c < 0 || c >= C || g[r][c] != 1) return;
        g[r][c] = 0;
        path += dir;
        dfs(r-1,c,path,'U'); dfs(r+1,c,path,'D');
        dfs(r,c-1,path,'L'); dfs(r,c+1,path,'R');
        path += 'B';                               // ⭐ backtrack marker
    };
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c)
            if (g[r][c] == 1) { string p; dfs(r, c, p, 'S'); shapes.insert(p); }
    return shapes.size();
}
```
⚠️ The `'B'` backtrack marker is essential — without it, differently-shaped islands can produce the same path string.

---

### 10. Shortest Path in Binary Matrix 🟡
```cpp
int shortestPathBinaryMatrix(vector<vector<int>>& g) {
    int n = g.size();
    if (g[0][0] || g[n-1][n-1]) return -1;
    queue<pair<int,int>> q{{{0,0}}};
    g[0][0] = 1;                                   // mark visited
    int dist = 1;
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            auto [r, c] = q.front(); q.pop();
            if (r == n-1 && c == n-1) return dist;
            for (int dr2 = -1; dr2 <= 1; ++dr2)    // ⭐ 8-directional
                for (int dc2 = -1; dc2 <= 1; ++dc2) {
                    int nr = r + dr2, nc = c + dc2;
                    if (nr < 0 || nr >= n || nc < 0 || nc >= n || g[nr][nc]) continue;
                    g[nr][nc] = 1;
                    q.push({nr, nc});
                }
        }
        ++dist;
    }
    return -1;
}
```

---

### 11. Walls and Gates 🟡
```cpp
void wallsAndGates(vector<vector<int>>& rooms) {
    int R = rooms.size(), C = rooms[0].size();
    queue<pair<int,int>> q;
    for (int r = 0; r < R; ++r)
        for (int c = 0; c < C; ++c)
            if (rooms[r][c] == 0) q.push({r, c});  // multi-source from all gates

    while (!q.empty()) {
        auto [r, c] = q.front(); q.pop();
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= R || nc < 0 || nc >= C || rooms[nr][nc] != INT_MAX) continue;
            rooms[nr][nc] = rooms[r][c] + 1;
            q.push({nr, nc});
        }
    }
}
```

---

### 12. Number of Enclaves / Closed Islands 🟡
```cpp
int numEnclaves(vector<vector<int>>& g) {
    int R = g.size(), C = g[0].size();
    function<void(int,int)> sink = [&](int r, int c) {
        if (r < 0 || r >= R || c < 0 || c >= C || g[r][c] != 1) return;
        g[r][c] = 0;
        for (int d = 0; d < 4; ++d) sink(r+dr[d], c+dc[d]);
    };
    for (int r = 0; r < R; ++r) { sink(r, 0); sink(r, C-1); }
    for (int c = 0; c < C; ++c) { sink(0, c); sink(R-1, c); }

    int cnt = 0;
    for (auto& row : g) for (int x : row) cnt += x;
    return cnt;
}
```

---

## B. BFS Shortest Path

### 13. Word Ladder 🔴
```cpp
int ladderLength(string begin, string end, vector<string>& wordList) {
    unordered_set<string> dict(wordList.begin(), wordList.end());
    if (!dict.count(end)) return 0;

    queue<string> q{{begin}};
    dict.erase(begin);
    int steps = 1;
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            string w = q.front(); q.pop();
            if (w == end) return steps;
            for (int j = 0; j < (int)w.size(); ++j) {      // ⭐ generate neighbors
                char orig = w[j];
                for (char c = 'a'; c <= 'z'; ++c) {
                    w[j] = c;
                    if (dict.erase(w)) q.push(w);          // erase = mark visited
                }
                w[j] = orig;
            }
        }
        ++steps;
    }
    return 0;
}
```
**Complexity:** O(N · L · 26).
**Key insight:** The graph is **implicit** — you generate neighbors on the fly rather than building an adjacency list. Generating all 26·L variants is far cheaper than comparing every pair of words (O(N²·L)).

🎤 **Follow-up:** bidirectional BFS from both ends roughly halves the search depth, turning b^d into 2·b^(d/2).

---

### 14. Word Ladder II (all shortest paths) 🔴
```cpp
vector<vector<string>> findLadders(string begin, string end, vector<string>& wordList) {
    unordered_set<string> dict(wordList.begin(), wordList.end());
    if (!dict.count(end)) return {};

    unordered_map<string, vector<string>> parents;
    unordered_set<string> level{begin};
    dict.erase(begin);
    bool found = false;

    while (!level.empty() && !found) {
        unordered_set<string> next;
        for (const string& w : level) dict.erase(w);       // ⭐ erase the whole level
        for (const string& w : level) {
            string cur = w;
            for (int j = 0; j < (int)cur.size(); ++j) {
                char orig = cur[j];
                for (char c = 'a'; c <= 'z'; ++c) {
                    cur[j] = c;
                    if (!dict.count(cur)) continue;
                    next.insert(cur);
                    parents[cur].push_back(w);             // record ALL parents
                    if (cur == end) found = true;
                }
                cur[j] = orig;
            }
        }
        level = move(next);
    }

    vector<vector<string>> out;
    vector<string> path{end};
    function<void(const string&)> backtrack = [&](const string& w) {
        if (w == begin) { vector<string> p(path.rbegin(), path.rend()); out.push_back(p); return; }
        for (const string& p : parents[w]) { path.push_back(p); backtrack(p); path.pop_back(); }
    };
    if (found) backtrack(end);
    return out;
}
```
**Key insight:** Erase visited words **per level**, not per node — otherwise you lose alternate shortest paths that arrive at the same depth.

---

### 15. Open the Lock 🟡
```cpp
int openLock(vector<string>& deadends, string target) {
    unordered_set<string> dead(deadends.begin(), deadends.end());
    if (dead.count("0000")) return -1;
    if (target == "0000") return 0;

    queue<string> q{{"0000"}};
    unordered_set<string> seen{"0000"};
    int steps = 0;
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            string s = q.front(); q.pop();
            if (s == target) return steps;
            for (int j = 0; j < 4; ++j) {
                for (int d : {1, -1}) {
                    string nxt = s;
                    nxt[j] = '0' + ((nxt[j] - '0' + d + 10) % 10);   // ⭐ wrap
                    if (!dead.count(nxt) && seen.insert(nxt).second) q.push(nxt);
                }
            }
        }
        ++steps;
    }
    return -1;
}
```

---

### 16. Minimum Genetic Mutation 🟡
```cpp
int minMutation(string start, string end, vector<string>& bank) {
    unordered_set<string> dict(bank.begin(), bank.end());
    if (!dict.count(end)) return -1;
    queue<string> q{{start}};
    int steps = 0;
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            string s = q.front(); q.pop();
            if (s == end) return steps;
            for (int j = 0; j < (int)s.size(); ++j) {
                char orig = s[j];
                for (char c : {'A','C','G','T'}) {
                    s[j] = c;
                    if (dict.erase(s)) q.push(s);
                }
                s[j] = orig;
            }
        }
        ++steps;
    }
    return -1;
}
```

---

### 17. Jump Game III / IV 🟡
```cpp
int minJumps(vector<int>& a) {                     // Jump Game IV
    int n = a.size();
    unordered_map<int, vector<int>> byValue;
    for (int i = 0; i < n; ++i) byValue[a[i]].push_back(i);

    queue<int> q{{0}};
    vector<bool> seen(n, false);
    seen[0] = true;
    int steps = 0;
    while (!q.empty()) {
        int sz = q.size();
        for (int k = 0; k < sz; ++k) {
            int i = q.front(); q.pop();
            if (i == n - 1) return steps;

            for (int j : byValue[a[i]])            // teleport to equal values
                if (!seen[j]) { seen[j] = true; q.push(j); }
            byValue[a[i]].clear();                 // ⭐ CRITICAL: avoid O(n²)

            for (int j : {i - 1, i + 1})
                if (j >= 0 && j < n && !seen[j]) { seen[j] = true; q.push(j); }
        }
        ++steps;
    }
    return -1;
}
```
⚠️ Clearing the value bucket after use is what keeps this O(n). Without it, a value repeated n times gives O(n²).

---

### 18. Shortest Path with Obstacles Elimination 🔴
```cpp
int shortestPath(vector<vector<int>>& g, int k) {
    int R = g.size(), C = g[0].size();
    if (k >= R + C - 2) return R + C - 2;          // enough to go straight

    vector<vector<int>> best(R, vector<int>(C, -1));  // max remaining k seen
    queue<tuple<int,int,int>> q{{{0,0,k}}};        // ⭐ STATE includes k
    best[0][0] = k;
    int steps = 0;
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            auto [r, c, rem] = q.front(); q.pop();
            if (r == R-1 && c == C-1) return steps;
            for (int d = 0; d < 4; ++d) {
                int nr = r + dr[d], nc = c + dc[d];
                if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
                int nrem = rem - g[nr][nc];
                if (nrem < 0 || best[nr][nc] >= nrem) continue;   // ⭐ dominated
                best[nr][nc] = nrem;
                q.push({nr, nc, nrem});
            }
        }
        ++steps;
    }
    return -1;
}
```
**Key insight:** **The state is (position, resource remaining)**, not just position. Reaching a cell with more eliminations left is strictly better, so a lower-`rem` visit can be pruned.

---

## C. Topological Sort

### 19. Course Schedule 🟡
> `n` courses, and a list of prerequisite pairs `[a, b]` meaning "you must take `b` before `a`". Can you finish all courses?

#### 💬 Think of it like this
Picture a university course catalogue. Some courses are gateways — you can take them right now because they have no prerequisites. Everything else is blocked behind something.

The natural strategy is exactly how a real student plans:
1. Find every course you can take **right now** (nothing blocking it).
2. Take them. That "unlocks" some other courses — each one loses a prerequisite.
3. Any course whose last prerequisite just got satisfied becomes available. Add it to your list.
4. Repeat until nothing is left available.

At the end, if you managed to schedule all `n` courses, you're done. If some courses were **never** unlocked, they must be stuck in a circular dependency — A needs B, B needs C, C needs A. Nobody can ever start.

The number tracking "how many prerequisites am I still waiting on" is called **in-degree**. This whole procedure is **Kahn's algorithm** for topological sort.

#### 📊 Tracing it

```
   Courses: 0,1,2,3     Prerequisites:  1←0   2←0   3←1   3←2
                        (arrow means "unlocks")

           ┌───┐
           │ 0 │  in-degree 0  ← can take immediately
           └─┬─┘
        ┌────┴────┐
        ▼         ▼
      ┌───┐     ┌───┐
      │ 1 │     │ 2 │   in-degree 1 each
      └─┬─┘     └─┬─┘
        └────┬────┘
             ▼
           ┌───┐
           │ 3 │  in-degree 2
           └───┘

   STEP 1  in-degrees: [0]=0  [1]=1  [2]=1  [3]=2
           queue = [0]                      taken = 0

   STEP 2  take 0 → decrement its targets
           in-degrees: [1]=0  [2]=0  [3]=2
           queue = [1, 2]                   taken = 1

   STEP 3  take 1 → decrement 3
           in-degrees: [2]=0  [3]=1
           queue = [2]                      taken = 2

   STEP 4  take 2 → decrement 3 → now 0!
           in-degrees: [3]=0
           queue = [3]                      taken = 3

   STEP 5  take 3                           taken = 4 ✅

           taken == n  →  return true
```

#### What a cycle looks like

```
      ┌───┐        ┌───┐
      │ 0 │───────▶│ 1 │        Nobody has in-degree 0.
      └───┘        └───┘        The queue starts EMPTY.
        ▲            │          taken = 0, but n = 2.
        └────────────┘          0 != 2  →  return false ❌
```

```cpp
bool canFinish(int n, vector<vector<int>>& pre) {
    vector<vector<int>> adj(n);
    vector<int> indeg(n, 0);
    for (auto& p : pre) { adj[p[1]].push_back(p[0]); ++indeg[p[0]]; }

    queue<int> q;
    for (int i = 0; i < n; ++i) if (!indeg[i]) q.push(i);

    int done = 0;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        ++done;
        for (int v : adj[u]) if (--indeg[v] == 0) q.push(v);
    }
    return done == n;                              // ⭐ fewer than n → cycle
}
```

---

### 20. Course Schedule II 🟡
```cpp
vector<int> findOrder(int n, vector<vector<int>>& pre) {
    vector<vector<int>> adj(n);
    vector<int> indeg(n, 0);
    for (auto& p : pre) { adj[p[1]].push_back(p[0]); ++indeg[p[0]]; }

    queue<int> q;
    for (int i = 0; i < n; ++i) if (!indeg[i]) q.push(i);

    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u]) if (--indeg[v] == 0) q.push(v);
    }
    return (int)order.size() == n ? order : vector<int>{};
}
```

---

### 21. Alien Dictionary 🔴
```cpp
string alienOrder(vector<string>& words) {
    unordered_map<char, unordered_set<char>> adj;
    unordered_map<char,int> indeg;
    for (auto& w : words) for (char c : w) indeg[c] = 0;   // ⭐ register all chars

    for (int i = 0; i + 1 < (int)words.size(); ++i) {
        const string &a = words[i], &b = words[i+1];
        int len = min(a.size(), b.size());
        // ⚠️ INVALID: a longer word cannot precede its own prefix
        if (a.size() > b.size() && a.compare(0, len, b) == 0) return "";
        for (int j = 0; j < len; ++j) {
            if (a[j] != b[j]) {
                if (adj[a[j]].insert(b[j]).second) ++indeg[b[j]];
                break;                             // ⭐ only the FIRST difference
            }
        }
    }

    queue<char> q;
    for (auto& [c, d] : indeg) if (!d) q.push(c);
    string out;
    while (!q.empty()) {
        char u = q.front(); q.pop();
        out += u;
        for (char v : adj[u]) if (--indeg[v] == 0) q.push(v);
    }
    return out.size() == indeg.size() ? out : "";  // cycle → invalid
}
```
**Key insight:** Only the *first* differing character between adjacent words gives ordering information. The prefix check is the edge case most people miss.

---

### 22. Minimum Height Trees 🟡
```cpp
vector<int> findMinHeightTrees(int n, vector<vector<int>>& edges) {
    if (n == 1) return {0};
    vector<unordered_set<int>> adj(n);
    for (auto& e : edges) { adj[e[0]].insert(e[1]); adj[e[1]].insert(e[0]); }

    vector<int> leaves;
    for (int i = 0; i < n; ++i) if (adj[i].size() == 1) leaves.push_back(i);

    int remaining = n;
    while (remaining > 2) {                        // ⭐ at most 2 centroids
        remaining -= leaves.size();
        vector<int> next;
        for (int leaf : leaves) {
            int nb = *adj[leaf].begin();
            adj[nb].erase(leaf);
            if (adj[nb].size() == 1) next.push_back(nb);
        }
        leaves = move(next);
    }
    return leaves;
}
```
**Key insight:** Peel leaves layer by layer, like topological sort on an undirected tree. The last one or two remaining nodes are the centroids.

---

### 23. Sequence Reconstruction 🟡
```cpp
bool sequenceReconstruction(vector<int>& nums, vector<vector<int>>& seqs) {
    int n = nums.size();
    vector<unordered_set<int>> adj(n + 1);
    vector<int> indeg(n + 1, 0);
    unordered_set<int> seen;

    for (auto& s : seqs)
        for (int i = 0; i < (int)s.size(); ++i) {
            if (s[i] < 1 || s[i] > n) return false;
            seen.insert(s[i]);
            if (i && adj[s[i-1]].insert(s[i]).second) ++indeg[s[i]];
        }
    if ((int)seen.size() != n) return false;

    queue<int> q;
    for (int i = 1; i <= n; ++i) if (!indeg[i]) q.push(i);

    int idx = 0;
    while (!q.empty()) {
        if (q.size() > 1) return false;            // ⭐ ambiguous → not unique
        int u = q.front(); q.pop();
        if (nums[idx++] != u) return false;
        for (int v : adj[u]) if (--indeg[v] == 0) q.push(v);
    }
    return idx == n;
}
```
**Key insight:** A **unique** topological order requires exactly one node with in-degree zero at every step.

---

### 24. Parallel Courses 🟡
```cpp
int minimumSemesters(int n, vector<vector<int>>& relations) {
    vector<vector<int>> adj(n + 1);
    vector<int> indeg(n + 1, 0);
    for (auto& r : relations) { adj[r[0]].push_back(r[1]); ++indeg[r[1]]; }

    queue<int> q;
    for (int i = 1; i <= n; ++i) if (!indeg[i]) q.push(i);

    int semesters = 0, done = 0;
    while (!q.empty()) {
        int sz = q.size();                         // ⭐ a whole level per semester
        for (int i = 0; i < sz; ++i) {
            int u = q.front(); q.pop();
            ++done;
            for (int v : adj[u]) if (--indeg[v] == 0) q.push(v);
        }
        ++semesters;
    }
    return done == n ? semesters : -1;
}
```

---

## D. Union-Find

#### 💬 What Union-Find actually is
Imagine a room of people forming friend groups. You need to answer two questions very fast, over and over:
- *"Are these two people in the same group?"*
- *"Merge these two groups."*

Union-Find (a.k.a. Disjoint Set Union, DSU) does this by giving every group a **leader**. To check if two people are in the same group, find each one's leader and compare. To merge, point one leader at the other.

The naive version degenerates into long chains. Two optimizations fix it:

**Path compression** — after finding your leader, everyone you walked past gets re-pointed *directly* at the leader, so next time is instant.

```
   BEFORE find(4)              AFTER find(4)

     1  ← leader                 1  ← leader
     ▲                         ▲ ▲ ▲
     2                         │ │ │
     ▲                         2 3 4    everyone points straight to the top
     3
     ▲
     4
```

**Union by rank** — always attach the *shorter* tree under the taller one, so trees stay shallow.

```
   merging these two:        ❌ bad (deepens)      ✅ good (stays flat)

     1        5                  5                    1
     ▲       ▲ ▲                 ▲                   ▲ ▲ ▲
     2       6 7                 1                   2 5 ...
                                 ▲                     ▲ ▲
                                 2                     6 7
```

Together these make every operation effectively **O(1)** (formally O(α(n)), where α is the inverse Ackermann function — under 5 for any input that fits in the universe).

**When to use DSU instead of DFS:** when connectivity changes **dynamically**. If edges arrive one at a time and you must answer questions in between, DFS would mean re-scanning the whole graph after every addition. DSU absorbs each edge in constant time.

```cpp
struct DSU {
    vector<int> p, r;
    int comps;
    DSU(int n) : p(n), r(n, 0), comps(n) { iota(p.begin(), p.end(), 0); }
    int find(int x) { return p[x] == x ? x : p[x] = find(p[x]); }
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (r[a] < r[b]) swap(a, b);
        p[b] = a;
        if (r[a] == r[b]) ++r[a];
        --comps;
        return true;
    }
};
```

### 25. Number of Connected Components 🟡
```cpp
int countComponents(int n, vector<vector<int>>& edges) {
    DSU dsu(n);
    for (auto& e : edges) dsu.unite(e[0], e[1]);
    return dsu.comps;
}
```

---

### 26. Graph Valid Tree 🟡
```cpp
bool validTree(int n, vector<vector<int>>& edges) {
    if ((int)edges.size() != n - 1) return false;  // ⭐ a tree has exactly n-1 edges
    DSU dsu(n);
    for (auto& e : edges) if (!dsu.unite(e[0], e[1])) return false;  // cycle
    return true;
}
```
**Key insight:** `n-1` edges plus no cycle implies connected. Checking both properties separately is unnecessary.

---

### 27. Redundant Connection 🟡
```cpp
vector<int> findRedundantConnection(vector<vector<int>>& edges) {
    DSU dsu(edges.size() + 1);
    for (auto& e : edges) if (!dsu.unite(e[0], e[1])) return e;   // first cycle edge
    return {};
}
```

---

### 28. Accounts Merge 🟡
```cpp
vector<vector<string>> accountsMerge(vector<vector<string>>& accounts) {
    DSU dsu(accounts.size());
    unordered_map<string,int> owner;                // email -> account index
    for (int i = 0; i < (int)accounts.size(); ++i)
        for (int j = 1; j < (int)accounts[i].size(); ++j) {
            const string& e = accounts[i][j];
            auto it = owner.find(e);
            if (it != owner.end()) dsu.unite(i, it->second);   // ⭐ shared email
            else owner[e] = i;
        }

    unordered_map<int, set<string>> merged;         // set → sorted output
    for (auto& [e, i] : owner) merged[dsu.find(i)].insert(e);

    vector<vector<string>> out;
    for (auto& [root, emails] : merged) {
        vector<string> acc{accounts[root][0]};
        acc.insert(acc.end(), emails.begin(), emails.end());
        out.push_back(move(acc));
    }
    return out;
}
```

---

### 29. Number of Islands II (dynamic) 🔴
```cpp
vector<int> numIslands2(int m, int n, vector<vector<int>>& positions) {
    DSU dsu(m * n);
    vector<bool> isLand(m * n, false);
    int count = 0;
    vector<int> out;

    for (auto& p : positions) {
        int r = p[0], c = p[1], id = r * n + c;
        if (isLand[id]) { out.push_back(count); continue; }   // duplicate
        isLand[id] = true;
        ++count;
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int nid = nr * n + nc;
            if (isLand[nid] && dsu.unite(id, nid)) --count;    // ⭐ merged two islands
        }
        out.push_back(count);
    }
    return out;
}
```
**Key insight:** This is exactly what DSU is for — **dynamic** connectivity, where DFS would require re-scanning the whole grid after every addition.

---

### 30. Most Stones Removed 🟡
```cpp
int removeStones(vector<vector<int>>& stones) {
    DSU dsu(stones.size());
    unordered_map<int,int> rowFirst, colFirst;
    for (int i = 0; i < (int)stones.size(); ++i) {
        int r = stones[i][0], c = stones[i][1];
        if (rowFirst.count(r)) dsu.unite(i, rowFirst[r]); else rowFirst[r] = i;
        if (colFirst.count(c)) dsu.unite(i, colFirst[c]); else colFirst[c] = i;
    }
    return stones.size() - dsu.comps;              // ⭐ each component leaves 1 stone
}
```

---

### 31. Satisfiability of Equality Equations 🟡
```cpp
bool equationsPossible(vector<string>& equations) {
    DSU dsu(26);
    for (auto& e : equations)
        if (e[1] == '=') dsu.unite(e[0]-'a', e[3]-'a');       // ⭐ process == FIRST
    for (auto& e : equations)
        if (e[1] == '!' && dsu.find(e[0]-'a') == dsu.find(e[3]-'a')) return false;
    return true;
}
```

---

### 32. Evaluate Division 🟡
```cpp
vector<double> calcEquation(vector<vector<string>>& eq, vector<double>& vals,
                            vector<vector<string>>& queries) {
    unordered_map<string, vector<pair<string,double>>> adj;
    for (int i = 0; i < (int)eq.size(); ++i) {
        adj[eq[i][0]].push_back({eq[i][1], vals[i]});
        adj[eq[i][1]].push_back({eq[i][0], 1.0 / vals[i]});   // ⭐ weighted, both ways
    }

    vector<double> out;
    for (auto& q : queries) {
        if (!adj.count(q[0]) || !adj.count(q[1])) { out.push_back(-1.0); continue; }
        unordered_set<string> seen;
        function<double(const string&, const string&, double)> dfs =
            [&](const string& cur, const string& target, double prod) -> double {
            if (cur == target) return prod;
            seen.insert(cur);
            for (auto& [nb, w] : adj[cur])
                if (!seen.count(nb)) {
                    double r = dfs(nb, target, prod * w);
                    if (r > 0) return r;
                }
            return -1.0;
        };
        out.push_back(dfs(q[0], q[1], 1.0));
    }
    return out;
}
```
**Key insight:** A weighted graph where the path product is the answer. Weighted DSU also solves this in near-O(1) per query.

---

## E. Shortest Path (Weighted)

### 33. Network Delay Time (Dijkstra) 🟡
> A network of `n` nodes. `times[i] = [u, v, w]` means a signal from `u` reaches `v` in `w` time. Starting from node `k`, how long until **all** nodes receive it? Return `-1` if some node is unreachable.

#### 💬 Think of it like this
BFS finds shortest paths when every edge costs the same — count the hops. But here edges have **different costs**, so "fewest hops" is no longer "fastest." A 3-hop route down cheap edges can beat a 1-hop expensive one.

Dijkstra fixes this with a simple greedy rule: **always expand from the closest unfinished node.**

Concretely, keep a "best known time" for every node, all starting at infinity except the source at 0. Then repeatedly pull out the node with the smallest known time. Because every edge cost is non-negative, once you pull a node out you can be *certain* its time is final — no route through a node further away could ever come back and beat it.

From that node, try to improve its neighbours ("relax" them), and put the improved ones back in the priority queue.

#### 📊 Tracing it

```
   Graph, source = 1

        1 ──2──▶ 2 ──1──▶ 3
        │                 ▲
        └───────5─────────┘

   dist = [_, 0, ∞, ∞]      heap = {(0,node1)}

   ─── pop (0, node1) ──────────────────────────────
   relax 1→2 cost 2:  0 + 2 = 2  <  ∞   ✅ dist[2]=2, push (2,node2)
   relax 1→3 cost 5:  0 + 5 = 5  <  ∞   ✅ dist[3]=5, push (5,node3)

   dist = [_, 0, 2, 5]      heap = {(2,n2), (5,n3)}

   ─── pop (2, node2) ──────────────────────────────
   relax 2→3 cost 1:  2 + 1 = 3  <  5   ✅ dist[3]=3, push (3,node3)

   dist = [_, 0, 2, 3]      heap = {(3,n3), (5,n3)}
                                            └── STALE! ignore it later

   ─── pop (3, node3) ──────────────────────────────
   3 == dist[3] → fresh, process (no outgoing edges)

   ─── pop (5, node3) ──────────────────────────────
   5 > dist[3] which is 3  →  ⭐ STALE, skip it

   Answer = max(0, 2, 3) = 3
```

#### Why the "stale entry" check exists

```
   C++ priority_queue CANNOT update a value already inside it.

   So when we find a better route to node 3, we don't edit the
   old (5, node3) entry — we just push a new (3, node3).

   heap now holds BOTH:   (3, node3)   (5, node3)
                           ▲            ▲
                        the truth    outdated garbage

   The line `if (d > dist[u]) continue;` throws away the garbage
   when it eventually surfaces. This is called LAZY DELETION.
```

⚠️ **Dijkstra breaks with negative edges.** The whole guarantee rests on "once popped, it's final" — a negative edge could later reduce a finalized distance, and Dijkstra never revisits. Use Bellman-Ford instead.

```cpp
int networkDelayTime(vector<vector<int>>& times, int n, int k) {
    vector<vector<pair<int,int>>> adj(n + 1);
    for (auto& t : times) adj[t[0]].push_back({t[1], t[2]});

    vector<int> dist(n + 1, INT_MAX);
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    dist[k] = 0;
    pq.push({0, k});

    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;                 // ⭐ stale entry, skip
        for (auto& [v, w] : adj[u])
            if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
    }

    int mx = 0;
    for (int i = 1; i <= n; ++i) {
        if (dist[i] == INT_MAX) return -1;
        mx = max(mx, dist[i]);
    }
    return mx;
}
```
**Key insight:** The `if (d > dist[u]) continue` line implements **lazy deletion** — C++'s priority queue can't update keys, so you push duplicates and skip outdated ones.

---

### 34. Cheapest Flights Within K Stops 🟡
```cpp
int findCheapestPrice(int n, vector<vector<int>>& flights, int src, int dst, int k) {
    vector<int> dist(n, INT_MAX);
    dist[src] = 0;
    for (int i = 0; i <= k; ++i) {                 // ⭐ Bellman-Ford, k+1 relaxations
        vector<int> tmp = dist;                    // ⭐ MUST use the previous round
        for (auto& f : flights) {
            if (dist[f[0]] == INT_MAX) continue;
            tmp[f[1]] = min(tmp[f[1]], dist[f[0]] + f[2]);
        }
        dist = move(tmp);
    }
    return dist[dst] == INT_MAX ? -1 : dist[dst];
}
```
⚠️ **The `tmp` copy is essential.** Relaxing in place would let a path use more than `i+1` edges in round `i`, breaking the stop limit. This is the single most common bug in this problem.

---

### 35. Path with Maximum Probability 🟡
```cpp
double maxProbability(int n, vector<vector<int>>& edges, vector<double>& prob,
                      int start, int end) {
    vector<vector<pair<int,double>>> adj(n);
    for (int i = 0; i < (int)edges.size(); ++i) {
        adj[edges[i][0]].push_back({edges[i][1], prob[i]});
        adj[edges[i][1]].push_back({edges[i][0], prob[i]});
    }

    vector<double> best(n, 0.0);
    priority_queue<pair<double,int>> pq;           // ⭐ MAX-heap for probability
    best[start] = 1.0;
    pq.push({1.0, start});

    while (!pq.empty()) {
        auto [p, u] = pq.top(); pq.pop();
        if (u == end) return p;
        if (p < best[u]) continue;
        for (auto& [v, w] : adj[u])
            if (p * w > best[v]) { best[v] = p * w; pq.push({best[v], v}); }
    }
    return 0.0;
}
```
**Key insight:** Dijkstra works for any *monotonic* path metric, not just additive distance — here it's multiplicative probability, maximized.

---

### 36. Path with Minimum Effort 🟡
```cpp
int minimumEffortPath(vector<vector<int>>& h) {
    int R = h.size(), C = h[0].size();
    vector<vector<int>> effort(R, vector<int>(C, INT_MAX));
    priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<>> pq;
    effort[0][0] = 0;
    pq.push({0, 0, 0});

    while (!pq.empty()) {
        auto [e, r, c] = pq.top(); pq.pop();
        if (r == R-1 && c == C-1) return e;
        if (e > effort[r][c]) continue;
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
            int ne = max(e, abs(h[nr][nc] - h[r][c]));   // ⭐ MINIMAX, not sum
            if (ne < effort[nr][nc]) { effort[nr][nc] = ne; pq.push({ne, nr, nc}); }
        }
    }
    return 0;
}
```
**Key insight:** "Minimize the maximum edge on the path" — Dijkstra with `max` replacing `+`. Binary search on the answer plus a connectivity check also works.

---

### 37. Swim in Rising Water 🔴
```cpp
int swimInWater(vector<vector<int>>& g) {
    int n = g.size();
    vector<vector<bool>> seen(n, vector<bool>(n, false));
    priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<>> pq;
    pq.push({g[0][0], 0, 0});
    seen[0][0] = true;

    while (!pq.empty()) {
        auto [t, r, c] = pq.top(); pq.pop();
        if (r == n-1 && c == n-1) return t;
        for (int d = 0; d < 4; ++d) {
            int nr = r + dr[d], nc = c + dc[d];
            if (nr < 0 || nr >= n || nc < 0 || nc >= n || seen[nr][nc]) continue;
            seen[nr][nc] = true;
            pq.push({max(t, g[nr][nc]), nr, nc});  // ⭐ minimax again
        }
    }
    return -1;
}
```

---

### 38. Bellman-Ford / Negative Cycle Detection 🔴
```cpp
bool hasNegativeCycle(int n, vector<vector<int>>& edges) {
    vector<long long> dist(n, 0);                  // 0 = virtual source to all
    for (int i = 0; i < n - 1; ++i)
        for (auto& e : edges)
            dist[e[1]] = min(dist[e[1]], dist[e[0]] + e[2]);

    for (auto& e : edges)                          // ⭐ one extra pass
        if (dist[e[0]] + e[2] < dist[e[1]]) return true;   // still improving → cycle
    return false;
}
```
**Key insight:** After `V-1` relaxations every shortest path is final (a simple path has at most `V-1` edges). Any further improvement proves a negative cycle.

---

### 39. Floyd-Warshall — City With Smallest Number of Neighbors 🟡
```cpp
int findTheCity(int n, vector<vector<int>>& edges, int threshold) {
    const int INF = 1e9;
    vector<vector<int>> d(n, vector<int>(n, INF));
    for (int i = 0; i < n; ++i) d[i][i] = 0;
    for (auto& e : edges) d[e[0]][e[1]] = d[e[1]][e[0]] = e[2];

    for (int k = 0; k < n; ++k)                    // ⭐ k MUST be the outer loop
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                d[i][j] = min(d[i][j], d[i][k] + d[k][j]);

    int best = -1, bestCount = n + 1;
    for (int i = 0; i < n; ++i) {
        int cnt = 0;
        for (int j = 0; j < n; ++j) if (i != j && d[i][j] <= threshold) ++cnt;
        if (cnt <= bestCount) { bestCount = cnt; best = i; }
    }
    return best;
}
```
⚠️ `k` outer is mandatory. The invariant is "after iteration k, `d[i][j]` uses only intermediates from `{0..k}`" — reordering the loops breaks it.

---

## F. MST & Advanced

### 40. Min Cost to Connect All Points (Prim) 🟡
```cpp
int minCostConnectPoints(vector<vector<int>>& pts) {
    int n = pts.size(), total = 0, visited = 0;
    vector<bool> inMST(n, false);
    vector<int> minCost(n, INT_MAX);
    minCost[0] = 0;

    for (int it = 0; it < n; ++it) {
        int u = -1;
        for (int i = 0; i < n; ++i)                // O(V²) Prim — fine for dense
            if (!inMST[i] && (u == -1 || minCost[i] < minCost[u])) u = i;
        inMST[u] = true;
        total += minCost[u];
        for (int v = 0; v < n; ++v) {
            if (inMST[v]) continue;
            int w = abs(pts[u][0]-pts[v][0]) + abs(pts[u][1]-pts[v][1]);
            minCost[v] = min(minCost[v], w);
        }
    }
    return total;
}
```

---

### 41. Connecting Cities With Minimum Cost (Kruskal) 🟡
```cpp
int minimumCost(int n, vector<vector<int>>& connections) {
    sort(connections.begin(), connections.end(),
         [](auto& a, auto& b){ return a[2] < b[2]; });   // ⭐ cheapest first
    DSU dsu(n + 1);
    int total = 0, used = 0;
    for (auto& c : connections)
        if (dsu.unite(c[0], c[1])) { total += c[2]; if (++used == n - 1) break; }
    return used == n - 1 ? total : -1;
}
```
**Key insight:** Kruskal = sort edges + DSU. Prim = grow from one vertex with a heap. Kruskal is better for sparse graphs, Prim for dense.

---

### 42. Critical Connections (Tarjan's bridges) 🔴
```cpp
vector<vector<int>> criticalConnections(int n, vector<vector<int>>& connections) {
    vector<vector<int>> adj(n);
    for (auto& c : connections) { adj[c[0]].push_back(c[1]); adj[c[1]].push_back(c[0]); }

    vector<int> disc(n, -1), low(n, 0);
    vector<vector<int>> bridges;
    int timer = 0;

    function<void(int,int)> dfs = [&](int u, int parent) {
        disc[u] = low[u] = timer++;
        for (int v : adj[u]) {
            if (v == parent) continue;
            if (disc[v] == -1) {
                dfs(v, u);
                low[u] = min(low[u], low[v]);
                if (low[v] > disc[u]) bridges.push_back({u, v});  // ⭐ a bridge
            } else {
                low[u] = min(low[u], disc[v]);     // back edge
            }
        }
    };
    dfs(0, -1);
    return bridges;
}
```
**Key insight:** `low[v]` is the earliest discovery time reachable from `v`'s subtree via at most one back edge. If `low[v] > disc[u]`, nothing in `v`'s subtree can reach `u` or above except through edge `(u,v)` — so it's a bridge.

---

### 43. Clone Graph 🟡
```cpp
Node* cloneGraph(Node* node) {
    unordered_map<Node*, Node*> copies;
    function<Node*(Node*)> dfs = [&](Node* n) -> Node* {
        if (!n) return nullptr;
        auto it = copies.find(n);
        if (it != copies.end()) return it->second; // ⭐ already cloned
        Node* c = new Node(n->val);
        copies[n] = c;                             // ⭐ register BEFORE recursing
        for (Node* nb : n->neighbors) c->neighbors.push_back(dfs(nb));
        return c;
    };
    return dfs(node);
}
```
⚠️ Registering the copy *before* recursing is what prevents infinite recursion on cycles.

---

### 44. Course Schedule IV (reachability) 🟡
```cpp
vector<bool> checkIfPrerequisite(int n, vector<vector<int>>& pre, vector<vector<int>>& q) {
    vector<vector<bool>> reach(n, vector<bool>(n, false));
    for (auto& p : pre) reach[p[0]][p[1]] = true;

    for (int k = 0; k < n; ++k)                    // transitive closure
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                if (reach[i][k] && reach[k][j]) reach[i][j] = true;

    vector<bool> out;
    for (auto& x : q) out.push_back(reach[x[0]][x[1]]);
    return out;
}
```

---

### 45. Is Graph Bipartite? 🟡
```cpp
bool isBipartite(vector<vector<int>>& g) {
    int n = g.size();
    vector<int> color(n, 0);                       // 0 = uncolored, 1/-1 = two sides
    for (int s = 0; s < n; ++s) {
        if (color[s]) continue;
        queue<int> q{{s}};
        color[s] = 1;
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int v : g[u]) {
                if (!color[v]) { color[v] = -color[u]; q.push(v); }
                else if (color[v] == color[u]) return false;   // ⭐ odd cycle
            }
        }
    }
    return true;
}
```
**Key insight:** A graph is bipartite iff it has no odd-length cycle. Two-coloring via BFS detects exactly that. Note the outer loop — the graph may be disconnected.

---

### 46. Possible Bipartition 🟡
```cpp
bool possibleBipartition(int n, vector<vector<int>>& dislikes) {
    vector<vector<int>> adj(n + 1);
    for (auto& d : dislikes) { adj[d[0]].push_back(d[1]); adj[d[1]].push_back(d[0]); }
    vector<int> color(n + 1, 0);
    for (int s = 1; s <= n; ++s) {
        if (color[s]) continue;
        queue<int> q{{s}}; color[s] = 1;
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int v : adj[u]) {
                if (!color[v]) { color[v] = -color[u]; q.push(v); }
                else if (color[v] == color[u]) return false;
            }
        }
    }
    return true;
}
```

---

### 47. Find Eventual Safe States 🟡
```cpp
vector<int> eventualSafeNodes(vector<vector<int>>& g) {
    int n = g.size();
    vector<int> color(n, 0);                       // 0=white 1=gray(in path) 2=black(safe)
    function<bool(int)> safe = [&](int u) -> bool {
        if (color[u]) return color[u] == 2;
        color[u] = 1;                              // gray: on the current DFS path
        for (int v : g[u]) if (!safe(v)) return false;
        color[u] = 2;
        return true;
    };
    vector<int> out;
    for (int i = 0; i < n; ++i) if (safe(i)) out.push_back(i);
    return out;
}
```
**Key insight:** The three-color DFS — gray means "currently on the recursion stack," so encountering gray is a back edge and therefore a cycle.

---

### 48. Reconstruct Itinerary (Hierholzer / Eulerian path) 🔴
```cpp
vector<string> findItinerary(vector<vector<string>>& tickets) {
    unordered_map<string, multiset<string>> adj;   // multiset = lexical order
    for (auto& t : tickets) adj[t[0]].insert(t[1]);

    vector<string> route;
    function<void(const string&)> dfs = [&](const string& u) {
        auto& dests = adj[u];
        while (!dests.empty()) {
            string v = *dests.begin();
            dests.erase(dests.begin());            // ⭐ consume the edge
            dfs(v);
        }
        route.push_back(u);                        // ⭐ POST-order
    };
    dfs("JFK");
    reverse(route.begin(), route.end());
    return route;
}
```
**Key insight:** Hierholzer's algorithm. Adding to the route *after* exhausting a node's edges, then reversing, correctly handles dead ends — a greedy forward walk would get stuck.

---

### 49. Minimum Number of Vertices to Reach All Nodes 🟡
```cpp
vector<int> findSmallestSetOfVertices(int n, vector<vector<int>>& edges) {
    vector<bool> hasIncoming(n, false);
    for (auto& e : edges) hasIncoming[e[1]] = true;
    vector<int> out;
    for (int i = 0; i < n; ++i) if (!hasIncoming[i]) out.push_back(i);
    return out;
}
```
**Key insight:** A node with no incoming edge can only be reached by starting there. Nodes with incoming edges are reachable from those. So the answer is exactly the zero-in-degree set — no traversal needed.

---

### 50. Snakes and Ladders 🟡
```cpp
int snakesAndLadders(vector<vector<int>>& board) {
    int n = board.size();
    auto pos = [&](int s) {                        // ⭐ boustrophedon indexing
        int r = (s - 1) / n, c = (s - 1) % n;
        if (r % 2) c = n - 1 - c;                  // odd rows go right-to-left
        return pair<int,int>{n - 1 - r, c};
    };

    vector<bool> seen(n * n + 1, false);
    queue<int> q{{1}};
    seen[1] = true;
    int moves = 0;
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            int s = q.front(); q.pop();
            if (s == n * n) return moves;
            for (int d = 1; d <= 6 && s + d <= n * n; ++d) {
                auto [r, c] = pos(s + d);
                int nxt = board[r][c] == -1 ? s + d : board[r][c];
                if (!seen[nxt]) { seen[nxt] = true; q.push(nxt); }
            }
        }
        ++moves;
    }
    return -1;
}
```

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║                     GRAPHS — PATTERN RECALL                       ║
╠═══════════════════════════════════════════════════════════════════╣
║ RECOGNIZE THE GRAPH: grids · dependencies · word transforms ·     ║
║   state machines · equations · "can X reach Y"                    ║
╠═══════════════════════════════════════════════════════════════════╣
║ SHORTEST PATH                                                     ║
║   unweighted        → BFS (mark seen when ENQUEUEING)             ║
║   many sources      → MULTI-SOURCE BFS (seed the queue with all)  ║
║   weights ≥ 0       → Dijkstra (heap + `if d > dist[u] continue`) ║
║   negative weights  → Bellman-Ford (V-1 rounds, +1 detects cycle) ║
║   ≤ k edges         → Bellman-Ford with a COPY of dist per round  ║
║   all pairs, V≤500  → Floyd-Warshall (k MUST be the outer loop)   ║
║   minimize the MAX  → Dijkstra with max() instead of +            ║
║   state has resource→ include it in the visited key: (pos, k)     ║
╠═══════════════════════════════════════════════════════════════════╣
║ CONNECTIVITY                                                      ║
║   static     → DFS/BFS flood fill                                 ║
║   DYNAMIC    → Union-Find (this is what DSU is FOR)               ║
║   MST        → Kruskal (sort + DSU) or Prim (heap)                ║
║   bridges    → Tarjan low-link                                    ║
║   bipartite  → 2-color BFS; conflict = odd cycle                  ║
╠═══════════════════════════════════════════════════════════════════╣
║ ORDERING                                                          ║
║   topological → Kahn's (in-degree + queue); output size < n = CYCLE║
║   unique order → exactly one zero-in-degree node at each step     ║
║   levels/rounds → process a whole queue level per round           ║
║   cycle in DIRECTED → 3-color DFS (gray = on the current path)    ║
║   Eulerian path → Hierholzer, POST-order append then reverse      ║
╠═══════════════════════════════════════════════════════════════════╣
║ GOTCHAS                                                           ║
║   ⚠️ mark visited when ENQUEUEING, not dequeueing                  ║
║   ⚠️ grid DFS recursion can overflow at 10⁶ cells → use BFS        ║
║   ⚠️ backtracking (word search) must UNMARK; flood fill must not   ║
║   ⚠️ clone graph: register the copy BEFORE recursing               ║
║   ⚠️ disconnected graphs need an outer loop over all nodes         ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Dynamic Programming →](09-dynamic-programming.md)
