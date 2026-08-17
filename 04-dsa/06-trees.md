# 🌳 Trees & Binary Search Trees — 50 Problems

> Trees are recursion made visible. Nearly every tree problem is answered by one question: **what do I need from my children, and what do I return to my parent?**

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 The Universal Tree Template

```cpp
struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode(int x = 0, TreeNode* l = nullptr, TreeNode* r = nullptr)
        : val(x), left(l), right(r) {}
};
```

```cpp
ReturnType solve(TreeNode* node) {
    // 1. BASE CASE — what does an empty subtree contribute?
    if (!node) return identity;

    // 2. RECURSE — get answers from both children
    auto L = solve(node->left);
    auto R = solve(node->right);

    // 3. COMBINE — what do I tell my parent?
    return combine(node->val, L, R);
}
```

```
   TWO KINDS OF TREE PROBLEMS

   TOP-DOWN (pass information DOWN)     BOTTOM-UP (collect UP)
   ───────────────────────────────     ──────────────────────
   depth, path sums, validation         height, diameter, LCA,
   with bounds                          subtree properties
   → pass parameters into the call      → return values from the call

   ⭐ Many "hard" tree problems become easy by returning a STRUCT
     with several pieces of information at once.
```

### The four traversals

```
                1
              /   \
             2     3
            / \
           4   5

   PREORDER   (root, L, R)  →  1 2 4 5 3    "copy the tree, serialize"
   INORDER    (L, root, R)  →  4 2 5 1 3    "BST → sorted order"  ⭐
   POSTORDER  (L, R, root)  →  4 5 2 3 1    "delete, compute from children"
   LEVEL      (BFS)         →  1 2 3 4 5    "level-by-level, shortest path"
```

---

## A. Traversal

### 1. Binary Tree Inorder Traversal 🟢
```cpp
// Recursive
void inorder(TreeNode* n, vector<int>& out) {
    if (!n) return;
    inorder(n->left, out);
    out.push_back(n->val);
    inorder(n->right, out);
}

// Iterative — the pattern you should know cold
vector<int> inorderTraversal(TreeNode* root) {
    vector<int> out;
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }   // go as far left as possible
        cur = st.top(); st.pop();
        out.push_back(cur->val);                          // visit
        cur = cur->right;                                 // then go right
    }
    return out;
}
```

---

### 2. Preorder / Postorder Iterative 🟡
```cpp
vector<int> preorderTraversal(TreeNode* root) {
    if (!root) return {};
    vector<int> out;
    stack<TreeNode*> st{{root}};
    while (!st.empty()) {
        TreeNode* n = st.top(); st.pop();
        out.push_back(n->val);
        if (n->right) st.push(n->right);           // ⭐ right first → left pops first
        if (n->left)  st.push(n->left);
    }
    return out;
}

vector<int> postorderTraversal(TreeNode* root) {
    if (!root) return {};
    vector<int> out;
    stack<TreeNode*> st{{root}};
    while (!st.empty()) {
        TreeNode* n = st.top(); st.pop();
        out.push_back(n->val);
        if (n->left)  st.push(n->left);            // reversed preorder
        if (n->right) st.push(n->right);
    }
    reverse(out.begin(), out.end());               // ⭐ root-R-L reversed = L-R-root
    return out;
}
```

---

### 3. Morris Traversal (O(1) space) 🔴
```cpp
vector<int> morrisInorder(TreeNode* root) {
    vector<int> out;
    TreeNode* cur = root;
    while (cur) {
        if (!cur->left) { out.push_back(cur->val); cur = cur->right; }
        else {
            TreeNode* pred = cur->left;
            while (pred->right && pred->right != cur) pred = pred->right;

            if (!pred->right) { pred->right = cur; cur = cur->left; }   // create thread
            else { pred->right = nullptr; out.push_back(cur->val); cur = cur->right; }
        }                                          // ⭐ remove thread, restore tree
    }
    return out;
}
```
**Key insight:** Temporarily use null right pointers of predecessors as "threads" back to the ancestor, then remove them. O(1) space, O(n) time.

---

### 4. Level Order Traversal 🟡
```cpp
vector<vector<int>> levelOrder(TreeNode* root) {
    if (!root) return {};
    vector<vector<int>> out;
    queue<TreeNode*> q{{root}};
    while (!q.empty()) {
        int sz = q.size();                         // ⭐ freeze the level size
        vector<int> level;
        for (int i = 0; i < sz; ++i) {
            TreeNode* n = q.front(); q.pop();
            level.push_back(n->val);
            if (n->left)  q.push(n->left);
            if (n->right) q.push(n->right);
        }
        out.push_back(move(level));
    }
    return out;
}
```

---

