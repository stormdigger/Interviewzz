# ⌨️ Coding Interview Craft

> Two candidates produce the same working solution. One passes, one doesn't. The difference is almost never the code — it's everything around it.

**Prerequisite:** [DSA Patterns](../04-dsa/00-patterns.md)

---

## 📑 Contents

1. [What's Actually Being Scored](#1-whats-actually-being-scored)
2. [The 45-Minute Structure](#2-the-45-minute-structure)
3. [Clarifying Questions](#3-clarifying-questions)
4. [Thinking Out Loud](#4-thinking-out-loud)
5. [When You're Stuck](#5-when-youre-stuck)
6. [Writing Interview Code](#6-writing-interview-code)
7. [Testing Your Solution](#7-testing-your-solution)
8. [Complexity Analysis Under Pressure](#8-complexity-analysis-under-pressure)
9. [Handling Hints](#9-handling-hints)
10. [Format-Specific Advice](#10-format-specific-advice)
11. [Recovering From Disasters](#11-recovering-from-disasters)
12. [Practice Protocol](#12-practice-protocol)
13. [Cheat Sheet](#13-cheat-sheet)

---

## 1. What's Actually Being Scored

```
   ⭐ MOST INTERVIEWERS SCORE FOUR DIMENSIONS

   ┌──────────────────────────────────────────────────────────────┐
   │ ① PROBLEM SOLVING   Can you get from a vague problem to a    │
   │                     working approach? ⭐ Do you have METHOD,  │
   │                     or do you pattern-match and hope?        │
   ├──────────────────────────────────────────────────────────────┤
   │ ② CODING            Is it correct, clean, and would you want │
   │                     to maintain it?                          │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ ⭐ COMMUNICATION   Can the interviewer FOLLOW your thinking?│
   │                     ⚠️ Silent brilliance scores zero.         │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ ⭐ VERIFICATION    Do you test your own work, or do you     │
   │                     declare victory and wait to be told      │
   │                     you're wrong?                            │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE ASYMMETRY WORTH KNOWING
     A partially-complete solution with excellent communication
     and clear reasoning frequently passes.
     A perfect solution delivered in silence frequently fails —
     because the interviewer cannot distinguish understanding
     from memorization.
```

```
   ⭐ WHAT THE INTERVIEWER IS ACTUALLY THINKING

   "Would I want this person on my team when we're debugging
    something at 2am?"

   That means: can they reason under pressure, do they take
   input well, do they check their work, and are they honest
   when they don't know something.
```

---

## 2. The 45-Minute Structure

```
   ┌──────────────────────────────────────────────────────────────┐
   │  0-5 min    ⭐ CLARIFY                                        │
   │             Understand the problem. Ask questions. State     │
   │             your understanding back.                         │
   ├──────────────────────────────────────────────────────────────┤
   │  5-10 min   ⭐ EXAMPLES + BRUTE FORCE                         │
   │             Work a small example BY HAND. State the brute    │
   │             force with its complexity.                       │
   ├──────────────────────────────────────────────────────────────┤
   │  10-15 min  ⭐ OPTIMIZE + AGREE ON THE APPROACH               │
   │             ⭐⭐ DO NOT WRITE CODE UNTIL THE INTERVIEWER       │
   │             AGREES WITH YOUR APPROACH.                       │
   ├──────────────────────────────────────────────────────────────┤
   │  15-35 min  CODE                                             │
   │             Narrate while writing. Clean names. Handle edges.│
   ├──────────────────────────────────────────────────────────────┤
   │  35-42 min  ⭐ TEST                                           │
   │             Trace a real example. Then edge cases.           │
   ├──────────────────────────────────────────────────────────────┤
   │  42-45 min  COMPLEXITY + IMPROVEMENTS + their questions      │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ THE MOST COMMON TIME-MANAGEMENT FAILURE
     Writing code at minute 3. You solve the wrong problem, or
     the wrong version of the right problem, and discover it at
     minute 30 with no time to recover.

   ⭐ Five minutes of clarification saves twenty minutes of
     rework. This is the single highest-ROI habit in coding
     interviews.
```

---

## 3. Clarifying Questions

```
   ⭐ THE STANDARD SET — ask these almost every time

   INPUT
   □ What's the size range? (⭐ this tells you the target
     complexity — see the constraints table below)
   □ Can it be empty? Can it be null?
   □ Are values sorted? Unique? Can they be negative? Zero?
   □ What types — integers, floats, unicode strings?
   □ ⭐ Can I assume the input is valid, or should I validate?

   OUTPUT
   □ What exactly should I return — an index, a value, a count?
   □ ⭐ If there are multiple valid answers, does it matter which?
   □ What should I return when there's no answer?
   □ Does the output order matter?

   CONSTRAINTS
   □ ⭐ Any memory constraints? Can I modify the input in place?
   □ Is there a time expectation?
   □ Can I use built-in library functions? (⭐ e.g. sort)
```

```
   ⭐⭐ CONSTRAINTS TELL YOU THE ANSWER

   n ≤ 20            → O(2ⁿ) — subsets, bitmask, backtracking
   n ≤ 500           → O(n³)
   n ≤ 5,000         → O(n²) — DP is likely
   n ≤ 100,000       → ⭐ O(n log n) — sort, heap, binary search
   n ≤ 1,000,000     → O(n) or O(n log n)
   n > 10⁹           → O(log n) or O(1) — binary search on the
                        answer, or math

   ⭐ SAY THIS OUT LOUD:
     "n is up to 10⁵, so O(n²) won't pass — I need O(n log n)
      or better. That points me toward sorting, a heap, or
      binary search."

   ⭐ This single sentence demonstrates that you reason from
     constraints rather than guessing, and it's rare enough to
     stand out.
```

```
   ⭐ CLOSE THE CLARIFICATION PHASE BY RESTATING

   "So to confirm: I'm given an unsorted array of up to 10⁵
    integers which may be negative, and I need to return the
    INDICES of two numbers summing to the target. There's
    exactly one solution, and I can't reuse the same element
    twice. Is that right?"

   ⭐ This catches misunderstandings immediately and shows you
     listen carefully. It takes fifteen seconds.
```

---

## 4. Thinking Out Loud

```
   ⭐⭐ THE SINGLE HIGHEST-LEVERAGE HABIT IN THE ENTIRE INTERVIEW.

   The interviewer cannot give credit for thoughts they can't
   hear, and cannot redirect you if they don't know where
   you're going.
```

```
   ⭐ NARRATION PHRASES THAT WORK

   ── EXPLORING ──────────────────────────────────────────────────
   "Let me start with the brute force so we have a baseline."
   "The brute force is O(n²) because I'm checking every pair."
   "⭐ The thing this is redoing is... which suggests I could
    cache it / use a hash map / sort first."
   "This feels like a sliding window problem because we're
    looking for a contiguous subarray."

   ── DECIDING ───────────────────────────────────────────────────
   "There are two approaches here — let me lay out both."
   "⭐ I'll go with the hash map. It's O(n) time at the cost of
    O(n) space, which seems right given the constraints."
   "Does that approach sound reasonable before I start coding?"

   ── WHILE CODING ───────────────────────────────────────────────
   "I'm using a dictionary from value to index so I can look up
    the complement in constant time."
   "⭐ I'll handle the empty input case up front."
   "I'm inserting AFTER checking, so I don't match an element
    with itself."

   ── WHEN STUCK ─────────────────────────────────────────────────
   "⭐ Let me think about this for a moment." (⭐ then actually
    think — a short deliberate silence is fine)
   "I'm not immediately seeing the optimization. Let me try a
    concrete example and look for structure."
   "⭐ I know a sorted array would let me use two pointers here —
    let me consider whether sorting is affordable."
```

```
   ⚠️ THE SILENT-THINKING PROBLEM

   If you genuinely need to think quietly, SAY SO:
     "Give me twenty seconds to work through this."

   ⭐ That's completely fine. What isn't fine is two minutes of
     silence where the interviewer has no idea whether you're
     stuck, confused, or nearly done.
```

---

## 5. When You're Stuck

```
   ⭐ THE ESCALATION LADDER — work down it out loud

   ┌──────────────────────────────────────────────────────────────┐
   │ 1. ⭐ WORK A CONCRETE EXAMPLE BY HAND                         │
   │    More problems are solved this way than by any other       │
   │    method. Patterns become visible when you stop thinking    │
   │    abstractly.                                               │
   ├──────────────────────────────────────────────────────────────┤
   │ 2. ⭐ SOLVE A SIMPLER VERSION                                 │
   │    Smaller n · only positive numbers · sorted input ·        │
   │    return a boolean instead of the actual answer.            │
   │    ⭐ Then ask what changes when you add the constraint back.│
   ├──────────────────────────────────────────────────────────────┤
   │ 3. ⭐ ASK "WHAT IS THE BRUTE FORCE REDOING?"                  │
   │    recomputation      → memoize / DP                         │
   │    repeated searching → hash map / sort / binary search      │
   │    repeated scanning  → two pointers / sliding window        │
   │    repeated min/max   → heap / monotonic stack               │
   ├──────────────────────────────────────────────────────────────┤
   │ 4. WALK THE PATTERN LIST                                     │
   │    Two pointers · sliding window · hash map · sort · binary  │
   │    search · heap · stack · BFS/DFS · DP · greedy · trie      │
   ├──────────────────────────────────────────────────────────────┤
   │ 5. ⭐ CONSIDER A DIFFERENT DATA STRUCTURE                     │
   │    "What if this were sorted?" "What if I had O(1) lookup?"  │
   │    "What if I processed it backwards?"                       │
   ├──────────────────────────────────────────────────────────────┤
   │ 6. ⭐ ASK FOR A HINT — explicitly and gracefully              │
   │    "I've been going down the sorting path and it's not       │
   │     leading anywhere. Am I on the right track, or should I   │
   │     be thinking about this differently?"                     │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ ASKING FOR A HINT IS NOT FAILURE.

   ⚠️ Burning fifteen minutes in silence on a dead end IS.

   Interviewers expect to give hints — many problems are
   calibrated assuming one. What they're scoring is how you
   USE the hint: do you immediately see where it leads, or do
   you need to be walked through it step by step?

   ⭐ The strong move: name what you've tried and why it isn't
     working, then ask a specific question. That shows the
     thinking behind the stuckness.
```

```
   ⭐ IF YOU GENUINELY CANNOT FIND THE OPTIMAL SOLUTION

   ⭐ CODE THE BRUTE FORCE. A working suboptimal solution
     scores far better than an incomplete optimal one.

   Say: "I'll implement the O(n²) version so we have something
    correct, and then talk through how I'd optimize it if
    there's time."

   ⭐ That's a professional, honest response and interviewers
     respect it.
```

---

## 6. Writing Interview Code

```
   ⭐ THE STANDARD IS "CODE YOU'D SUBMIT FOR REVIEW" —
     not competitive-programming golf.

   ✅ DO
   □ ⭐ Descriptive names: `left`, `right`, `seen`, `maxLength`
     — not `i`, `j`, `tmp`, `x`
   □ Small helper functions for distinct logical steps
   □ ⭐ Handle edge cases explicitly and visibly at the top
   □ Use language idioms the interviewer will recognize
   □ Consistent formatting
   □ ⭐ A brief comment for a non-obvious line, not for obvious ones

   ❌ DON'T
   □ ⚠️ Single-letter names beyond a loop index
   □ ⚠️ Clever one-liners that need explaining
   □ Premature abstraction — no class hierarchy for a 20-line
     function
   □ Copy-pasted blocks — extract instead
   □ ⚠️ Silent assumptions (⭐ if you assume input is valid, SAY SO)
```

```cpp
// ⭐ WHAT GOOD INTERVIEW CODE LOOKS LIKE
vector<int> twoSum(const vector<int>& nums, int target) {
    // Map from value to the index where we saw it
    unordered_map<int, int> valueToIndex;

    for (int i = 0; i < (int)nums.size(); ++i) {
        int complement = target - nums[i];

        auto it = valueToIndex.find(complement);
        if (it != valueToIndex.end()) {
            return {it->second, i};
        }

        // Insert AFTER checking so we don't pair an element
        // with itself when target == 2 * nums[i]
        valueToIndex[nums[i]] = i;
    }

    return {};   // no valid pair
}
```

```
   ⭐ WHY THAT CODE READS WELL
     • `valueToIndex` and `complement` explain themselves
     • The one non-obvious decision (insert after check) has a
       comment explaining WHY, not what
     • The no-answer case is handled explicitly
     • No cleverness that requires explanation
```

```
   ⭐ HANDLING EDGE CASES VISIBLY

   Write the guards first, so the interviewer sees you thought
   about them:

   if (nums.empty()) return {};
   if (k <= 0 || k > (int)nums.size()) return {};

   ⭐ And say it: "Let me handle the empty case up front."
     This converts an implicit assumption into visible rigour.
```

---

## 7. Testing Your Solution

```
   ⭐⭐ TESTING IS A SCORED DIMENSION, NOT A FORMALITY.

   ⚠️ Finishing the code and saying "done" leaves points on the
     table. Finding your own bug before the interviewer does is
     one of the strongest signals available.
```

```
   ⭐ THE TESTING SEQUENCE

   1. ⭐ TRACE THE EXAMPLE YOU DISCUSSED EARLIER
      Line by line, with actual values, out loud. Write the
      variable state down as you go.
      ⭐ This finds most bugs.

   2. EDGE CASES — say each one and check it
      □ Empty input
      □ Single element
      □ Two elements
      □ All identical values
      □ Already sorted / reverse sorted
      □ Negatives, zero
      □ ⭐ Target not present / no valid answer
      □ Maximum size (⚠️ overflow? stack depth?)
      □ Duplicates

   3. ⭐ CHECK THE BOUNDARIES SPECIFICALLY
      □ Loop conditions: `<` vs `<=`
      □ Off-by-one on indices
      □ ⭐ Integer overflow: `mid = lo + (hi - lo) / 2`
      □ Did I update every variable I intended to?
      □ Does every path return something?
```

```
   ⭐ WHAT TO SAY WHILE TESTING

   "Let me trace through [1, 3, 5, 7] with target 8.
    i=0: complement is 8-1=7, not in the map. Insert 1→0.
    i=1: complement is 8-3=5, not in the map. Insert 3→1.
    i=2: complement is 8-5=3, which IS in the map at index 1.
         Return [1, 2]. ⭐ That's correct — nums[1] + nums[2]
         = 3 + 5 = 8."

   ⭐ Then: "Now the edge cases. Empty input returns an empty
     vector at the loop. Single element — the loop runs once,
     finds nothing, returns empty. Both correct."
```

```
   ⭐ IF YOU FIND A BUG — this is a GOOD moment

   Say clearly: "I found a bug — when the array is empty this
   would... let me fix that."

   ⚠️ Do NOT silently fix it and hope nobody noticed. Announcing
     it demonstrates exactly the verification instinct being
     scored.
```

---

## 8. Complexity Analysis Under Pressure

```
   ⭐ STATE BOTH TIME AND SPACE, AND EXPLAIN WHY

   ❌ "It's O(n)."
   ✅ "Time is O(n) — one pass through the array, and each hash
      map operation is O(1) on average. Space is O(n) for the
      map in the worst case, where no pair is found until the
      end."
```

```
   ⭐ THE THINGS PEOPLE FORGET

   □ ⭐ RECURSION STACK counts toward space complexity
     (DFS on a path graph of n nodes is O(n) space)
   □ ⭐ The output usually doesn't count, but say so explicitly
   □ Sorting is O(n log n) time and O(log n) to O(n) space
     depending on the algorithm
   □ Hash operations are O(1) AVERAGE — worst case O(n)
   □ ⭐ Slicing or substring operations often copy: `s[1:]` in
     Python is O(n), not O(1). This turns an O(n) algorithm
     into O(n²) invisibly.
   □ String concatenation in a loop is O(n²) in most languages
```

```
   ⭐ IF ASKED "CAN YOU DO BETTER?"

   ⭐ Don't just say yes or no. Reason about the LOWER BOUND:

   "We have to look at every element at least once to know the
    answer, so O(n) is a lower bound on time. We're at O(n),
    so that's optimal.

    ⭐ Space is where there's room — we're using O(n) for the
    hash map. If the array were sorted we could use two
    pointers for O(1) space. Sorting costs O(n log n) though,
    so it's only worth it if space matters more than time or
    the input is already sorted."

   ⭐ That answer demonstrates you understand the tradeoff
     space, not just this problem.
```

---

## 9. Handling Hints

```
   ⭐⭐ A HINT IS NOT A CRITICISM. IT'S A RESCUE ATTEMPT — AND
     A TEST OF HOW YOU RECEIVE INPUT.

   ⚠️ THE FATAL RESPONSE: ignoring it and continuing on your
     path. That reads as not listening, which is far worse than
     being stuck.
```

```
   ⭐ HOW TO RECEIVE A HINT WELL

   1. ⭐ STOP TALKING and listen fully
   2. Acknowledge it explicitly: "That's a good point, let me
      think about that"
   3. ⭐ Say what it makes you think: "So if I sorted first,
      then I could use two pointers and drop to O(1) space..."
   4. Follow it. Even if you're not sure yet — the interviewer
      knows where it leads.

   ⭐ RECOGNIZING A HINT: any question the interviewer asks
     unprompted is almost always a hint.
       "What happens if the array has duplicates?"
       "Have you considered what the constraint on n implies?"
       "Is there anything special about the input being sorted?"

     ⚠️ These are never idle curiosity. Treat every one as a
       signpost.
```

```
   ⚠️ IF YOU DISAGREE WITH A HINT

   Engage rather than dismiss:
     "I want to make sure I understand — are you suggesting I
      sort first? My concern is that would be O(n log n) and
      I think we can do it in O(n) with a hash map. Is there
      something about the sorted approach I'm missing?"

   ⭐ Sometimes you're right. Sometimes there's a constraint you
     missed. Either way, engaging respectfully scores well;
     ignoring scores badly.
```

---

## 10. Format-Specific Advice

```
   ⭐ WHITEBOARD / NO IDE
     • Write LARGER and SPACED — you'll need to insert lines
     • ⭐ Leave a blank line between logical sections
     • Use the top corner for examples and variable state
     • Syntax errors are forgiven; ⭐ logic errors are not
     • Say "I'd double-check this signature in a real editor"
       rather than agonizing over exact API details

   ⭐ SHARED EDITOR (no execution)
     • Same rules; you can move faster
     • ⭐ Still trace manually — nothing runs
     • Keep the code visible without scrolling if possible

   ⭐ WITH EXECUTION (CoderPad, HackerRank)
     • ⭐ RUN IT EARLY and often, not just at the end
     • Write a few test cases first if you're comfortable
     • ⚠️ Don't let "make the test pass" replace understanding —
       explain WHY it failed before fixing

   ⭐ TAKE-HOME
     • ⭐ Read the ENTIRE brief before writing anything
     • Ask clarifying questions by email — this is scored too
     • Write a README: how to run, decisions made, tradeoffs,
       ⭐ what you'd do with more time
     • ⭐ TESTS. A take-home without tests reads as careless.
     • Respect the stated time box, and say what you cut
     • Commit incrementally with clear messages

   ⭐ PAIR PROGRAMMING WITH AN ENGINEER
     • Treat them as a colleague, not an examiner
     • ⭐ Ask their opinion: "would you handle this case here or
       validate earlier?"
     • ⭐ This format explicitly rewards collaboration — use it
```

---

## 11. Recovering From Disasters

```
   ⭐ "I'VE SEEN THIS PROBLEM BEFORE"

   ⭐ SAY SO IMMEDIATELY. "I've seen this one — should I go
     ahead, or would you prefer a different problem?"

   ⚠️ Pretending is transparent and it destroys trust. Most
     interviewers will say go ahead, and then probe your
     understanding harder — which you can handle if you
     genuinely understand it.
```

```
   ⭐ "I'M 30 MINUTES IN WITH BROKEN CODE"

   1. ⭐ Stop and assess out loud: "This approach is getting
      complicated. Let me step back."
   2. Is it a small bug or a wrong approach?
   3. ⭐ If the approach is wrong, say so and pivot. Sunk cost
      is not an argument.
      "I think the sliding window doesn't handle negatives.
       Let me switch to prefix sums with a hash map."
   4. If time is short, describe the correct approach clearly
      even if you can't finish coding it.

   ⭐ Recognizing and articulating your own wrong turn is a
     positive signal. Grinding silently on a doomed approach
     is not.
```

```
   ⭐ "I COMPLETELY BLANKED"

   • Say it honestly: "I've lost my thread for a second, let
     me re-read the problem."
   • ⭐ Go back to a concrete example. It restarts the reasoning.
   • Restate what you know and what you need.
   • ⚠️ Do not apologize repeatedly — it burns time and
     amplifies the impression.
```

```
   ⭐ "THE INTERVIEWER SEEMS UNIMPRESSED"

   ⚠️ You are almost certainly misreading them. Many
     interviewers are deliberately neutral, distracted by
     note-taking, or simply have a flat affect.

   ⭐ Don't let a perceived reaction change your behaviour.
     Keep executing your process.
```

```
   ⭐ "I FINISHED WITH 15 MINUTES LEFT"

   ⭐ Don't sit in silence. Use it:
     • Test more thoroughly, including larger edge cases
     • "Let me think about how this would change if the input
       didn't fit in memory."
     • "If this were production code I'd add input validation
       and handle the concurrent-access case."
     • ⭐ "Is there a follow-up you'd like to explore?"

   ⭐ Extending the problem yourself is a senior signal.
```

---

## 12. Practice Protocol

```
   ⭐ PRACTICE THE INTERVIEW, NOT JUST THE PROBLEM.

   ⚠️ Solving 300 problems silently in an IDE trains a
     different skill from the one being tested.

   ⭐ THE SESSION FORMAT
     □ Set a 45-minute timer
     □ ⭐ TALK OUT LOUD the entire time — record yourself
     □ ⭐ No looking anything up during the session
     □ Write code in a plain editor with no autocomplete
     □ Test by hand before running anything
     □ ⭐ Afterwards, listen to the recording

   ⭐ WHAT THE RECORDING REVEALS (and nothing else does)
     • Long silences you didn't notice
     • Claims made without justification
     • The exact moment you got stuck and what you did
     • Whether you actually stated complexity
     • Filler and rambling
```

```
   ⭐ INTERLEAVE, DON'T BLOCK

   ⚠️ Doing 40 array problems in a row teaches "array chapter →
     two pointers" rather than "this problem's shape → two
     pointers." Real interviews give you unlabelled problems.

   ⭐ Mix at least three topics per session. It feels worse and
     transfers far better.
```

```
   ⭐ THE POST-PROBLEM RITUAL — more valuable than the problem

   After every problem, whether you solved it or not:
     □ ⭐ What PATTERN was this? Could I recognize it unlabelled?
     □ What was the KEY INSIGHT in one sentence?
     □ ⭐ If I failed: was it the pattern, the implementation,
       or an edge case? Those need different remedies.
     □ Add it to your tracker with the pattern tag
     □ ⭐ Schedule a re-test in 7 days

   ⭐ Tracking WHICH PATTERN YOU MISSED — not just pass/fail —
     is the data that actually improves you.
     → [Progress Tracker](../00-meta/progress-tracker.md#dsa-solve-log)
```

```
   ⭐ MOCK INTERVIEWS

   Do at least 5 before a real loop. With a person, not alone.

   ⭐ Ask your mock interviewer to:
     • Interrupt with questions
     • Give a hint at some point and see how you use it
     • ⭐ Push back on one decision even if it's correct
     • Score you on communication separately from correctness
```

---

## 13. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║               CODING INTERVIEW CRAFT — ONE PAGE                      ║
╠══════════════════════════════════════════════════════════════════════╣
║ SCORED: problem solving · coding · ⭐ COMMUNICATION · ⭐ VERIFICATION  ║
║ ⭐ Partial + great communication PASSES.                              ║
║   Perfect + silence often FAILS.                                     ║
╠══════════════════════════════════════════════════════════════════════╣
║ STRUCTURE  clarify(5) → examples+brute force(5) → optimize(5) →      ║
║   code(20) → ⭐ TEST(7) → complexity(3)                               ║
║ ⭐⭐ DON'T WRITE CODE UNTIL THE INTERVIEWER AGREES WITH THE APPROACH   ║
║ ⭐ 5 min of clarifying saves 20 min of rework                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ CONSTRAINTS TELL YOU THE ANSWER — say it out loud                 ║
║   n≤20→2ⁿ · n≤5k→n² · ⭐ n≤10⁵→n log n · n≤10⁶→n · n>10⁹→log n        ║
║ ⭐ Restate the problem back before starting.                          ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ NARRATE CONTINUOUSLY. Unspoken thoughts score zero.               ║
║   Need to think? SAY "give me 20 seconds" — silence is fine when     ║
║   announced, fatal when not.                                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ STUCK LADDER: ⭐ work an example BY HAND → solve a simpler version →  ║
║   ⭐ "what is the brute force REDOING?" → walk the pattern list →     ║
║   different data structure → ⭐ ASK FOR A HINT (not failure —         ║
║   15 silent minutes IS)                                              ║
║ ⭐ Can't find optimal? CODE THE BRUTE FORCE. Working beats            ║
║   incomplete-optimal.                                                ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ EVERY UNPROMPTED QUESTION FROM THE INTERVIEWER IS A HINT.          ║
║   Stop, acknowledge, say where it leads, follow it.                  ║
║   ⚠️ Ignoring a hint is worse than being stuck.                       ║
╠══════════════════════════════════════════════════════════════════════╣
║ CODE: descriptive names · explicit edge guards FIRST · comment the   ║
║   WHY not the what · ⚠️ no clever one-liners                          ║
║ ⭐ TEST: trace the example line by line OUT LOUD, then edge cases     ║
║   (empty, single, duplicates, negatives, no-answer, overflow)        ║
║   ⭐ Finding your OWN bug is one of the strongest signals available.  ║
╠══════════════════════════════════════════════════════════════════════╣
║ COMPLEXITY: state time AND space, ⭐ and WHY.                         ║
║   ⚠️ forgotten: recursion stack · slicing copies (O(n)!) ·            ║
║     string concat in a loop · hash is O(1) AVERAGE                   ║
║   "Can you do better?" → reason about the LOWER BOUND                ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ SEEN IT BEFORE? SAY SO IMMEDIATELY.                                ║
║ ⭐ Wrong approach at min 30? Say so and PIVOT — sunk cost isn't an    ║
║   argument. Articulating your own wrong turn is a POSITIVE signal.   ║
║ ⭐ PRACTICE OUT LOUD, TIMED, RECORDED. Interleave topics.            ║
║   Track WHICH PATTERN you missed, not just pass/fail.                ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Negotiation →](negotiation.md) · **Related:** [DSA Patterns](../04-dsa/00-patterns.md) · [System Design Framework](../05-system-design/02-framework.md) · [Behavioral](behavioral.md)
