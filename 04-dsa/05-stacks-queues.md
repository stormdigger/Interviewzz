# 🥞 Stacks & Queues — 30 Problems

> The stack's superpower is the **monotonic stack** — it answers "next greater / previous smaller" for every element in O(n) total. That single pattern accounts for most stack problems in interviews.

**Prerequisite:** [Patterns & Foundations](00-patterns.md)

---

## 🧠 The Monotonic Stack

```cpp
// NEXT GREATER ELEMENT — stack holds indices with DECREASING values
vector<int> nextGreater(vector<int>& a) {
    int n = a.size();
    vector<int> res(n, -1);
    stack<int> st;
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && a[st.top()] < a[i]) {
            res[st.top()] = a[i];                  // a[i] resolves everyone waiting
            st.pop();
        }
        st.push(i);
    }
    return res;
}
```

```
   THE FOUR VARIANTS — just change the comparison and direction

   ┌──────────────────────┬───────────┬──────────────────────┐
   │ Want                 │ Direction │ Pop while            │
   ├──────────────────────┼───────────┼──────────────────────┤
   │ next greater         │  L → R    │ a[top] <  a[i]       │
   │ next smaller         │  L → R    │ a[top] >  a[i]       │
   │ previous greater     │  R → L    │ a[top] <  a[i]       │
   │ previous smaller     │  R → L    │ a[top] >  a[i]       │
   └──────────────────────┴───────────┴──────────────────────┘

   Use <= / >= instead of < / > to control duplicate handling.
```

🧠 **Intuition:** the stack holds elements "still waiting for an answer." When a larger element arrives, every waiting element smaller than it gets its answer at once and leaves permanently. Each index is pushed once and popped once → **O(n)**.

**Signals that a problem is a monotonic stack problem:**
```
   "next/previous greater/smaller"
   "how many days until..."
   "largest rectangle"
   "trapping water"
   "remove k digits to make the smallest number"
   "maximum of every window"  (monotonic DEQUE)
```

---

## A. Classic Stack

### 1. Valid Parentheses 🟢
```cpp
bool isValid(string s) {
    stack<char> st;
    unordered_map<char,char> pair{{')','('},{']','['},{'}','{'}};
    for (char c : s) {
        if (pair.count(c)) {
            if (st.empty() || st.top() != pair[c]) return false;
            st.pop();
        } else st.push(c);
    }
    return st.empty();                             // ⭐ must be empty at the end
}
```

---

### 2. Min Stack 🟡
```cpp
class MinStack {
    stack<pair<int,int>> st;                       // {value, min so far}
public:
    void push(int x) { st.push({x, st.empty() ? x : min(x, st.top().second)}); }
    void pop()       { st.pop(); }
    int top()        { return st.top().first; }
    int getMin()     { return st.top().second; }
};

// ⭐ O(1) extra space variant — store the ENCODED difference
class MinStackOptimal {
    stack<long long> st;
    long long mn = 0;
public:
    void push(int x) {
        if (st.empty()) { st.push(0); mn = x; }
        else {
            st.push((long long)x - mn);            // store the delta
            if (x < mn) mn = x;
        }
    }
    void pop() {
        long long d = st.top(); st.pop();
        if (d < 0) mn = mn - d;                    // restore the previous min
    }
    int top()    { long long d = st.top(); return d > 0 ? d + mn : mn; }
    int getMin() { return mn; }
};
```

---

### 3. Evaluate Reverse Polish Notation 🟡
```cpp
int evalRPN(vector<string>& tokens) {
    stack<long long> st;
    for (auto& t : tokens) {
        if (t.size() > 1 || isdigit(t[0])) { st.push(stoll(t)); continue; }
        long long b = st.top(); st.pop();
        long long a = st.top(); st.pop();
        switch (t[0]) {
            case '+': st.push(a + b); break;
            case '-': st.push(a - b); break;       // ⭐ order matters
            case '*': st.push(a * b); break;
            case '/': st.push(a / b); break;       // C++ truncates toward zero
        }
    }
    return st.top();
}
```
⚠️ `t.size() > 1 || isdigit(t[0])` correctly identifies negative numbers like `"-5"`.

---