### 5. Zigzag Level Order 🟡
```cpp
vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
    if (!root) return {};
    vector<vector<int>> out;
    queue<TreeNode*> q{{root}};
    bool ltr = true;
    while (!q.empty()) {
        int sz = q.size();
        vector<int> level(sz);
        for (int i = 0; i < sz; ++i) {
            TreeNode* n = q.front(); q.pop();
            level[ltr ? i : sz - 1 - i] = n->val;  // ⭐ write in place, no reverse
            if (n->left)  q.push(n->left);
            if (n->right) q.push(n->right);
        }
        out.push_back(move(level));
        ltr = !ltr;
    }
    return out;
}
```

---

### 6. Right Side View 🟡
```cpp
vector<int> rightSideView(TreeNode* root) {
    if (!root) return {};
    vector<int> out;
    queue<TreeNode*> q{{root}};
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            TreeNode* n = q.front(); q.pop();
            if (i == sz - 1) out.push_back(n->val);   // last of each level
            if (n->left)  q.push(n->left);
            if (n->right) q.push(n->right);
        }
    }
    return out;
}
```

---

### 7. Vertical Order Traversal 🔴
```cpp
vector<vector<int>> verticalTraversal(TreeNode* root) {
    map<int, map<int, multiset<int>>> m;           // col -> row -> values
    function<void(TreeNode*,int,int)> dfs = [&](TreeNode* n, int r, int c) {
        if (!n) return;
        m[c][r].insert(n->val);                    // ⭐ multiset sorts ties by value
        dfs(n->left, r + 1, c - 1);
        dfs(n->right, r + 1, c + 1);
    };
    dfs(root, 0, 0);

    vector<vector<int>> out;
    for (auto& [c, rows] : m) {
        vector<int> col;
        for (auto& [r, vals] : rows) col.insert(col.end(), vals.begin(), vals.end());
        out.push_back(move(col));
    }
    return out;
}
```

---

### 8. Average of Levels 🟢
```cpp
vector<double> averageOfLevels(TreeNode* root) {
    vector<double> out;
    queue<TreeNode*> q{{root}};
    while (!q.empty()) {
        int sz = q.size();
        double sum = 0;
        for (int i = 0; i < sz; ++i) {
            TreeNode* n = q.front(); q.pop();
            sum += n->val;
            if (n->left)  q.push(n->left);
            if (n->right) q.push(n->right);
        }
        out.push_back(sum / sz);
    }
    return out;
}
```

---

## B. Depth & Structure

### 9. Maximum Depth 🟢
```cpp
int maxDepth(TreeNode* root) {
    return root ? 1 + max(maxDepth(root->left), maxDepth(root->right)) : 0;
}
```

---

### 10. Minimum Depth 🟢
```cpp
int minDepth(TreeNode* root) {
    if (!root) return 0;
    if (!root->left)  return 1 + minDepth(root->right);   // ⭐ not a leaf!
    if (!root->right) return 1 + minDepth(root->left);
    return 1 + min(minDepth(root->left), minDepth(root->right));
}
```
⚠️ The naive `min` is wrong: a node with only a left child would return 1, but it isn't a leaf. BFS is even better here — return at the first leaf found.

---

### 11. Balanced Binary Tree 🟢
```cpp
int check(TreeNode* n) {                           // returns height, or -1 if unbalanced
    if (!n) return 0;
    int l = check(n->left);  if (l < 0) return -1;
    int r = check(n->right); if (r < 0) return -1;
    if (abs(l - r) > 1) return -1;                 // ⭐ early exit
    return 1 + max(l, r);
}
bool isBalanced(TreeNode* root) { return check(root) >= 0; }
```
**Complexity:** O(n) — the naive version that calls `height()` inside `isBalanced()` is O(n²).

---

### 12. Diameter of Binary Tree 🟢
> The diameter is the longest path between any two nodes. It may or may not pass through the root.

#### 💬 Think of it like this
Any path in a tree has a single highest point — the topmost node it passes through. So instead of hunting for paths, visit every node and ask: *"what's the longest path whose highest point is me?"*

That's easy to answer: it's the deepest reach into my left subtree plus the deepest reach into my right subtree. Take the maximum of that over all nodes and you have the diameter.

But here's the part that trips people up. **What you report to your parent is not the same as what you record.**

- **Record** (the candidate answer): left depth + right depth — a path going *down one side and up the other*
- **Return** (to your parent): 1 + max(left, right) — because a path continuing upward can only use *one* branch. It can't go down both and still climb.

#### 📊 Seeing the two quantities at node 1

