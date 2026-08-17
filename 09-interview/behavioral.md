# 🗣️ Behavioral Interviews

> The behavioral round is not a personality test. It's an evidence-gathering exercise: *has this person actually done the things the role requires, and what will they be like to work with when something goes wrong?*

---

## 📑 Contents

1. [What's Actually Being Measured](#1-whats-actually-being-measured)
2. [The STAR Framework](#2-the-star-framework)
3. [Building Your Story Bank](#3-building-your-story-bank)
4. [The 12 Core Stories](#4-the-12-core-stories)
5. [60 Questions by Category](#5-60-questions-by-category)
6. [Amazon Leadership Principles](#6-amazon-leadership-principles)
7. [Hard Questions](#7-hard-questions)
8. [Level Calibration](#8-level-calibration)
9. [Common Failure Modes](#9-common-failure-modes)
10. [Questions to Ask Them](#10-questions-to-ask-them)
11. [Preparation Protocol](#11-preparation-protocol)
12. [Cheat Sheet](#12-cheat-sheet)

---

## 1. What's Actually Being Measured

```
   ⭐ FOUR THINGS, IN EVERY QUESTION

   ┌──────────────────────────────────────────────────────────────┐
   │ ① IMPACT        Did you actually change an outcome, or were  │
   │                 you present while it happened?               │
   │                 ⭐ "We" is a red flag when it never becomes   │
   │                   "I did this specific thing."               │
   ├──────────────────────────────────────────────────────────────┤
   │ ② JUDGMENT      Did you make good decisions with incomplete  │
   │                 information? Can you articulate the tradeoff?│
   ├──────────────────────────────────────────────────────────────┤
   │ ③ COLLABORATION How do you behave when you disagree, when    │
   │                 you're wrong, or when someone else is?       │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ ⭐ GROWTH      What did you learn? Do you update your        │
   │                 behaviour, or repeat the same mistakes with  │
   │                 better vocabulary?                           │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE INTERVIEWER IS BUILDING A PREDICTION

   Past behaviour is the best available predictor of future
   behaviour. So they want SPECIFIC PAST EVENTS, not
   philosophy.

   ⚠️ "I believe in giving direct feedback" tells them nothing.
   ✅ "Last March I told a senior engineer his design had a
      race condition in front of the team. Here's how that
      went and what I'd do differently."

   ⭐ HYPOTHETICALS ARE A TRAP. If asked "what would you do
     if...", answer with "here's what I actually did in a
     similar situation" whenever you can.
```

---

## 2. The STAR Framework

```
   ⭐ SITUATION   Context. Brief — 2-3 sentences maximum.
                 Just enough for the rest to make sense.
   ⭐ TASK        Your specific responsibility. ⭐ What was YOURS,
                 as distinct from the team's?
   ⭐⭐ ACTION     What YOU did, step by step. ⭐ This is 60% of
                 the answer. Use "I", not "we".
   ⭐ RESULT      The outcome, ⭐ QUANTIFIED where possible, plus
                 what you learned.

   ┌──────────────────────────────────────────────────────────────┐
   │  TIME BUDGET for a 2-3 minute answer                         │
   │                                                              │
   │  Situation  ████                    15%                      │
   │  Task       ███                     10%                      │
   │  Action     ██████████████████████  60%   ⭐ the substance    │
   │  Result     ████                    15%                      │
   └──────────────────────────────────────────────────────────────┘

   ⚠️ THE MOST COMMON FAILURE: three minutes of context and
     twenty seconds of what you did. The interviewer learns
     nothing about you.
```

```
   ⭐ THE "AND WHAT I LEARNED" EXTENSION

   Strong answers add a fifth beat: what you'd do differently,
   or how it changed your approach afterwards.

   ⭐ This signals self-awareness and growth, which is exactly
     what dimension ④ measures — and most candidates skip it.
```

### A worked example

```
   ❌ WEAK
   "We had performance problems so we optimized the database
    and things got better."

   ✅ STRONG
   S: "Our checkout API's p99 latency had degraded from 200ms
       to 2 seconds over about six weeks, and support tickets
       about failed checkouts were climbing."

   T: "I owned the checkout service. My job was to find the
       cause and fix it without pausing the feature roadmap."

   A: "I started with traces rather than guessing, and found
       most of the time was in one database call. The query
       plan showed a sequential scan — a migration two months
       earlier had added a column to the WHERE clause but not
       to the index.

       Rather than just adding the index, I checked whether
       this had happened elsewhere. It had, in two other
       services. So I did three things: added the index with
       CREATE INDEX CONCURRENTLY to avoid locking, added an
       alert on p99 latency per endpoint since we'd only been
       alerting on errors, and added a CI check that flags
       migrations changing columns used in existing queries."

   R: "p99 went back to 180ms within an hour of the index
       being built. Checkout failure tickets dropped to
       roughly zero the next day.

       ⭐ What I took from it was that the missing index was
       the symptom — the real problem was that we had no way
       to notice gradual degradation. Latency alerting caught
       two similar regressions in the following quarter before
       users did."
```

```
   ⭐ WHY THAT VERSION WORKS
     • Specific numbers, so the impact is verifiable
     • Shows METHOD (traces before guessing), not just outcome
     • ⭐ Goes beyond the immediate fix to the systemic cause
     • Mentions an operational detail (CONCURRENTLY) that
       demonstrates real experience
     • The learning is concrete and had a measurable follow-on
```

---

## 3. Building Your Story Bank

```
   ⭐ YOU NEED 8-12 STORIES, NOT 40.

   Well-prepared stories can each answer several questions
   from different angles. The goal is depth and flexibility,
   not coverage of every possible prompt.

   ⭐ THE MATRIX APPROACH
     List your last 2-3 years of significant work. For each,
     identify which of these it demonstrates. A story covering
     four themes is worth more than four shallow stories.
```

```
   ⭐ FOR EACH STORY, WRITE DOWN (this is the actual prep work)

   □ The one-line summary
   □ ⭐ The numbers — before and after, scale, duration, team size
   □ ⭐ The hardest decision you made and the alternative you
     rejected (⭐ interviewers probe here; have the answer ready)
   □ What went wrong or what you'd do differently
   □ Who you worked with and how you influenced them
   □ ⭐ Which themes it can cover

   ⚠️ Write these out. Stories you've only thought about
     collapse under follow-up questions. Stories you've
     written and said out loud hold up.
```

---

## 4. The 12 Core Stories

```
   ⭐ THESE COVER THE OVERWHELMING MAJORITY OF QUESTIONS

   ┌──────────────────────────────────────────────────────────────┐
   │  1. ⭐ HARDEST TECHNICAL PROBLEM                              │
   │     Depth, method, persistence                               │
   ├──────────────────────────────────────────────────────────────┤
   │  2. ⭐ A TIME YOU FAILED                                      │
   │     ⭐ Must be a REAL failure with real consequences, and     │
   │       a genuine lesson. This question separates people.      │
   ├──────────────────────────────────────────────────────────────┤
   │  3. CONFLICT WITH A COLLEAGUE                                │
   │     ⭐ How you disagree AND how you resolve it                │
   ├──────────────────────────────────────────────────────────────┤
   │  4. ⭐ INFLUENCING WITHOUT AUTHORITY                          │
   │     ⭐ The single most important story for senior+ roles      │
   ├──────────────────────────────────────────────────────────────┤
   │  5. LEADING A PROJECT                                        │
   │     Planning, delegation, risk, delivery                     │
   ├──────────────────────────────────────────────────────────────┤
   │  6. ⭐ A DIFFICULT TRADEOFF                                   │
   │     Judgment under real constraints                          │
   ├──────────────────────────────────────────────────────────────┤
   │  7. MENTORING SOMEONE                                        │
   │     Growing others; essential above mid-level                │
   ├──────────────────────────────────────────────────────────────┤
   │  8. ⭐ HANDLING AN INCIDENT                                   │
   │     Composure, prioritization, follow-through                │
   ├──────────────────────────────────────────────────────────────┤
   │  9. AMBIGUOUS PROBLEM                                        │
   │     Scoping something with no clear definition               │
   ├──────────────────────────────────────────────────────────────┤
   │ 10. ⭐ PUSHING BACK ON A DECISION                             │
   │     Judgment plus courage plus knowing when to commit        │
   ├──────────────────────────────────────────────────────────────┤
   │ 11. LEARNING SOMETHING QUICKLY                               │
   │     Adaptability and learning method                         │
   ├──────────────────────────────────────────────────────────────┤
   │ 12. ⭐ SOMETHING YOU'RE GENUINELY PROUD OF                    │
   │     Reveals what you actually value                          │
   └──────────────────────────────────────────────────────────────┘
```

---

## 5. 60 Questions by Category

### A. Technical depth (1–10)

```
   1.  Tell me about the hardest technical problem you've solved.
   2.  Describe a system you designed. What would you change now?
   3.  Tell me about a bug that took a long time to find.
   4.  When did you have to make a significant technical tradeoff?
   5.  Describe a time you improved performance substantially.
   6.  Tell me about technical debt you addressed — or chose not to.
   7.  When did you have to learn a technology quickly?
   8.  Describe a code review where you found something important.
   9.  Tell me about a time you disagreed with an architectural decision.
   10. What's the most complex system you've worked on? What made it complex?
```

```
   ⭐ FOR TECHNICAL DEPTH QUESTIONS, ALWAYS INCLUDE
     • ⭐ Your METHOD — how you narrowed it down, not just the
       answer. Anyone can describe a fix; describing how you
       found it is what demonstrates skill.
     • The alternatives you considered and why you rejected them
     • ⭐ What you'd do differently now
```

### B. Failure and mistakes (11–18)

```
   11. Tell me about a time you failed.
   12. Describe a mistake that had real consequences.
   13. When did you miss a deadline? What happened?
   14. Tell me about a time you were wrong about something technical.
   15. Describe a project that didn't go as planned.
   16. When did you receive difficult feedback? How did you respond?
   17. Tell me about a decision you regret.
   18. When did you have to admit you didn't know something?
```

```
   ⭐⭐ THE FAILURE QUESTION IS THE MOST DIAGNOSTIC ONE ASKED.

   ⚠️ THE ANSWERS THAT FAIL
     • "I work too hard" or any disguised strength
     • A failure that was entirely someone else's fault
     • A trivial failure with no consequences
     • ⚠️ A failure with no lesson, or a lesson so generic it's
       meaningless ("I learned to communicate better")

   ⭐ WHAT A GOOD FAILURE STORY HAS
     • Real consequences — the project slipped, the outage
       happened, the customer was affected
     • ⭐ YOUR ownership, clearly stated, without excessive
       self-flagellation
     • A specific, concrete lesson
     • ⭐ EVIDENCE you applied it afterwards

   ⭐ The last point is what separates good from great. "I
     learned X" is a claim. "I learned X, and here's what I did
     differently three months later" is evidence.
```

### C. Collaboration and conflict (19–30)

```
   19. Tell me about a conflict with a teammate.
   20. Describe working with someone difficult.
   21. When did you have to influence without authority?
   22. Tell me about giving difficult feedback.
   23. Describe a time you changed your mind after hearing another view.
   24. When did you have to work with an uncooperative team?
   25. Tell me about a disagreement with your manager.
   26. Describe collaborating across teams or functions.
   27. When did you have to say no to a stakeholder?
   28. Tell me about mentoring someone.
   29. Describe a time a teammate was underperforming.
   30. How have you handled a toxic dynamic on a team?
```

```
   ⭐ THE CONFLICT QUESTION — what they're actually testing

   Not whether you avoid conflict. ⭐ Whether you can DISAGREE
   PRODUCTIVELY and then COMMIT to the outcome.

   ⭐ THE STRUCTURE THAT WORKS
     1. State the disagreement neutrally, and represent the
        other person's position fairly (⭐ this alone
        distinguishes you — most candidates make the other
        person sound unreasonable)
     2. Explain how you sought to understand their reasoning
     3. Describe how you made your case — ⭐ ideally with data
        or a prototype rather than argument
     4. State the outcome honestly, including if you lost
     5. ⭐ Describe how you committed to the decision either way

   ⭐ "I disagreed, made my case with data, was overruled,
     committed fully, and it turned out they were right about
     X" is a STRONGER answer than winning — it demonstrates
     judgment, humility, and professionalism simultaneously.
```

### D. Leadership and ownership (31–42)

```
   31. Tell me about a project you led end to end.
   32. Describe taking ownership of something outside your remit.
   33. When did you identify a problem nobody else had noticed?
   34. Tell me about driving a decision across multiple teams.
   35. Describe improving a process.
   36. When did you have to make a decision with incomplete information?
   37. Tell me about prioritizing under heavy constraints.
   38. Describe delivering under significant pressure.
   39. When did you push back on a requirement?
   40. Tell me about a time you raised the bar for the team.
   41. Describe onboarding or ramping up a new team member.
   42. When did you have to deliver bad news?
```

### E. Ambiguity and problem solving (43–50)

```
   43. Tell me about the most ambiguous problem you've handled.
   44. Describe a time requirements changed mid-project.
   45. When did you have to define a problem before solving it?
   46. Tell me about making a decision without enough data.
   47. Describe a time you simplified something complex.
   48. When did you decide NOT to build something?
   49. Tell me about balancing speed against quality.
   50. Describe scoping a project down to hit a deadline.
```

```
   ⭐ QUESTION 48 IS UNDERRATED — the decision NOT to build.

   Senior engineers are distinguished partly by what they
   choose not to do. A story about killing a project,
   rejecting a feature, or choosing a boring solution over an
   interesting one demonstrates judgment that building stories
   cannot.
```

### F. Motivation and culture (51–60)

```
   51. Why are you leaving your current role?
   52. Why this company?
   53. What are you looking for in your next role?
   54. What's the best team you've worked on? What made it good?
   55. How do you stay current technically?
   56. What kind of work energizes you?
   57. Where do you want to be in three years?
   58. What's your biggest strength? Biggest weakness?
   59. What would your last manager say about you?
   60. Tell me about something you're proud of.
```

```
   ⚠️ "WHY ARE YOU LEAVING?" — the one that trips people up

   ⭐ NEVER criticize your current employer, manager, or
     colleagues. Even when justified, it reads as a preview of
     how you'll talk about THEM.

   ⭐ THE FRAME THAT WORKS: what you're moving TOWARD.
     "I've learned a lot about X and I'm looking for more
      depth in Y, which this role has more of."
     "I want to work on problems at a larger scale than my
      current company operates at."
     "I've been in a generalist role and I want to specialize."

   ⭐ If the real reason is negative — a bad manager, a broken
     culture — translate it into a positive: "I'm looking for
     a team with a stronger engineering culture around
     testing and code review" rather than "my last team didn't
     test anything."
```

---

## 6. Amazon Leadership Principles

```
   ⭐ AMAZON INTERVIEWS EXPLICITLY MAP QUESTIONS TO PRINCIPLES,
     and many other companies have adopted similar frameworks.
     Worth knowing even elsewhere.

   ┌──────────────────────────────────────────────────────────────┐
   │ Customer Obsession       start from the customer, work back  │
   │ ⭐ Ownership              act beyond your own scope; ⭐ "never │
   │                          say 'that's not my job'"            │
   │ Invent and Simplify      innovation AND simplification       │
   │ Are Right, A Lot         strong judgment; seek disconfirming │
   │                          views                               │
   │ Learn and Be Curious                                         │
   │ Hire and Develop the Best                                    │
   │ ⭐ Insist on the Highest Standards                            │
   │ Think Big                                                    │
   │ Bias for Action          ⭐ speed matters; many decisions are │
   │                          reversible                          │
   │ Frugality                constraints breed resourcefulness   │
   │ Earn Trust               ⭐ self-critical, vocally so         │
   │ Dive Deep                ⭐ no task is beneath you; audit     │
   │                          frequently                          │
   │ ⭐ Have Backbone;         disagree respectfully, then         │
   │   Disagree and Commit    COMMIT FULLY                        │
   │ Deliver Results                                              │
   │ Strive to be Earth's Best Employer                           │
   │ Success and Scale Bring Broad Responsibility                 │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ HOW TO PREPARE FOR PRINCIPLE-BASED INTERVIEWS

   • ⭐ Map your existing stories to principles, don't invent
     stories per principle
   • ⭐ Expect DEEP follow-ups: "what exactly did YOU do?",
     "what was the data?", "what would you do differently?"
     Amazon interviewers are trained to drill until they hit
     the specifics.
   • ⭐ Have numbers. Amazon in particular expects quantified
     impact.
   • "Disagree and Commit" is asked constantly — have a strong
     story where you disagreed, lost, and committed genuinely.
```

---

## 7. Hard Questions

```
   ⭐ "WHAT'S YOUR BIGGEST WEAKNESS?"

   ⚠️ WRONG: a disguised strength ("I'm a perfectionist")
   ⚠️ WRONG: something disqualifying ("I miss deadlines")

   ⭐ RIGHT: a genuine, non-disqualifying weakness, plus what
     you're actively doing about it.

   "I tend to go deeper on technical detail than the audience
    needs — I've had feedback that my design docs were too
    long for executives. I've started writing a one-paragraph
    summary at the top and moving the detail to an appendix,
    and the last two docs got much faster decisions."

   ⭐ THE STRUCTURE: real weakness → concrete evidence you're
     addressing it → evidence it's improving.
```

```
   ⭐ "TELL ME ABOUT A CONFLICT WITH YOUR MANAGER"

   ⚠️ Saying "I've never disagreed with a manager" reads as
     either dishonest or passive.

   ⭐ Pick a genuine professional disagreement — a priority, a
     technical direction, a process. Show that you raised it
     directly and privately, made your case, and then
     committed to the outcome.
```

```
   ⭐ "WHY IS THERE A GAP IN YOUR RESUME?"

   Answer briefly, factually, without apology, and move on.
   Caring for family, health, a failed startup, redundancy,
   deliberate time off — all normal. ⭐ The length of your
   discomfort is more noticeable than the gap itself.

   If you did anything relevant during it — learning, a
   project, contracting — mention it in one sentence.
```

```
   ⭐ "WHAT SALARY ARE YOU LOOKING FOR?" (early in the process)

   ⭐ Deflect to the end if possible — you have more leverage
     after they want you.

   "I'd rather focus on whether this is a good fit first.
    I'm confident we can align on compensation if it is —
    what range has been budgeted for the role?"

   ⭐ Many jurisdictions now require the employer to disclose
     the range on request. Ask.
   → Full treatment in [Negotiation](negotiation.md).
```

```
   ⭐ "DO YOU HAVE ANY QUESTIONS FOR ME?"

   ⚠️ "No, I think you covered everything" is a genuinely bad
     answer. It reads as disinterest.
   ⭐ Always have three questions. See §10.
```

---

## 8. Level Calibration

```
   ⭐ THE SAME STORY IS SCORED DIFFERENTLY BY LEVEL

   ┌──────────────────────────────────────────────────────────────┐
   │ JUNIOR       Learns fast · asks good questions · delivers    │
   │              well-scoped tasks · receptive to feedback       │
   ├──────────────────────────────────────────────────────────────┤
   │ MID          Owns a feature or component end to end ·        │
   │              unblocks themselves · reliable delivery ·       │
   │              reviews others' code meaningfully               │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ SENIOR     Owns AMBIGUOUS problems · ⭐ influences beyond   │
   │              their own work · mentors · makes tradeoffs      │
   │              explicit · thinks about operations and          │
   │              maintenance, not just building                  │
   ├──────────────────────────────────────────────────────────────┤
   │ STAFF        ⭐ Changes how the ORGANIZATION works · sets     │
   │              technical direction · influences across teams · │
   │              identifies problems nobody has framed yet ·     │
   │              multiplies others' effectiveness                │
   ├──────────────────────────────────────────────────────────────┤
   │ PRINCIPAL    Multi-year technical strategy · industry-level  │
   │              expertise · organizational impact               │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐⭐ THE SENIOR-LEVEL DIFFERENTIATOR

   Junior and mid stories describe WHAT WAS BUILT.
   Senior and staff stories describe WHY IT WAS BUILT THAT WAY,
   what was traded away, what it cost to operate, and how the
   engineer changed what OTHER PEOPLE did.

   ⭐ Concretely, senior answers include:
     • The alternative that was rejected, and why
     • The cost accepted (complexity, latency, money, risk)
     • How they got other people to agree
     • What happened AFTER the launch — did it hold up?
     • ⭐ What systemic change followed, not just the local fix
```

---

## 9. Common Failure Modes

```
   ❌ RAMBLING WITHOUT STRUCTURE
      Five minutes with no clear arc. The interviewer loses
      the thread and can't score it.
      ⭐ FIX: STAR, and keep answers to 2-3 minutes. Pause and
        ask "would you like more detail on any part?"

   ❌ ⭐ "WE" WITHOUT "I"
      The single most common problem. The interviewer cannot
      tell what YOU did.
      ⭐ FIX: describe the team's context, then say "my part
        was..." explicitly.

   ❌ NO NUMBERS
      "It got much faster" is unverifiable.
      ⭐ FIX: before/after, scale, duration, team size. Estimate
        if you must, and say you're estimating.

   ❌ ⭐ BLAMING OTHERS
      Even when accurate, it reads as a preview.
      ⭐ FIX: focus on what YOU controlled and what you'd do
        differently.

   ❌ TOO MUCH SETUP
      Three minutes of background, twenty seconds of action.
      ⭐ FIX: 2-3 sentences of situation, maximum.

   ❌ NO LEARNING
      A story with no reflection reads as no growth.
      ⭐ FIX: end with what changed in your approach.

   ❌ ⭐ REHEARSED-SOUNDING DELIVERY
      Memorized word-for-word answers sound hollow and
      collapse when the follow-up goes sideways.
      ⭐ FIX: know the BEATS, not the script. Practice out loud
        with variation.

   ❌ NOT ANSWERING THE ACTUAL QUESTION
      Telling your favourite story regardless of what was asked.
      ⭐ FIX: listen, then pick the story that fits. It's fine to
        say "let me think for a moment."
```

---

## 10. Questions to Ask Them

```
   ⭐ GOOD QUESTIONS DEMONSTRATE JUDGMENT AND GENUINELY INFORM
     YOUR DECISION. Ask things you actually want to know.

   ── ABOUT THE ROLE ─────────────────────────────────────────────
   • What would success look like in the first six months?
   • ⭐ What's the hardest problem the team is facing right now?
   • What does a typical week look like?
   • Why is this role open?

   ── ABOUT THE TEAM AND ENGINEERING CULTURE ────────────────────
   • ⭐ How do technical decisions get made when people disagree?
   • What does the code review culture look like?
   • ⭐ How much of the team's time goes to new work versus
     maintenance and on-call?
   • How does on-call work? What's the page volume like?
   • ⭐ What's the deploy frequency, and how long does a change
     take to reach production?

   ── ABOUT GROWTH ───────────────────────────────────────────────
   • What separates someone performing well here from someone
     performing exceptionally?
   • ⭐ How has the last person in this role progressed?
   • How does promotion work in practice?

   ── ⭐ THE HIGH-SIGNAL ONES ─────────────────────────────────────
   • ⭐ "What's something about working here that you'd change
     if you could?"  (⭐ tests candour — a rehearsed "nothing"
     is itself informative)
   • ⭐ "What's the biggest risk to this team succeeding?"
   • ⭐ "How does the team handle it when something goes badly
     wrong?"  (⭐ reveals whether postmortems are blameless in
     practice or only in theory)
   • "What's changed most about engineering here in the last
     year?"
```

```
   ⚠️ DON'T ASK, AT LEAST NOT EARLY
     • Anything answered on the careers page
     • Compensation and benefits in a first technical round
     • "How much overtime is expected?" (⭐ ask instead about
       on-call load and deploy cadence — you'll learn the same
       thing and it reads as engineering interest)
```

---

## 11. Preparation Protocol

```
   ⭐ FOUR WEEKS OUT
   □ List every significant project from the last 2-3 years
   □ ⭐ Map them to the 12 core story types
   □ Identify gaps — themes you have no story for
   □ Gather the NUMBERS: metrics, scale, timelines, team size

   ⭐ THREE WEEKS OUT
   □ Write each story in STAR form (⭐ written, not just thought)
   □ ⭐ For each, write the hardest follow-up you'd be asked and
     answer it
   □ Cut every story to 2-3 minutes

   ⭐ TWO WEEKS OUT
   □ ⭐ Say them OUT LOUD. Record yourself.
   □ Listen back — you will hear rambling, missing numbers,
     and "we" where it should be "I"
   □ Practice with a partner who interrupts and probes

   ⭐ ONE WEEK OUT
   □ Research the company: products, engineering blog, recent
     news, their values framework if they have one
   □ ⭐ Prepare 5+ questions to ask
   □ Practice the specific opener: "tell me about yourself"

   ⭐ THE DAY BEFORE
   □ Re-read your story bank once — do not re-memorize
   □ ⭐ Sleep. Fatigue costs more than an extra hour of prep.
```

```
   ⭐ "TELL ME ABOUT YOURSELF" — prepare this specifically

   It opens most interviews and most people improvise badly.

   ⭐ THE STRUCTURE (60-90 seconds)
     1. Where you are now, in one sentence
     2. ⭐ Two or three highlights that are RELEVANT to this role
     3. Why you're interested in THIS role, specifically

   ⚠️ Not a chronological life story. Not everything on your
     resume. ⭐ A curated trailer aimed at this job.
```

---

## 12. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                  BEHAVIORAL INTERVIEWS — ONE PAGE                    ║
╠══════════════════════════════════════════════════════════════════════╣
║ MEASURED: impact · judgment · collaboration · ⭐ growth               ║
║ ⭐ They want SPECIFIC PAST EVENTS, not philosophy.                    ║
║   Hypothetical asked? → answer with what you ACTUALLY did.           ║
╠══════════════════════════════════════════════════════════════════════╣
║ STAR — ⭐ ACTION IS 60% OF THE ANSWER                                 ║
║   Situation 15% · Task 10% · ⭐ ACTION 60% · Result 15%               ║
║   ⭐ + a fifth beat: "and what I learned / did differently after"     ║
║ ⚠️ The classic failure: 3 min of context, 20 sec of what you did.     ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ 8-12 STORIES, each covering multiple themes. Depth > coverage.     ║
║ WRITE THEM DOWN — stories only thought about collapse under          ║
║   follow-up. Prepare the HARDEST follow-up for each.                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ THE FAILURE STORY IS THE MOST DIAGNOSTIC QUESTION                 ║
║   real consequences + YOUR ownership + specific lesson +             ║
║   ⭐ EVIDENCE you applied it later                                    ║
║   ⚠️ never: disguised strength · someone else's fault · no lesson     ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ CONFLICT: they test whether you DISAGREE PRODUCTIVELY then COMMIT  ║
║   represent the other person FAIRLY · make the case with DATA ·      ║
║   ⭐ "I disagreed, lost, committed fully, they were right about X"    ║
║     is STRONGER than winning                                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ SENIOR DIFFERENTIATOR: junior stories say WHAT WAS BUILT;         ║
║   senior stories say WHY, what was TRADED AWAY, what it COST to      ║
║   operate, and how you changed what OTHER PEOPLE did.                ║
╠══════════════════════════════════════════════════════════════════════╣
║ FAILURE MODES: rambling · ⭐ "we" without "I" · no numbers ·          ║
║   blaming · too much setup · no learning · memorized delivery ·      ║
║   answering a different question than the one asked                  ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ "Why are you leaving?" → what you're moving TOWARD.                ║
║   NEVER criticize your current employer — it reads as a preview.     ║
║ ⭐ ALWAYS have questions. Best ones: "how do technical decisions get  ║
║   made when people disagree?" · "what's the biggest risk to this     ║
║   team?" · "what would you change if you could?"                     ║
║ ⭐ Prepare "tell me about yourself" specifically — 90 sec, curated    ║
║   for THIS role, not a chronological life story.                     ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Next:** [Coding Interview Craft →](coding-craft.md) · **Related:** [Negotiation](negotiation.md) · [System Design Framework](../05-system-design/02-framework.md)