### 4. Basic Calculator (with parentheses) 🔴
```cpp
int calculate(string s) {
    stack<int> st;
    int result = 0, num = 0, sign = 1;
    for (char c : s) {
        if (isdigit(c)) num = num * 10 + (c - '0');
        else if (c == '+') { result += sign * num; num = 0; sign = 1; }
        else if (c == '-') { result += sign * num; num = 0; sign = -1; }
        else if (c == '(') {
            st.push(result); st.push(sign);        // ⭐ save the outer context
            result = 0; sign = 1;
        } else if (c == ')') {
            result += sign * num; num = 0;
            result *= st.top(); st.pop();          // apply the saved sign
            result += st.top(); st.pop();          // add the saved result
        }
    }
    return result + sign * num;
}
```

---

### 5. Basic Calculator II (* and /, no parens) 🟡
```cpp
int calculate(string s) {
    stack<int> st;
    int num = 0;
    char op = '+';
    for (int i = 0; i < (int)s.size(); ++i) {
        char c = s[i];
        if (isdigit(c)) num = num * 10 + (c - '0');
        if ((!isdigit(c) && c != ' ') || i == (int)s.size() - 1) {
            if (op == '+') st.push(num);
            else if (op == '-') st.push(-num);
            else if (op == '*') { int t = st.top(); st.pop(); st.push(t * num); }
            else                { int t = st.top(); st.pop(); st.push(t / num); }
            op = c; num = 0;
        }
    }
    int r = 0;
    while (!st.empty()) { r += st.top(); st.pop(); }
    return r;
}
```
**Key insight:** Push additive terms; resolve multiplicative operators immediately against the stack top. Summing at the end handles precedence correctly.

---

### 6. Decode String 🟡
> `"3[a2[c]]"` → `"accaccacc"`

```cpp
string decodeString(string s) {
    stack<int> nums;
    stack<string> strs;
    string cur;
    int num = 0;
    for (char c : s) {
        if (isdigit(c)) num = num * 10 + (c - '0');
        else if (c == '[') { nums.push(num); strs.push(cur); num = 0; cur.clear(); }
        else if (c == ']') {
            string repeated;
            for (int i = 0; i < nums.top(); ++i) repeated += cur;
            nums.pop();
            cur = strs.top() + repeated;           // ⭐ prepend the saved prefix
            strs.pop();
        } else cur += c;
    }
    return cur;
}
```

---

### 7. Simplify Path 🟡
```cpp
string simplifyPath(string path) {
    vector<string> st;
    stringstream ss(path);
    string part;
    while (getline(ss, part, '/')) {
        if (part.empty() || part == ".") continue;
        if (part == "..") { if (!st.empty()) st.pop_back(); }
        else st.push_back(part);
    }
    string out;
    for (auto& p : st) out += "/" + p;
    return out.empty() ? "/" : out;
}
```

---

### 8. Remove All Adjacent Duplicates 🟢
```cpp
string removeDuplicates(string s) {
    string st;
    for (char c : s) {
        if (!st.empty() && st.back() == c) st.pop_back();
        else st.push_back(c);
    }
    return st;
}
```
**Key insight:** A `string` used as a stack is both faster and simpler than `stack<char>` here.

---

### 9. Remove All Adjacent Duplicates II (k copies) 🟡
```cpp
string removeDuplicates(string s, int k) {
    vector<pair<char,int>> st;                     // {char, run length}
    for (char c : s) {
        if (!st.empty() && st.back().first == c) {
            if (++st.back().second == k) st.pop_back();
        } else st.push_back({c, 1});
    }
    string out;
    for (auto& [c, n] : st) out += string(n, c);
    return out;
}
```

---

### 10. Minimum Remove to Make Valid Parentheses 🟡
```cpp
string minRemoveToMakeValid(string s) {
    vector<int> st;                                // indices of unmatched '('
    vector<bool> remove(s.size(), false);
    for (int i = 0; i < (int)s.size(); ++i) {
        if (s[i] == '(') st.push_back(i);
        else if (s[i] == ')') {
            if (st.empty()) remove[i] = true;      // unmatched ')'
            else st.pop_back();
        }
    }
    for (int i : st) remove[i] = true;             // leftover unmatched '('
    string out;
    for (int i = 0; i < (int)s.size(); ++i) if (!remove[i]) out += s[i];
    return out;
}
```