```
            1
           / \
          2   3          Depths: left subtree reaches 2 levels,
         / \             right subtree reaches 1 level.
        4   5

   ┌──────────────────────────────────────────────────────────────┐
   │ ⭐ RECORD at node 1:  left(2) + right(1) = 3                  │
   │    That's the path  4 → 2 → 1 → 3  — three edges.            │
   │    ⭐ It uses BOTH children.                                  │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ RETURN from node 1:  1 + max(2, 1) = 3                     │
   │    That's the depth available to node 1's parent.            │
   │    ⭐ It uses only ONE child — a path coming from above can   │
   │      descend one branch, not both.                           │
   └──────────────────────────────────────────────────────────────┘
```

#### The full traversal

```
   Post-order — children are resolved before the parent

   ┌─────────────────────────────────────────────────────────────┐
   │ node 4:  leaf.  record 0+0 = 0.   return 1                  │
   │ node 5:  leaf.  record 0+0 = 0.   return 1                  │
   │ node 2:  left=1, right=1                                    │
   │          ⭐ record 1+1 = 2  (path 4→2→5)                     │
   │          return 1 + max(1,1) = 2                            │
   │ node 3:  leaf.  record 0.         return 1                  │
   │ node 1:  left=2, right=1                                    │
   │          ⭐ record 2+1 = 3  (path 4→2→1→3)  ⭐ BEST           │
   │          return 1 + max(2,1) = 3                            │
   └─────────────────────────────────────────────────────────────┘

   ANSWER = 3
```

```
   ⭐⭐ THIS "RECORD BOTH, RETURN ONE" PATTERN IS EVERYWHERE

   It's the same shape in:
     • Binary Tree Maximum Path Sum
     • Longest Univalue Path
     • Longest ZigZag Path

   ⭐ Whenever a problem asks for the best path THROUGH a node
     but the recursion must report something USABLE BY THE
     PARENT, you'll need these two different values.
```

```cpp
int diameter = 0;
int depth(TreeNode* n) {
    if (!n) return 0;
    int l = depth(n->left), r = depth(n->right);
    diameter = max(diameter, l + r);               // ⭐ path THROUGH this node
    return 1 + max(l, r);                          // ⭐ but return only ONE side
}
int diameterOfBinaryTree(TreeNode* root) { depth(root); return diameter; }
```
**Key insight:** The classic pattern — the answer at a node uses *both* children, but the value returned to the parent uses only *one*. This shows up in max path sum, longest univalue path, and many others.

---

### 13. Same Tree 🟢
```cpp
bool isSameTree(TreeNode* p, TreeNode* q) {
    if (!p || !q) return p == q;
    return p->val == q->val && isSameTree(p->left, q->left) && isSameTree(p->right, q->right);
}
```

---

### 14. Symmetric Tree 🟢
```cpp
bool mirror(TreeNode* a, TreeNode* b) {
    if (!a || !b) return a == b;
    return a->val == b->val && mirror(a->left, b->right) && mirror(a->right, b->left);
}
bool isSymmetric(TreeNode* root) { return !root || mirror(root->left, root->right); }
```
**Key insight:** Compare left-with-right and right-with-left — the crossed comparison is the whole trick.

---

### 15. Subtree of Another Tree 🟢
```cpp
bool isSubtree(TreeNode* s, TreeNode* t) {
    if (!s) return false;
    if (isSameTree(s, t)) return true;
    return isSubtree(s->left, t) || isSubtree(s->right, t);
}
```
**Complexity:** O(m·n). O(m+n) is possible by serializing both and using KMP.

---

### 16. Invert Binary Tree 🟢
```cpp
TreeNode* invertTree(TreeNode* root) {
    if (!root) return nullptr;
    swap(root->left, root->right);
    invertTree(root->left);
    invertTree(root->right);
    return root;
}
```

---

### 17. Merge Two Binary Trees 🟢
```cpp
TreeNode* mergeTrees(TreeNode* a, TreeNode* b) {
    if (!a) return b;
    if (!b) return a;
    a->val += b->val;
    a->left  = mergeTrees(a->left,  b->left);
    a->right = mergeTrees(a->right, b->right);
    return a;
}
```

---

### 18. Count Complete Tree Nodes 🟡
```cpp
int countNodes(TreeNode* root) {
    if (!root) return 0;
    int lh = 0, rh = 0;
    for (TreeNode* p = root; p; p = p->left)  ++lh;
    for (TreeNode* p = root; p; p = p->right) ++rh;
    if (lh == rh) return (1 << lh) - 1;             // ⭐ perfect subtree, O(1)
    return 1 + countNodes(root->left) + countNodes(root->right);
}
```
**Complexity:** O(log²n) — at each level only one subtree is imperfect, so recursion follows a single path.

---

## C. Path Problems

### 19. Path Sum 🟢
```cpp
bool hasPathSum(TreeNode* root, int target) {
    if (!root) return false;
    if (!root->left && !root->right) return target == root->val;   // ⭐ leaf check
    return hasPathSum(root->left,  target - root->val)
        || hasPathSum(root->right, target - root->val);
}
```

---

### 20. Path Sum II (all root-to-leaf paths) 🟡
```cpp
vector<vector<int>> pathSum(TreeNode* root, int target) {
    vector<vector<int>> out;
    vector<int> path;
    function<void(TreeNode*,int)> dfs = [&](TreeNode* n, int rem) {
        if (!n) return;
        path.push_back(n->val);
        if (!n->left && !n->right && rem == n->val) out.push_back(path);
        else { dfs(n->left, rem - n->val); dfs(n->right, rem - n->val); }
        path.pop_back();                           // ⭐ backtrack
    };
    dfs(root, target);
    return out;
}
```

---

### 21. Path Sum III (any downward path) 🟡
```cpp
int pathSum(TreeNode* root, int target) {
    unordered_map<long long,int> prefix{{0, 1}};   // ⭐ prefix sums on a tree
    int count = 0;
    function<void(TreeNode*, long long)> dfs = [&](TreeNode* n, long long sum) {
        if (!n) return;
        sum += n->val;
        auto it = prefix.find(sum - target);
        if (it != prefix.end()) count += it->second;

        prefix[sum]++;
        dfs(n->left, sum);
        dfs(n->right, sum);
        prefix[sum]--;                             // ⭐ UNDO on the way back up
    };
    dfs(root, 0);
    return count;
}
```
**Complexity:** O(n) instead of the naive O(n²).
**Key insight:** The same prefix-sum-plus-hashmap technique from arrays, applied along root-to-node paths. The decrement on exit is essential — otherwise paths from sibling branches get counted.

---

### 22. Binary Tree Maximum Path Sum 🔴
```cpp
int best = INT_MIN;
int gain(TreeNode* n) {
    if (!n) return 0;
    int l = max(0, gain(n->left));                 // ⭐ negative gains are dropped
    int r = max(0, gain(n->right));
    best = max(best, n->val + l + r);              // path THROUGH this node
    return n->val + max(l, r);                     // return only ONE branch
}
int maxPathSum(TreeNode* root) { gain(root); return best; }
```
**Key insight:** Same "answer uses both, return uses one" pattern as diameter. `max(0, ...)` implements "a negative subtree is worth skipping entirely."

---

### 23. Sum Root to Leaf Numbers 🟡
```cpp
int sumNumbers(TreeNode* root, int cur = 0) {
    if (!root) return 0;
    cur = cur * 10 + root->val;
    if (!root->left && !root->right) return cur;
    return sumNumbers(root->left, cur) + sumNumbers(root->right, cur);
}
```

---

### 24. Binary Tree Paths 🟢
```cpp
vector<string> binaryTreePaths(TreeNode* root) {
    vector<string> out;
    function<void(TreeNode*, string)> dfs = [&](TreeNode* n, string path) {
        if (!n) return;
        path += (path.empty() ? "" : "->") + to_string(n->val);
        if (!n->left && !n->right) { out.push_back(path); return; }
        dfs(n->left, path); dfs(n->right, path);
    };
    dfs(root, "");
    return out;
}
```

---

### 25. Longest Univalue Path 🟡
```cpp
int best = 0;
int arrow(TreeNode* n) {
    if (!n) return 0;
    int l = arrow(n->left), r = arrow(n->right);
    int la = (n->left  && n->left->val  == n->val) ? l + 1 : 0;
    int ra = (n->right && n->right->val == n->val) ? r + 1 : 0;
    best = max(best, la + ra);
    return max(la, ra);
}
```

---

### 26. All Nodes Distance K in Binary Tree 🟡
```cpp
vector<int> distanceK(TreeNode* root, TreeNode* target, int k) {
    unordered_map<TreeNode*, TreeNode*> parent;    // ⭐ make it an undirected graph
    function<void(TreeNode*, TreeNode*)> build = [&](TreeNode* n, TreeNode* p) {
        if (!n) return;
        parent[n] = p;
        build(n->left, n); build(n->right, n);
    };
    build(root, nullptr);

    unordered_set<TreeNode*> seen{target};
    queue<TreeNode*> q{{target}};
    int dist = 0;
    while (!q.empty()) {
        if (dist == k) {
            vector<int> out;
            while (!q.empty()) { out.push_back(q.front()->val); q.pop(); }
            return out;
        }
        int sz = q.size();
        for (int i = 0; i < sz; ++i) {
            TreeNode* n = q.front(); q.pop();
            for (TreeNode* nb : {n->left, n->right, parent[n]})
                if (nb && seen.insert(nb).second) q.push(nb);
        }
        ++dist;
    }
    return {};
}
```
**Key insight:** Adding parent pointers converts the tree into an undirected graph, then it's plain BFS. This conversion is a recurring move.