---

### 11. Score of Parentheses 🟡
```cpp
int scoreOfParentheses(string s) {
    stack<int> st;
    st.push(0);
    for (char c : s) {
        if (c == '(') st.push(0);
        else {
            int v = st.top(); st.pop();
            st.top() += max(2 * v, 1);             // ⭐ "()" = 1, "(X)" = 2X
        }
    }
    return st.top();
}
```

---

### 12. Longest Valid Parentheses 🔴
```cpp
int longestValidParentheses(string s) {
    stack<int> st;
    st.push(-1);                                   // ⭐ sentinel: base index
    int best = 0;
    for (int i = 0; i < (int)s.size(); ++i) {
        if (s[i] == '(') st.push(i);
        else {
            st.pop();
            if (st.empty()) st.push(i);            // new base
            else best = max(best, i - st.top());
        }
    }
    return best;
}
```
**Key insight:** The stack holds the index just *before* the current valid run. The `-1` sentinel makes the length arithmetic uniform.

---

## B. Monotonic Stack

### 13. Next Greater Element I 🟢
```cpp
vector<int> nextGreaterElement(vector<int>& q, vector<int>& a) {
    unordered_map<int,int> nge;
    stack<int> st;
    for (int x : a) {
        while (!st.empty() && st.top() < x) { nge[st.top()] = x; st.pop(); }
        st.push(x);
    }
    vector<int> out;
    for (int x : q) out.push_back(nge.count(x) ? nge[x] : -1);
    return out;
}
```

---

### 14. Next Greater Element II (circular) 🟡
```cpp
vector<int> nextGreaterElements(vector<int>& a) {
    int n = a.size();
    vector<int> res(n, -1);
    stack<int> st;
    for (int i = 0; i < 2 * n; ++i) {              // ⭐ two passes simulate a circle
        int idx = i % n;
        while (!st.empty() && a[st.top()] < a[idx]) { res[st.top()] = a[idx]; st.pop(); }
        if (i < n) st.push(idx);                   // only push during the first pass
    }
    return res;
}
```

---

### 15. Daily Temperatures 🟡
```cpp
vector<int> dailyTemperatures(vector<int>& t) {
    int n = t.size();
    vector<int> res(n, 0);
    stack<int> st;
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && t[st.top()] < t[i]) {
            res[st.top()] = i - st.top();          // ⭐ distance, not value
            st.pop();
        }
        st.push(i);
    }
    return res;
}
```

---

### 16. Largest Rectangle in Histogram 🔴
> Given bar heights, find the area of the largest rectangle that fits inside the histogram.

#### 💬 Think of it like this
Flip the question around. Instead of asking "where's the biggest rectangle?", ask about **each bar individually**: *if this bar's height were the rectangle's height, how wide could the rectangle get?*

It can extend left until it hits a bar shorter than itself, and right until it hits a bar shorter than itself. Anything shorter blocks it — the rectangle can't be that tall there.

So for every bar you need two things: the index of the **previous smaller** bar, and the index of the **next smaller** bar. The width is the gap between them.

That's exactly what a monotonic stack computes — and beautifully, it finds *both* boundaries in a single pass.

#### 📊 Watching it on `[2, 1, 5, 6, 2, 3]`

```
   heights:  2  1  5  6  2  3
             ▆  ▃  █  █  ▆  ▇

   For the bar of height 5 at index 2:
     ← blocked by height 1 at index 1
     → blocked by height 2 at index 4
     width = 4 − 1 − 1 = 2,  area = 5 × 2 = 10

   For the bar of height 6 at index 3:
     ← blocked by 5 (index 2),  → blocked by 2 (index 4)
     width = 1,  area = 6

   ⭐ For the bar of height 1 at index 1:
     ← nothing shorter to the left,  → nothing shorter to the right
     width = 6,  area = 6

   ⭐ THE ANSWER IS 10 — the [5,6] pair at height 5.
```

#### How the stack finds both boundaries at once