---

## D. Binary Search Trees

### 27. Validate BST 🟡
```cpp
bool valid(TreeNode* n, long long lo, long long hi) {
    if (!n) return true;
    if (n->val <= lo || n->val >= hi) return false;
    return valid(n->left, lo, n->val) && valid(n->right, n->val, hi);
}
bool isValidBST(TreeNode* root) { return valid(root, LLONG_MIN, LLONG_MAX); }
```
⚠️ **Comparing only with immediate children is wrong.** A node deep in the left subtree must still be less than the root — the bounds must be threaded down. Use `long long` because node values can be `INT_MIN`/`INT_MAX`.

---

### 28. Search in a BST 🟢
```cpp
TreeNode* searchBST(TreeNode* root, int val) {
    while (root && root->val != val) root = val < root->val ? root->left : root->right;
    return root;
}
```

---

### 29. Insert into a BST 🟡
```cpp
TreeNode* insertIntoBST(TreeNode* root, int val) {
    if (!root) return new TreeNode(val);
    if (val < root->val) root->left  = insertIntoBST(root->left, val);
    else                 root->right = insertIntoBST(root->right, val);
    return root;
}
```

---

### 30. Delete Node in a BST 🟡
```cpp
TreeNode* deleteNode(TreeNode* root, int key) {
    if (!root) return nullptr;
    if (key < root->val)      root->left  = deleteNode(root->left, key);
    else if (key > root->val) root->right = deleteNode(root->right, key);
    else {
        if (!root->left)  { TreeNode* r = root->right; delete root; return r; }
        if (!root->right) { TreeNode* l = root->left;  delete root; return l; }
        TreeNode* succ = root->right;                  // ⭐ inorder successor
        while (succ->left) succ = succ->left;
        root->val = succ->val;
        root->right = deleteNode(root->right, succ->val);
    }
    return root;
}
```
**Key insight:** Three cases — no children, one child (splice), two children (replace with the inorder successor, then delete that successor).

---

### 31. Kth Smallest Element in a BST 🟡
```cpp
int kthSmallest(TreeNode* root, int k) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        if (--k == 0) return cur->val;             // ⭐ inorder = sorted order
        cur = cur->right;
    }
    return -1;
}
```
🎤 **Follow-up (frequent modifications):** augment each node with a subtree size count, giving O(log n) per query.

---

### 32. Lowest Common Ancestor of a BST 🟢
```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    while (root) {
        if (p->val < root->val && q->val < root->val) root = root->left;
        else if (p->val > root->val && q->val > root->val) root = root->right;
        else return root;                          // ⭐ the split point
    }
    return nullptr;
}
```

---

### 33. Lowest Common Ancestor of a Binary Tree 🟡
> Find the deepest node that has both `p` and `q` as descendants. No BST property — values are arbitrary.

#### 💬 Think of it like this
The recursion answers a deliberately loose question: *"searching my subtree, what did you find?"*

Three possible replies:
- **Nothing** → return null
- **One of them** → return that node
- **Both** (one in my left subtree, one in my right) → ⭐ *I am the LCA* — return myself

The elegance is that the same return value means different things at different points, and it works out. If a node hears back from *both* children, the two targets are on opposite sides, so this node is the meeting point. If only one child reports a find, that answer just bubbles upward unchanged.

#### 📊 Tracing LCA(4, 5)

```
            3
           / \
          5   1
         / \
        6   2

   Post-order — leaves resolve first

   ┌─────────────────────────────────────────────────────────────┐
   │ node 6:  not a target, no children → return null            │
   │ node 2:  not a target, no children → return null            │
   ├─────────────────────────────────────────────────────────────┤
   │ node 5:  ⭐ IS a target → return 5 immediately               │
   │          (no need to search below — if the other target is  │
   │           down there, 5 is still the correct ancestor)      │
   ├─────────────────────────────────────────────────────────────┤
   │ node 1:  not a target, both children null → return null     │
   ├─────────────────────────────────────────────────────────────┤
   │ node 3:  left returned 5,  right returned null              │
   │          ⭐ only ONE side found something → pass it up       │
   │          return 5                                            │
   └─────────────────────────────────────────────────────────────┘

   Now LCA(6, 2):

   ┌─────────────────────────────────────────────────────────────┐
   │ node 6:  ⭐ IS a target → return 6                           │
   │ node 2:  ⭐ IS a target → return 2                           │
   │ node 5:  left=6, right=2 → ⭐⭐ BOTH non-null                 │
   │          → node 5 IS the LCA → return 5                     │
   │ node 3:  left=5, right=null → pass up → return 5            │
   └─────────────────────────────────────────────────────────────┘

   ANSWER = 5
```

#### Why returning early on a match is correct