```
   The stack holds indices whose heights are INCREASING.
   Think of it as "bars still waiting to find their right edge."

   ┌──────────────────────────────────────────────────────────────┐
   │ i=0, h=2:  stack empty → push.        stack: [0]             │
   ├──────────────────────────────────────────────────────────────┤
   │ i=1, h=1:  1 < h[0]=2 → ⭐ bar 0 is BLOCKED here.             │
   │            POP index 0 (height 2).                           │
   │              right edge = i = 1                              │
   │              left edge  = new stack top = none → -1          │
   │              width = 1 − (−1) − 1 = 1,  area = 2×1 = 2       │
   │            push 1.                    stack: [1]             │
   ├──────────────────────────────────────────────────────────────┤
   │ i=2, h=5:  5 > 1 → nothing blocked. push.  stack: [1,2]      │
   │ i=3, h=6:  6 > 5 → push.                   stack: [1,2,3]    │
   ├──────────────────────────────────────────────────────────────┤
   │ i=4, h=2:  2 < 6 → POP index 3 (height 6)                    │
   │              right = 4, left = stack top = 2                 │
   │              width = 4 − 2 − 1 = 1,  area = 6                │
   │            2 < 5 → POP index 2 (height 5)                    │
   │              right = 4, left = stack top = 1                 │
   │              width = 4 − 1 − 1 = 2,  area = ⭐ 10  BEST       │
   │            2 > 1 → push.               stack: [1,4]          │
   ├──────────────────────────────────────────────────────────────┤
   │ i=5, h=3:  3 > 2 → push.               stack: [1,4,5]        │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ SENTINEL h=0 appended → pops everything remaining,         │
   │   giving each leftover bar its right edge at the end.        │
   └──────────────────────────────────────────────────────────────┘

   ANSWER = 10
```

#### The two insights that make this click

```
   ⭐ 1. WHEN A BAR IS POPPED, BOTH ITS BOUNDARIES ARE KNOWN.
        The bar `i` that triggered the pop is its NEXT SMALLER.
        The new stack top is its PREVIOUS SMALLER.
        One pass, both edges.

   ⭐ 2. THE SENTINEL (appending height 0) IS ESSENTIAL.
        Without it, bars still on the stack at the end never
        get popped and never get measured. A height of 0 is
        smaller than everything, so it flushes the stack.

   ⚠️ Note `width = right − left − 1`, using the stack top AFTER
     popping. That's the index of the previous smaller bar, and
     the rectangle sits strictly between the two boundaries.
```

```cpp
int largestRectangleArea(vector<int>& h) {
    h.push_back(0);                                // ⭐ sentinel flushes the stack
    stack<int> st;
    int best = 0;
    for (int i = 0; i < (int)h.size(); ++i) {
        while (!st.empty() && h[st.top()] >= h[i]) {
            int height = h[st.top()]; st.pop();
            int left = st.empty() ? -1 : st.top();
            best = max(best, height * (i - left - 1));
        }
        st.push(i);
    }
    h.pop_back();
    return best;
}
```
**Complexity:** O(n).
**Key insight:** For each bar, the maximal rectangle *using it as the height* extends left to the previous smaller bar and right to the next smaller bar. The monotonic stack finds both boundaries in one pass — when bar `i` pops bar `j`, `i` is `j`'s next-smaller and the new stack top is `j`'s previous-smaller.

---

### 17. Maximal Rectangle 🔴
```cpp
int maximalRectangle(vector<vector<char>>& m) {
    if (m.empty()) return 0;
    int C = m[0].size(), best = 0;
    vector<int> h(C, 0);
    for (auto& row : m) {
        for (int j = 0; j < C; ++j) h[j] = (row[j] == '1') ? h[j] + 1 : 0;
        best = max(best, largestRectangleArea(h));  // ⭐ reduce 2D to 1D
    }
    return best;
}
```
**Complexity:** O(R·C).
**Key insight:** Each row becomes a histogram of consecutive 1s above it. Reduction to a solved problem — the most valuable move in problem solving.

---

### 18. Trapping Rain Water (stack version) 🔴
```cpp
int trap(vector<int>& h) {
    stack<int> st;
    int water = 0;
    for (int i = 0; i < (int)h.size(); ++i) {
        while (!st.empty() && h[st.top()] < h[i]) {
            int bottom = h[st.top()]; st.pop();
            if (st.empty()) break;
            int width = i - st.top() - 1;
            int bounded = min(h[st.top()], h[i]) - bottom;
            water += width * bounded;              // ⭐ fills HORIZONTAL layers
        }
        st.push(i);
    }
    return water;
}
```
**Key insight:** The stack version accumulates water in horizontal slabs; the two-pointer version (Two Pointers §7) does vertical columns. Both are O(n); two pointers uses O(1) space.