```
   ⭐ When node 5 discovers it IS a target, it returns
     immediately without searching its own subtree.

     Isn't that a problem if node 2 is beneath it?

     No — because if the other target IS beneath node 5, then
     node 5 is genuinely the lowest common ancestor. Searching
     deeper could not produce a better answer.

   ⭐ This is why the algorithm assumes both nodes EXIST in the
     tree. If one might be absent, you need a full traversal
     with explicit found-flags rather than this early return.
```

```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;
    TreeNode* l = lowestCommonAncestor(root->left, p, q);
    TreeNode* r = lowestCommonAncestor(root->right, p, q);
    if (l && r) return root;                       // ⭐ found in both → this is the LCA
    return l ? l : r;                              // propagate whichever exists
}
```
**Key insight:** Returns "the node I found, or the LCA if I found both." If both children report a find, the current node is the meeting point.

---

### 34. Convert Sorted Array to BST 🟢
```cpp
TreeNode* build(vector<int>& a, int lo, int hi) {
    if (lo > hi) return nullptr;
    int mid = lo + (hi - lo) / 2;
    return new TreeNode(a[mid], build(a, lo, mid - 1), build(a, mid + 1, hi));
}
TreeNode* sortedArrayToBST(vector<int>& a) { return build(a, 0, a.size() - 1); }
```

---

### 35. Convert Sorted List to BST 🟡
```cpp
ListNode* cur;
TreeNode* build(int n) {                           // ⭐ inorder simulation, O(n)
    if (n == 0) return nullptr;
    TreeNode* left = build(n / 2);
    TreeNode* root = new TreeNode(cur->val);
    cur = cur->next;
    root->left = left;
    root->right = build(n - n / 2 - 1);
    return root;
}
TreeNode* sortedListToBST(ListNode* head) {
    int n = 0;
    for (ListNode* p = head; p; p = p->next) ++n;
    cur = head;
    return build(n);
}
```
**Key insight:** Build in inorder so the list is consumed left-to-right — no need for random access, so it's O(n) not O(n log n).

---

### 36. BST Iterator 🟡
```cpp
class BSTIterator {
    stack<TreeNode*> st;
    void pushLeft(TreeNode* n) { while (n) { st.push(n); n = n->left; } }
public:
    BSTIterator(TreeNode* root) { pushLeft(root); }
    int next() {
        TreeNode* n = st.top(); st.pop();
        pushLeft(n->right);
        return n->val;
    }
    bool hasNext() { return !st.empty(); }
};
```
**Complexity:** amortized O(1) per `next`, O(h) space — a paused iterative inorder traversal.

---

### 37. Recover Binary Search Tree 🔴
```cpp
void recoverTree(TreeNode* root) {
    TreeNode *first = nullptr, *second = nullptr, *prev = nullptr;
    function<void(TreeNode*)> inorder = [&](TreeNode* n) {
        if (!n) return;
        inorder(n->left);
        if (prev && prev->val > n->val) {           // ⭐ a descent in inorder
            if (!first) first = prev;               // first violation: take prev
            second = n;                             // second: take current
        }
        prev = n;
        inorder(n->right);
    };
    inorder(root);
    swap(first->val, second->val);
}
```
**Key insight:** Inorder on a valid BST is strictly increasing. Two swapped nodes produce either two descents (take the first's `prev` and the second's `cur`) or one descent (adjacent swap).

---

### 38. Range Sum of BST 🟢
```cpp
int rangeSumBST(TreeNode* root, int lo, int hi) {
    if (!root) return 0;
    if (root->val < lo) return rangeSumBST(root->right, lo, hi);   // ⭐ prune
    if (root->val > hi) return rangeSumBST(root->left, lo, hi);
    return root->val + rangeSumBST(root->left, lo, hi) + rangeSumBST(root->right, lo, hi);
}
```

---

### 39. Minimum Absolute Difference in BST 🟢
```cpp
int getMinimumDifference(TreeNode* root) {
    int best = INT_MAX;
    TreeNode* prev = nullptr;
    function<void(TreeNode*)> inorder = [&](TreeNode* n) {
        if (!n) return;
        inorder(n->left);
        if (prev) best = min(best, n->val - prev->val);   // ⭐ adjacent in sorted order
        prev = n;
        inorder(n->right);
    };
    inorder(root);
    return best;
}
```

---

### 40. Two Sum IV — Input is a BST 🟢
```cpp
bool findTarget(TreeNode* root, int k) {
    unordered_set<int> seen;
    function<bool(TreeNode*)> dfs = [&](TreeNode* n) -> bool {
        if (!n) return false;
        if (seen.count(k - n->val)) return true;
        seen.insert(n->val);
        return dfs(n->left) || dfs(n->right);
    };
    return dfs(root);
}
```
🎤 **Follow-up (O(h) space):** two BST iterators, one forward and one backward, converging like two pointers.