---

### 19. Sum of Subarray Minimums 🟡
```cpp
int sumSubarrayMins(vector<int>& a) {
    const long long MOD = 1e9 + 7;
    int n = a.size();
    vector<int> left(n), right(n);                 // counts of extension
    stack<int> st;

    for (int i = 0; i < n; ++i) {                  // previous smaller (strict)
        while (!st.empty() && a[st.top()] > a[i]) st.pop();
        left[i] = st.empty() ? i + 1 : i - st.top();
        st.push(i);
    }
    while (!st.empty()) st.pop();
    for (int i = n - 1; i >= 0; --i) {             // next smaller-or-equal
        while (!st.empty() && a[st.top()] >= a[i]) st.pop();
        right[i] = st.empty() ? n - i : st.top() - i;
        st.push(i);
    }

    long long total = 0;
    for (int i = 0; i < n; ++i)
        total = (total + (long long)a[i] * left[i] % MOD * right[i]) % MOD;
    return total;
}
```
**Key insight:** Instead of enumerating subarrays, count for each element **how many subarrays it is the minimum of**: `left[i] × right[i]`. The strict/non-strict asymmetry (`>` one way, `>=` the other) prevents double-counting equal values.

---

### 20. Remove K Digits 🟡
```cpp
string removeKdigits(string num, int k) {
    string st;
    for (char c : num) {
        while (k && !st.empty() && st.back() > c) { st.pop_back(); --k; }
        st.push_back(c);
    }
    st.resize(st.size() - k);                      // ⭐ still have removals left
    int i = 0;
    while (i < (int)st.size() && st[i] == '0') ++i;   // strip leading zeros
    string out = st.substr(i);
    return out.empty() ? "0" : out;
}
```
**Key insight:** Greedy — removing a digit that's larger than its successor always reduces the number. A monotonically non-decreasing stack achieves this.

---

### 21. Create Maximum Number 🔴
```cpp
vector<int> maxSubsequence(vector<int>& a, int k) {
    vector<int> st;
    int drop = a.size() - k;
    for (int x : a) {
        while (drop && !st.empty() && st.back() < x) { st.pop_back(); --drop; }
        st.push_back(x);
    }
    st.resize(k);
    return st;
}
bool greaterVec(vector<int>& a, int i, vector<int>& b, int j) {
    while (i < (int)a.size() && j < (int)b.size() && a[i] == b[j]) { ++i; ++j; }
    return j == (int)b.size() || (i < (int)a.size() && a[i] > b[j]);
}
vector<int> maxNumber(vector<int>& n1, vector<int>& n2, int k) {
    vector<int> best;
    for (int i = max(0, k - (int)n2.size()); i <= min(k, (int)n1.size()); ++i) {
        auto a = maxSubsequence(n1, i), b = maxSubsequence(n2, k - i);
        vector<int> merged;
        int p = 0, q = 0;
        while (p < (int)a.size() || q < (int)b.size())
            merged.push_back(greaterVec(a, p, b, q) ? a[p++] : b[q++]);
        if (merged > best) best = merged;
    }
    return best;
}
```

---

### 22. Remove Duplicate Letters 🟡
```cpp
string removeDuplicateLetters(string s) {
    int cnt[26] = {};
    bool inStack[26] = {};
    for (char c : s) cnt[c - 'a']++;

    string st;
    for (char c : s) {
        cnt[c - 'a']--;
        if (inStack[c - 'a']) continue;            // already placed
        while (!st.empty() && st.back() > c && cnt[st.back() - 'a'] > 0) {
            inStack[st.back() - 'a'] = false;      // ⭐ safe: it appears again later
            st.pop_back();
        }
        st.push_back(c);
        inStack[c - 'a'] = true;
    }
    return st;
}
```
**Key insight:** Pop a larger character only if it appears again later — otherwise removing it would lose it permanently.

---