---

## E. Construction & Serialization

### 41. Construct Tree from Preorder and Inorder 🟡
```cpp
TreeNode* buildTree(vector<int>& pre, vector<int>& in) {
    unordered_map<int,int> pos;
    for (int i = 0; i < (int)in.size(); ++i) pos[in[i]] = i;
    int p = 0;
    function<TreeNode*(int,int)> build = [&](int lo, int hi) -> TreeNode* {
        if (lo > hi) return nullptr;
        int rootVal = pre[p++];                    // ⭐ preorder gives roots in order
        int mid = pos[rootVal];
        TreeNode* n = new TreeNode(rootVal);
        n->left  = build(lo, mid - 1);             // ⭐ LEFT first (preorder order)
        n->right = build(mid + 1, hi);
        return n;
    };
    return build(0, in.size() - 1);
}
```
**Complexity:** O(n) with the position map.

---

### 42. Construct Tree from Inorder and Postorder 🟡
```cpp
TreeNode* buildTree(vector<int>& in, vector<int>& post) {
    unordered_map<int,int> pos;
    for (int i = 0; i < (int)in.size(); ++i) pos[in[i]] = i;
    int p = post.size() - 1;
    function<TreeNode*(int,int)> build = [&](int lo, int hi) -> TreeNode* {
        if (lo > hi) return nullptr;
        int rootVal = post[p--];
        int mid = pos[rootVal];
        TreeNode* n = new TreeNode(rootVal);
        n->right = build(mid + 1, hi);             // ⭐ RIGHT first (reverse postorder)
        n->left  = build(lo, mid - 1);
        return n;
    };
    return build(0, in.size() - 1);
}
```

---

### 43. Construct BST from Preorder 🟡
```cpp
TreeNode* bstFromPreorder(vector<int>& pre) {
    int i = 0;
    function<TreeNode*(int)> build = [&](int bound) -> TreeNode* {
        if (i == (int)pre.size() || pre[i] > bound) return nullptr;
        TreeNode* n = new TreeNode(pre[i++]);
        n->left  = build(n->val);                  // ⭐ left subtree bounded by root
        n->right = build(bound);
        return n;
    };
    return build(INT_MAX);
}
```
**Complexity:** O(n) — the upper-bound parameter avoids searching for the split point.

---

### 44. Serialize and Deserialize Binary Tree 🔴
```cpp
class Codec {
public:
    string serialize(TreeNode* root) {
        string s;
        function<void(TreeNode*)> dfs = [&](TreeNode* n) {
            if (!n) { s += "#,"; return; }         // ⭐ null markers make it unique
            s += to_string(n->val) + ",";
            dfs(n->left); dfs(n->right);
        };
        dfs(root);
        return s;
    }
    TreeNode* deserialize(string data) {
        int i = 0;
        function<TreeNode*()> build = [&]() -> TreeNode* {
            int j = data.find(',', i);
            string tok = data.substr(i, j - i);
            i = j + 1;
            if (tok == "#") return nullptr;
            TreeNode* n = new TreeNode(stoi(tok));
            n->left = build(); n->right = build();
            return n;
        };
        return build();
    }
};
```
**Key insight:** Preorder with explicit null markers is uniquely decodable. Without markers, preorder alone is ambiguous — that's why the "two traversals" problems exist.

---

### 45. Serialize and Deserialize BST 🟡
```cpp
// A BST needs no null markers — the ordering property provides the structure
string serialize(TreeNode* root) {
    string s;
    function<void(TreeNode*)> dfs = [&](TreeNode* n) {
        if (!n) return;
        s += to_string(n->val) + ",";
        dfs(n->left); dfs(n->right);
    };
    dfs(root);
    return s;
}
TreeNode* deserialize(string data) {
    vector<int> pre;
    stringstream ss(data);
    string tok;
    while (getline(ss, tok, ',')) if (!tok.empty()) pre.push_back(stoi(tok));
    int i = 0;
    function<TreeNode*(int)> build = [&](int bound) -> TreeNode* {
        if (i == (int)pre.size() || pre[i] > bound) return nullptr;
        TreeNode* n = new TreeNode(pre[i++]);
        n->left = build(n->val);
        n->right = build(bound);
        return n;
    };
    return build(INT_MAX);
}
```

---

### 46. Flatten Binary Tree to Linked List 🟡
```cpp
void flatten(TreeNode* root) {
    TreeNode* cur = root;
    while (cur) {
        if (cur->left) {
            TreeNode* pred = cur->left;
            while (pred->right) pred = pred->right;   // rightmost of left subtree
            pred->right = cur->right;                 // ⭐ attach the right subtree
            cur->right = cur->left;
            cur->left = nullptr;
        }
        cur = cur->right;
    }
}
```
**Complexity:** O(n) / O(1) — Morris-style threading.

---

### 47. Populating Next Right Pointers 🟡
```cpp
Node* connect(Node* root) {
    Node* leftmost = root;
    while (leftmost && leftmost->left) {           // perfect tree assumption
        Node* head = leftmost;
        while (head) {
            head->left->next = head->right;        // same parent
            if (head->next) head->right->next = head->next->left;   // ⭐ across parents
            head = head->next;
        }
        leftmost = leftmost->left;
    }
    return root;
}
```
**Complexity:** O(n) / O(1) — uses the already-built `next` pointers of the level above instead of a queue.

---

## F. Advanced

### 48. Binary Tree Cameras 🔴
```cpp
int cameras = 0;
// 0 = needs cover, 1 = has camera, 2 = covered
int dfs(TreeNode* n) {
    if (!n) return 2;                              // ⭐ null counts as covered
    int l = dfs(n->left), r = dfs(n->right);
    if (l == 0 || r == 0) { ++cameras; return 1; } // a child is uncovered → place here
    if (l == 1 || r == 1) return 2;                // a child has a camera → covered
    return 0;                                      // both covered but I'm not
}
int minCameraCover(TreeNode* root) {
    return (dfs(root) == 0 ? ++cameras : cameras);
}
```
**Key insight:** Greedy bottom-up — place cameras as high as possible while still covering leaves. A three-state return encodes everything the parent needs.

---

### 49. House Robber III 🟡
```cpp
pair<int,int> rob(TreeNode* n) {                   // {rob this node, skip this node}
    if (!n) return {0, 0};
    auto [rl, sl] = rob(n->left);
    auto [rr, sr] = rob(n->right);
    return { n->val + sl + sr,                     // rob → children must be skipped
             max(rl, sl) + max(rr, sr) };          // skip → children are free
}
int rob(TreeNode* root) { auto [a, b] = rob(root); return max(a, b); }
```
**Key insight:** Tree DP — return a pair covering both states so the parent can choose.

---

### 50. Distribute Coins in Binary Tree 🟡
```cpp
int moves = 0;
int dfs(TreeNode* n) {                             // returns the net excess/deficit
    if (!n) return 0;
    int l = dfs(n->left), r = dfs(n->right);
    moves += abs(l) + abs(r);                      // ⭐ coins crossing these edges
    return n->val + l + r - 1;                     // keep 1, pass the rest up
}
int distributeCoins(TreeNode* root) { dfs(root); return moves; }
```
**Key insight:** The number of coins that must traverse an edge is exactly the absolute excess of the subtree below it. Summing over all edges gives the answer.

---

### Bonus: Trie
> See [Patterns §17](00-patterns.md#pattern-17--trie) for the implementation. Common problems: Implement Trie, Word Search II, Add and Search Word, Longest Word in Dictionary, Replace Words, Maximum XOR of Two Numbers.

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║                     TREES — PATTERN RECALL                        ║
╠═══════════════════════════════════════════════════════════════════╣
║ TEMPLATE: base case → recurse both children → combine             ║
║   Ask: "what do I need FROM children, what do I RETURN to parent?"║
╠═══════════════════════════════════════════════════════════════════╣
║ ⭐ THE KEY PATTERN (diameter / max path sum / longest univalue):   ║
║   answer at a node uses BOTH children                             ║
║   value returned to parent uses ONE child                         ║
╠═══════════════════════════════════════════════════════════════════╣
║ TRAVERSALS                                                        ║
║   inorder on a BST = SORTED order → validate, kth, min-diff       ║
║   preorder = roots first → construction, serialization            ║
║   postorder = children first → deletion, bottom-up computation    ║
║   BFS with `int sz = q.size()` → level-by-level                   ║
║   Morris = O(1) space via temporary threads                       ║
╠═══════════════════════════════════════════════════════════════════╣
║ BST                                                               ║
║   validate → pass (lo, hi) BOUNDS down, not just child comparison ║
║   delete → 0 children / 1 child splice / 2 children use successor ║
║   LCA in BST → first node where p and q land on opposite sides    ║
║   LCA general → return "found node, or LCA if both found"         ║
╠═══════════════════════════════════════════════════════════════════╣
║ TRICKS                                                            ║
║   parent map → tree becomes an undirected graph → BFS             ║
║   prefix sum + hashmap works on ROOT-TO-NODE paths (undo on exit) ║
║   tree DP → return a PAIR/STRUCT of states                        ║
║   serialize with NULL MARKERS or preorder is ambiguous            ║
║   balanced check → return -1 sentinel for early exit, O(n) not O(n²)║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Heaps & Intervals →](07-heaps-intervals.md)