### 23. 132 Pattern 🟡
```cpp
bool find132pattern(vector<int>& a) {
    stack<int> st;
    int third = INT_MIN;                           // the "2" in the 1-3-2 pattern
    for (int i = a.size() - 1; i >= 0; --i) {      // ⭐ scan right to left
        if (a[i] < third) return true;             // found the "1"
        while (!st.empty() && st.top() < a[i]) { third = st.top(); st.pop(); }
        st.push(a[i]);
    }
    return false;
}
```
**Key insight:** Scanning right-to-left, maintain the largest value that has some larger value to its right. That's the "2"; then any smaller element found later (to the left) is the "1".

---

## C. Queues & Deques

### 24. Implement Queue using Stacks 🟢
```cpp
class MyQueue {
    stack<int> in, out;
    void transfer() { if (out.empty()) while (!in.empty()) { out.push(in.top()); in.pop(); } }
public:
    void push(int x) { in.push(x); }
    int pop()  { transfer(); int v = out.top(); out.pop(); return v; }
    int peek() { transfer(); return out.top(); }
    bool empty() { return in.empty() && out.empty(); }
};
```
**Complexity:** **Amortized O(1)** — each element is moved between stacks at most once.

---

### 25. Implement Stack using Queues 🟢
```cpp
class MyStack {
    queue<int> q;
public:
    void push(int x) {
        q.push(x);
        for (int i = 0; i < (int)q.size() - 1; ++i) { q.push(q.front()); q.pop(); }
    }                                              // ⭐ rotate so the new element is front
    int pop()  { int v = q.front(); q.pop(); return v; }
    int top()  { return q.front(); }
    bool empty() { return q.empty(); }
};
```

---

### 26. Sliding Window Maximum 🔴
```cpp
vector<int> maxSlidingWindow(vector<int>& a, int k) {
    deque<int> dq;                                 // indices, values DECREASING
    vector<int> out;
    for (int i = 0; i < (int)a.size(); ++i) {
        if (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
        dq.push_back(i);
        if (i >= k - 1) out.push_back(a[dq.front()]);
    }
    return out;
}
```

---

### 27. Shortest Subarray with Sum at Least K 🔴
> Works with **negative** numbers, unlike the sliding window version.

```cpp
int shortestSubarray(vector<int>& a, int k) {
    int n = a.size();
    vector<long long> pre(n + 1, 0);
    for (int i = 0; i < n; ++i) pre[i+1] = pre[i] + a[i];

    deque<int> dq;                                 // indices, prefix INCREASING
    int best = n + 1;
    for (int i = 0; i <= n; ++i) {
        while (!dq.empty() && pre[i] - pre[dq.front()] >= k) {
            best = min(best, i - dq.front());
            dq.pop_front();                        // ⭐ can never be better later
        }
        while (!dq.empty() && pre[dq.back()] >= pre[i]) dq.pop_back();  // dominated
        dq.push_back(i);
    }
    return best == n + 1 ? -1 : best;
}
```
**Key insight:** Two invariants. Front-popping: once a prefix satisfies the sum, any later `i` gives a longer subarray, so discard it. Back-popping: if `pre[j] >= pre[i]` with `j < i`, then `j` is dominated — `i` is both a smaller prefix and closer.

---

### 28. Design Circular Queue 🟡
```cpp
class MyCircularQueue {
    vector<int> buf;
    int head = 0, count = 0, cap;
public:
    MyCircularQueue(int k) : buf(k), cap(k) {}
    bool enQueue(int v) {
        if (count == cap) return false;
        buf[(head + count) % cap] = v;             // ⭐ modular indexing
        ++count;
        return true;
    }
    bool deQueue() {
        if (!count) return false;
        head = (head + 1) % cap; --count;
        return true;
    }
    int Front() { return count ? buf[head] : -1; }
    int Rear()  { return count ? buf[(head + count - 1) % cap] : -1; }
    bool isEmpty() { return count == 0; }
    bool isFull()  { return count == cap; }
};
```
**Key insight:** Storing `count` rather than a tail index disambiguates full from empty, which a head/tail-only design cannot.

---

### 29. Design Hit Counter 🟡
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
        if (!q.empty() && q.back().first == t) ++q.back().second;
        else q.push_back({t, 1});
        ++total;
    }
    int getHits(int t) { evict(t); return total; }
};
```

---

### 30. Asteroid Collision 🟡
```cpp
vector<int> asteroidCollision(vector<int>& a) {
    vector<int> st;
    for (int x : a) {
        bool alive = true;
        while (alive && x < 0 && !st.empty() && st.back() > 0) {  // ⭐ collision
            if (st.back() < -x) st.pop_back();                    // stack one explodes
            else if (st.back() == -x) { st.pop_back(); alive = false; }  // both
            else alive = false;                                   // incoming explodes
        }
        if (alive) st.push_back(x);
    }
    return st;
}
```
**Key insight:** A collision only happens when a right-mover (positive, on the stack) meets a left-mover (negative, incoming). Every other pairing never meets.

---

### Bonus: Exclusive Time of Functions 🟡
```cpp
vector<int> exclusiveTime(int n, vector<string>& logs) {
    vector<int> res(n, 0);
    stack<int> st;
    int prev = 0;
    for (auto& log : logs) {
        int p1 = log.find(':'), p2 = log.rfind(':');
        int id = stoi(log.substr(0, p1));
        string type = log.substr(p1 + 1, p2 - p1 - 1);
        int t = stoi(log.substr(p2 + 1));

        if (type == "start") {
            if (!st.empty()) res[st.top()] += t - prev;
            st.push(id);
            prev = t;
        } else {
            res[st.top()] += t - prev + 1;         // ⭐ end is INCLUSIVE
            st.pop();
            prev = t + 1;
        }
    }
    return res;
}
```

---

### Bonus: Basic Calculator III (all operators + parens) 🔴
```cpp
int calculate(string s) {
    int i = 0;
    function<int()> parse = [&]() -> int {
        stack<int> st;
        int num = 0;
        char op = '+';
        while (i < (int)s.size()) {
            char c = s[i++];
            if (isdigit(c)) num = num * 10 + (c - '0');
            else if (c == '(') num = parse();       // ⭐ recurse
            if ((!isdigit(c) && c != ' ') || i == (int)s.size()) {
                if (op == '+') st.push(num);
                else if (op == '-') st.push(-num);
                else if (op == '*') { int t = st.top(); st.pop(); st.push(t * num); }
                else if (op == '/') { int t = st.top(); st.pop(); st.push(t / num); }
                op = c; num = 0;
                if (c == ')') break;                // return to the caller
            }
        }
        int r = 0;
        while (!st.empty()) { r += st.top(); st.pop(); }
        return r;
    };
    return parse();
}
```

---

## 📋 Section Summary

```
╔═══════════════════════════════════════════════════════════════════╗
║               STACKS & QUEUES — PATTERN RECALL                    ║
╠═══════════════════════════════════════════════════════════════════╣
║ MONOTONIC STACK — the highest-value pattern here                  ║
║   "next greater"     L→R, pop while a[top] < a[i]                 ║
║   "next smaller"     L→R, pop while a[top] > a[i]                 ║
║   "previous X"       same, but scan R→L                           ║
║   ⭐ each index pushed once + popped once → O(n)                   ║
║   ⭐ sentinel (0 or -1) flushes the stack at the end               ║
║                                                                   ║
║ APPLICATIONS                                                      ║
║   largest rectangle  → prev smaller + next smaller = the span     ║
║   maximal rectangle  → per-row histogram, reuse the above         ║
║   trapping water     → stack = horizontal layers                  ║
║                        two pointers = vertical columns, O(1) space║
║   sum of subarray min→ count "how many subarrays am I the min of" ║
║   remove k digits    → greedy monotonic non-decreasing stack      ║
║                                                                   ║
║ MONOTONIC DEQUE — sliding window max/min in O(n)                  ║
║   also: shortest subarray sum ≥ k WITH NEGATIVES (prefix+deque)   ║
║                                                                   ║
║ PARENTHESES: stack of indices; -1 sentinel for longest-valid      ║
║ EXPRESSIONS: push additive terms, resolve * and / immediately     ║
║ QUEUE FROM STACKS: two stacks, amortized O(1)                     ║
╚═══════════════════════════════════════════════════════════════════╝
```

**Next:** [Trees →](06-trees.md)
