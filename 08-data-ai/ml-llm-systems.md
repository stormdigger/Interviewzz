# 🤖 ML & LLM Systems

> This is a systems book, not a modelling book. The hard parts of production ML are almost never the model architecture — they're data, serving, drift, and evaluation.

**Prerequisite:** [Data Engineering](data-engineering.md) · [System Design Fundamentals](../05-system-design/00-fundamentals.md)

---

## 📑 Contents

1. [The Mental Model](#1-the-mental-model)
2. [The ML Lifecycle](#2-the-ml-lifecycle)
3. [Features and the Feature Store](#3-features-and-the-feature-store)
4. [Training](#4-training)
5. [Evaluation](#5-evaluation)
6. [Serving](#6-serving)
7. [Monitoring and Drift](#7-monitoring-and-drift)
8. [Recommendation Systems](#8-recommendation-systems)
9. [LLM Fundamentals](#9-llm-fundamentals)
10. [Prompting and Context](#10-prompting-and-context)
11. [RAG](#11-rag)
12. [Agents and Tool Use](#12-agents-and-tool-use)
13. [Fine-Tuning vs RAG vs Prompting](#13-fine-tuning-vs-rag-vs-prompting)
14. [LLM Evaluation](#14-llm-evaluation)
15. [LLM Production Concerns](#15-llm-production-concerns)
16. [Interview Section](#16-interview-section)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. The Mental Model

```
   ⭐ ML SYSTEMS FAIL DIFFERENTLY FROM NORMAL SOFTWARE

   ┌──────────────────────────────────────────────────────────────┐
   │ NORMAL SOFTWARE          ML SYSTEMS                          │
   │ ────────────────         ──────────                          │
   │ Fails loudly             ⭐ Fails SILENTLY — it keeps         │
   │ (exception, 500)           returning confident predictions   │
   │                            that are wrong                    │
   │ Behaviour is in code     Behaviour is in DATA                │
   │ Tests are deterministic  Metrics are statistical             │
   │ Deploy = new logic       ⭐ Deploy = new logic AND new data   │
   │                            distribution assumptions          │
   │ Degrades when changed    ⭐ Degrades when NOTHING changes     │
   │                            (the world moves; the model       │
   │                            doesn't)                          │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE PRACTICAL CONSEQUENCE: monitoring and evaluation are
     not optional extras. They are the primary engineering work,
     because without them you cannot tell a working system from
     a broken one.
```

```
   ⭐ THE HIERARCHY OF WHAT ACTUALLY MOVES THE NEEDLE

   1. ⭐ Is ML even the right tool? (rules are often better,
      and always more debuggable)
   2. ⭐ DATA quality and quantity
   3. Feature engineering
   4. The right problem framing / objective function
   5. ⚠️ Model architecture — usually the LEAST important,
      and where most attention goes
```

---

## 2. The ML Lifecycle

```
   ┌──────────────────────────────────────────────────────────────┐
   │  ① PROBLEM FRAMING                                           │
   │     ⭐ What business decision does this inform? What's the     │
   │       baseline? What's the cost of a false positive vs a     │
   │       false negative?                                        │
   │  ② DATA COLLECTION & LABELLING                               │
   │  ③ EXPLORATION & FEATURE ENGINEERING                         │
   │  ④ TRAINING & TUNING                                         │
   │  ⑤ ⭐ OFFLINE EVALUATION                                      │
   │  ⑥ DEPLOYMENT (shadow → canary → full)                       │
   │  ⑦ ⭐ ONLINE EVALUATION (A/B test)                            │
   │  ⑧ MONITORING & RETRAINING                                   │
   │        └────────────── back to ② ──────────────┘             │
   └──────────────────────────────────────────────────────────────┘

   ⭐ THE LOOP IS THE POINT. A model is not a deliverable — it's
     a continuously maintained artifact whose accuracy decays
     unless the loop runs.
```

```
   ⭐ ALWAYS ESTABLISH A BASELINE FIRST

   Before any model: what does the simplest possible approach
   achieve?
     • Predict the majority class
     • Predict the previous value
     • A handful of hand-written rules
     • The existing heuristic the business already uses

   ⚠️ A model that beats random but not the existing rule is a
     regression, and it's remarkably common for that to go
     unnoticed because nobody measured the rule.
```

---

## 3. Features and the Feature Store

```
   ⭐⭐ TRAIN/SERVE SKEW — THE #1 PRODUCTION ML BUG

   The model performs well offline and badly in production,
   because the features it receives at serving time differ from
   those it trained on.

   CAUSES
     • ⭐ Training features computed in SQL/Spark; serving
       features computed in application code. The two
       implementations drift.
     • Different default or null handling
     • ⭐ TIME TRAVEL: training used a value that wasn't
       available at prediction time
     • Different aggregation windows

   ⭐ THE FIX: ONE definition, used by BOTH paths. That is
     precisely what a feature store exists to provide.
```

```
   ⭐ FEATURE STORE ARCHITECTURE

   ┌──────────────────────────────────────────────────────────────┐
   │  ONE feature definition                                      │
   │       │                                                      │
   │       ├──▶ OFFLINE STORE (warehouse / Parquet)               │
   │       │      ⭐ POINT-IN-TIME correct historical values       │
   │       │      → used to build training sets                   │
   │       │                                                      │
   │       └──▶ ONLINE STORE (Redis / DynamoDB)                   │
   │              latest values, ⭐ single-digit ms lookup          │
   │              → used at inference                             │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐⭐ POINT-IN-TIME CORRECTNESS — the subtlest and most
     damaging bug in ML

   ⚠️ THE LEAK
     Training a churn model, you join "customer's total
     lifetime orders" as of TODAY onto a label from six months
     ago. The feature contains information from AFTER the
     prediction moment. The model looks excellent offline and
     is useless in production.

   ⭐ THE RULE: every feature value must be exactly what would
     have been KNOWN at prediction time — not a moment later.

   ⭐ This requires as-of joins: for each label at time T, join
     the feature value as it stood at time T. Feature stores
     implement this; hand-rolled pipelines usually get it wrong
     at least once.
```

```
   ⭐ OTHER LEAKAGE SOURCES WORTH KNOWING

   • ⚠️ Target leakage — a feature that is a proxy for the label
     ("account_closed_date" when predicting churn)
   • ⭐ Preprocessing before splitting — computing normalization
     statistics on the full dataset leaks test information into
     training
   • Duplicate rows spanning the train/test split
   • ⭐ Random splits on TEMPORAL data — you train on the future
     and test on the past. Always split chronologically for
     time-dependent problems.
```

---

## 4. Training

```
   ⭐ SPLITTING — get this right or nothing downstream is valid

   RANDOM SPLIT       ✅ i.i.d. data only
   ⭐ TEMPORAL SPLIT   train on the past, test on the future.
                      REQUIRED for anything time-dependent —
                      which is most real problems.
   GROUP SPLIT        ⭐ keep all rows for one entity on ONE side,
                      or the model memorizes the entity
   STRATIFIED         preserve class balance for imbalanced data

   ⚠️ A random split on temporal data produces beautiful offline
     metrics and a model that fails immediately in production.
```

```
   ⭐ IMBALANCED DATA — a fraud model predicting "not fraud"
     always is 99.9% accurate and completely worthless.

   TECHNIQUES
     • ⭐ Use the right METRIC (precision/recall/PR-AUC, never
       accuracy)
     • Class weights in the loss function
     • ⚠️ Resampling — SMOTE and similar. Use with care; they
       can create unrealistic synthetic points, and they change
       the base rate so calibration suffers.
     • ⭐ Adjust the DECISION THRESHOLD rather than the data.
       Usually the cleanest lever, and it makes the
       precision/recall tradeoff explicit.
```

```
   ⭐ REPRODUCIBILITY REQUIREMENTS

   □ Version the DATA (not just the code) — DVC, or immutable
     snapshots
   □ Version the code, and pin dependencies
   □ Log hyperparameters, random seeds, and the environment
   □ ⭐ Track experiments (MLflow, W&B) — otherwise "which run
     produced the model in production?" becomes unanswerable
   □ ⭐ Model registry with lineage: this model came from this
     data, this code, this config

   ⚠️ Without this, you cannot reproduce a result, debug a
     regression, or satisfy an audit.
```

---

## 5. Evaluation

```
   ⭐ CLASSIFICATION METRICS — and when each misleads

   ┌──────────────────────────────────────────────────────────────┐
   │ ACCURACY     (TP+TN)/all                                     │
   │   ⚠️ USELESS on imbalanced data                               │
   │ PRECISION    TP/(TP+FP)  "of my positive predictions,        │
   │              how many were right?"                           │
   │   ⭐ Optimize when FALSE POSITIVES are costly                 │
   │      (spam filter — don't block real mail)                   │
   │ RECALL       TP/(TP+FN)  "of the actual positives, how many  │
   │              did I catch?"                                   │
   │   ⭐ Optimize when FALSE NEGATIVES are costly                 │
   │      (cancer screening — don't miss a case)                  │
   │ F1           harmonic mean of precision and recall           │
   │ ⭐ PR-AUC     better than ROC-AUC on imbalanced data          │
   │ ROC-AUC      ⚠️ can look great on heavily imbalanced data     │
   │              even when the model is useless                  │
   │ ⭐ CALIBRATION  when the model says 0.7, does it happen 70%   │
   │              of the time? ⭐ Essential whenever the           │
   │              PROBABILITY is used in a downstream decision,   │
   │              and routinely ignored.                          │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE THRESHOLD IS A BUSINESS DECISION, NOT A MODEL PROPERTY

   A classifier outputs a probability. Where you cut it
   determines the precision/recall balance, and the right
   cut depends on the relative cost of the two error types.

   ⭐ Frame it that way explicitly: "at this threshold we catch
     80% of fraud and falsely flag 2% of legitimate
     transactions — is that the trade you want?" That's a
     conversation with the business, not a modelling choice.
```

```
   ⭐ OFFLINE METRICS ARE A PROXY. ONLINE METRICS ARE THE TRUTH.

   ⚠️ A model with better AUC can perform WORSE in production:
     • The offline metric doesn't match the business objective
     • The model changes user behaviour, which changes the data
       (⭐ a feedback loop that offline evaluation cannot capture)
     • Latency increases and the latency cost exceeds the
       accuracy gain
     • The offline test set doesn't reflect live traffic

   ⭐ THEREFORE: A/B test everything. Offline evaluation decides
     what's worth testing; the online test decides what ships.
```

---

## 6. Serving

```
   ┌──────────────────────────────────────────────────────────────┐
   │ BATCH             Precompute predictions on a schedule,      │
   │                   serve from a lookup table.                 │
   │   ✅ ⭐ Simplest by far, cheapest, no latency risk             │
   │   ⚠️ Stale; only works with a bounded, known input space      │
   │   Use for: daily recommendations, risk scores, segments      │
   ├──────────────────────────────────────────────────────────────┤
   │ ONLINE / REAL-TIME  Predict per request                      │
   │   ✅ Fresh, uses request-time context                         │
   │   ⚠️ Latency budget, scaling, higher operational burden       │
   ├──────────────────────────────────────────────────────────────┤
   │ STREAMING         Predict on events as they arrive           │
   │ EDGE / ON-DEVICE  ⭐ Privacy, offline capability, zero        │
   │                   network latency. ⚠️ Model size limits,      │
   │                   fragmented hardware, hard to update.       │
   └──────────────────────────────────────────────────────────────┘

   ⭐ START WITH BATCH. A surprising number of "real-time ML"
     requirements are satisfied by precomputed predictions
     refreshed hourly, at a fraction of the complexity.
```

```
   ⭐ LATENCY OPTIMIZATION, roughly by impact

   1. ⭐ Do you need the model at all for this request? Cache
      predictions for repeated inputs.
   2. ⭐ Smaller model: distillation, pruning, quantization.
      Often 4× faster for 1-2% accuracy loss — usually a good
      trade.
   3. Batch requests together (dynamic batching) — GPUs are
      far more efficient on batches.
   4. Hardware: GPU/TPU for deep models, though ⚠️ for small
      models CPU is often cheaper and simpler.
   5. Optimized runtime: ONNX Runtime, TensorRT, vLLM.
   6. ⭐ Feature lookup is frequently the bottleneck, not
      inference. Measure before optimizing the model.
```

```
   ⭐ DEPLOYMENT PATTERN FOR MODELS

   SHADOW      Run the new model alongside the old, log its
               predictions, serve the OLD one.
               ⭐ Zero user risk, real production traffic.
               The best first step, and underused.
   CANARY      Route a small percentage of traffic, watch
               metrics, expand.
   A/B TEST    ⭐ Measure BUSINESS impact, not just model metrics.
   MULTI-ARMED ⭐ Automatically shift traffic toward the better
     BANDIT    performer — better than a fixed A/B split when
               you want to minimize regret rather than maximize
               statistical rigour.
```

---

## 7. Monitoring and Drift

```
   ⭐⭐ MODELS DEGRADE WITHOUT ANY CODE CHANGING.
     The world moves; the model doesn't.

   ┌──────────────────────────────────────────────────────────────┐
   │ DATA DRIFT       P(X) changes — the input distribution       │
   │                  shifts. New user demographics, a new        │
   │                  product line, seasonality.                  │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ CONCEPT DRIFT  P(Y|X) changes — the RELATIONSHIP itself    │
   │                  changes. Fraud patterns adapt to your       │
   │                  detection. This is the dangerous one,       │
   │                  because inputs look normal.                 │
   ├──────────────────────────────────────────────────────────────┤
   │ ⚠️ FEEDBACK LOOPS The model's predictions change behaviour,   │
   │                  which changes future training data.         │
   │                  ⭐ A recommender only ever gets feedback on  │
   │                  what it recommended, so it becomes          │
   │                  increasingly confident about a narrowing    │
   │                  slice of reality.                           │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHAT TO MONITOR — in order of how quickly it alerts you

   ┌──────────────────────────────────────────────────────────────┐
   │ OPERATIONAL   latency · throughput · errors · resource use   │
   │               (fast, always available)                       │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ INPUT       feature distributions vs training · missing     │
   │               rate · out-of-range values · ⭐ schema changes  │
   │               (fast — you don't need labels for this)        │
   ├──────────────────────────────────────────────────────────────┤
   │ OUTPUT        prediction distribution · confidence           │
   │               distribution · class balance shifts            │
   │               (fast, and often the first real signal)        │
   ├──────────────────────────────────────────────────────────────┤
   │ ⚠️ PERFORMANCE accuracy, precision, recall                    │
   │               ⭐ REQUIRES LABELS, which often arrive days or  │
   │               months later — sometimes never. This is why    │
   │               input and output monitoring matter so much.    │
   ├──────────────────────────────────────────────────────────────┤
   │ ⭐ BUSINESS    conversion · revenue · user satisfaction       │
   │               ⭐ THE ONLY METRIC THAT ULTIMATELY MATTERS      │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ THE LABEL DELAY PROBLEM

   For a churn model, you learn whether a prediction was right
   30 days later. For a loan default model, months or years.

   ⚠️ So you CANNOT rely on accuracy monitoring to detect
     problems quickly. Input drift and output distribution
     shifts are your early warning, and they're available
     immediately.

   ⭐ Also invest in a labelled holdout that's collected
     continuously, even if small — it gives you a delayed but
     honest accuracy signal.
```

---

## 8. Recommendation Systems

```
   ⭐ THE TWO-STAGE FUNNEL — universal across recommendation,
     search, and ads

   ┌──────────────────────────────────────────────────────────────┐
   │ ① CANDIDATE GENERATION   millions → hundreds                 │
   │    Cheap, high-RECALL retrieval from several sources:        │
   │      • collaborative filtering                               │
   │      • ⭐ embedding similarity via approximate nearest        │
   │        neighbour search (HNSW/IVF)                           │
   │      • content-based matching                                │
   │      • trending / popular                                    │
   │      • ⭐ an EXPLORATION bucket                               │
   │    ⭐ Diversity of SOURCES creates diversity of RESULTS.      │
   ├──────────────────────────────────────────────────────────────┤
   │ ② RANKING                hundreds → ~20 ordered              │
   │    Expensive model, high PRECISION. Predicts multiple        │
   │    objectives and blends them.                               │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ RE-RANKING             business rules                      │
   │    diversity · freshness · ⭐ integrity filters · ads         │
   └──────────────────────────────────────────────────────────────┘

   ⭐ WHY TWO STAGES: running the expensive model over millions
     of candidates at high QPS is computationally impossible.
     Cheap recall then expensive precision is the only tractable
     shape.
```

```
   ⭐ COLD START — the defining practical problem

   NEW USER   popular content in their region → ⭐ update the
              interest model in near-real-time as they interact.
              Fast skips are as informative as engagement.
   NEW ITEM   ⭐ CONTENT features (text, image, audio embeddings)
              place it in the space with ZERO interactions.
              Then ⭐ STAGED EXPOSURE: show to a small cohort,
              promote based on engagement RATE within that
              cohort — not absolute counts, so new creators
              compete fairly.
   NEW SYSTEM popularity ranking, content-based, or bootstrap
              from a related domain
```

```
   ⚠️ THE FEEDBACK LOOP / FILTER BUBBLE IS STRUCTURAL

   Optimizing engagement narrows the signal, which narrows
   recommendations, which narrows the signal further. The
   system converges to a degenerate local optimum: high
   short-term engagement, declining long-term retention.

   ⭐ COUNTERMEASURES MUST BE DELIBERATE — the optimizer will
     never find them on its own:
     • A guaranteed exploration budget per session
     • Diversity constraints in re-ranking
     • ⭐ Penalize repetition even when predicted engagement is high
     • Include retention and satisfaction proxies in the
       objective, not just immediate engagement
```

---

## 9. LLM Fundamentals

```
   ⭐ WHAT AN LLM ACTUALLY DOES

   It predicts the next TOKEN given the preceding tokens.
   Everything else — reasoning, code, translation — is an
   emergent consequence of doing that extremely well at scale.

   ⭐ IMPLICATIONS THAT MATTER IN PRACTICE
     • It has no persistent memory between calls. ⭐ The context
       window IS the memory.
     • It doesn't "know" anything — it produces statistically
       likely continuations. ⚠️ Confident-sounding fabrication
       is the expected failure mode, not an aberration.
     • ⭐ It cannot verify its own claims.
     • Output is stochastic unless temperature is 0 (and even
       then, not perfectly reproducible across hardware).
```

```
   ⭐ TOKENS AND CONTEXT

   A token is roughly ¾ of an English word. Code and non-Latin
   scripts tokenize less efficiently.

   ⭐ THE CONTEXT WINDOW holds the system prompt, conversation
     history, retrieved documents, tool definitions, and the
     response. Everything competes for the same budget.

   ⚠️ COST AND LATENCY SCALE WITH TOKENS, and attention is
     quadratic in sequence length — so long contexts are
     expensive in a way that compounds.

   ⚠️ "LOST IN THE MIDDLE": models attend most reliably to the
     BEGINNING and END of a long context. ⭐ Put the most
     important material at the edges, not buried in the middle.
```

```
   ⭐ SAMPLING PARAMETERS

   TEMPERATURE   0 = deterministic/focused · higher = more varied
                 ⭐ Use 0 for extraction, classification, and
                   anything where you'll parse the output
   TOP-P         nucleus sampling — consider tokens covering
                 the top p probability mass
   ⭐ STOP SEQUENCES  end generation at a marker — essential for
                 structured output
   MAX TOKENS    a cost and runaway safeguard
```

---

## 10. Prompting and Context

```
   ⭐ WHAT ACTUALLY IMPROVES OUTPUT QUALITY, in order

   1. ⭐ BE SPECIFIC ABOUT THE TASK, the format, and the
      constraints. Most bad output is an underspecified prompt.
   2. ⭐ GIVE EXAMPLES (few-shot). Usually the single largest
      quality improvement available.
   3. ⭐ PROVIDE THE RELEVANT CONTEXT rather than relying on
      the model's parametric knowledge.
   4. Ask for reasoning before the answer on multi-step tasks.
   5. Specify the output format precisely, and validate it.
   6. Decompose complex tasks into chained calls.
```

```
   ⭐ STRUCTURED OUTPUT — the practical necessity

   ⚠️ Asking for JSON in the prompt and parsing the result is
     unreliable. Use the API's structured output / function
     calling with a schema, which constrains generation so the
     output is valid by construction.

   ⭐ AND STILL VALIDATE. Schema-valid does not mean
     semantically correct — the model can return a
     well-formed object with wrong values.
```

```
   ⭐ CONTEXT ENGINEERING — the real discipline

   The limiting factor in most LLM applications is not the
   model, it's what you put in front of it.

   ⭐ PRINCIPLES
     • RELEVANCE over volume — more context is not better
       context. Irrelevant material actively degrades output.
     • Put critical information at the START or END
     • Structure it (headings, delimiters) so boundaries are
       unambiguous
     • ⭐ For long conversations, SUMMARIZE older turns rather
       than truncating them
     • Budget explicitly: system prompt, history, retrieved
       documents, and response all compete
```

---

## 11. RAG

```
   ⭐ RETRIEVAL-AUGMENTED GENERATION — the standard architecture
     for grounding an LLM in your own data

   ┌──────────────────────────────────────────────────────────────┐
   │  INDEXING (offline)                                          │
   │    documents → ⭐ CHUNK → embed → vector store                │
   │                                                              │
   │  QUERY (online)                                              │
   │    question → embed → ⭐ RETRIEVE top-k → ⭐ RERANK →          │
   │    assemble context → generate → ⭐ CITE sources             │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐⭐ CHUNKING IS THE MOST UNDER-APPRECIATED DECISION IN RAG.

   ⚠️ Too small  → the chunk lacks the context needed to be
                  useful, and retrieval returns fragments
   ⚠️ Too large  → embeddings become diluted averages and
                  retrieval precision collapses

   ⭐ STRATEGIES, roughly best to worst
     1. ⭐ STRUCTURE-AWARE — split on document structure
        (sections, headings, function definitions). Respects
        natural semantic boundaries.
     2. ⭐ Semantic chunking — split where topic shifts
     3. Recursive character splitting with OVERLAP (a common
        default: ~500-1000 tokens with 10-20% overlap)
     4. ⚠️ Fixed-size splitting — simple and usually the worst

   ⭐ ALWAYS attach metadata: source, section, date, permissions.
     You need it for filtering, for citations, and for access
     control.
```

```
   ⭐ RETRIEVAL QUALITY — where most RAG systems actually fail

   ⚠️ Pure vector search misses exact matches — product codes,
     error numbers, names, acronyms. Embeddings capture
     semantics, not literals.

   ⭐ HYBRID SEARCH is the reliable default:
     dense (embedding) + sparse (BM25 keyword), fused with
     reciprocal rank fusion.

   ⭐ THEN RERANK. Retrieve ~50 candidates cheaply, then use a
     cross-encoder to rerank to the top 5. A cross-encoder sees
     the query and document TOGETHER rather than comparing
     independent embeddings, which is substantially more
     accurate. ⭐ This is usually the single biggest RAG quality
     improvement available.
```

```
   ⭐ ADVANCED RAG TECHNIQUES WORTH KNOWING

   QUERY REWRITING     expand or clarify the query before
                       retrieval (⭐ especially for follow-up
                       questions in a conversation, which are
                       often unintelligible standalone)
   HyDE                generate a hypothetical answer, embed
                       THAT, and search with it
   ⭐ PARENT-DOCUMENT   retrieve small precise chunks, then feed
                       the LARGER surrounding section to the
                       model — precision in retrieval, context
                       in generation
   ⭐ METADATA FILTERS  filter by date, source, or PERMISSIONS
                       before semantic search
   MULTI-HOP           retrieve, reason, retrieve again
   ⭐ CONTEXTUAL CHUNKS prepend a short document-level summary to
                       each chunk before embedding, so isolated
                       chunks carry their context
```

```
   ⚠️⚠️ THE RAG SECURITY ISSUE PEOPLE MISS

   ⭐ RETRIEVAL MUST RESPECT THE USER'S PERMISSIONS.

   If the vector store contains documents from across the
   organization and you retrieve purely by semantic similarity,
   you will leak confidential material to users who shouldn't
   see it — and it will be laundered through a fluent, confident
   answer that gives no indication of its source's sensitivity.

   ⭐ Filter by permission BEFORE or DURING retrieval, never
     after generation.
```

---

## 12. Agents and Tool Use

```
   ⭐ AN AGENT IS A LOOP: the model decides to call a tool, sees
     the result, and decides again — until it's done.

   ┌──────────────────────────────────────────────────────────────┐
   │  user goal                                                   │
   │      ▼                                                       │
   │  ┌───────────────┐                                           │
   │  │ model decides │◀─────────────────┐                        │
   │  └───────┬───────┘                  │                        │
   │          │ calls a tool             │ tool result            │
   │          ▼                          │                        │
   │  ┌───────────────┐                  │                        │
   │  │ execute tool  │──────────────────┘                        │
   │  └───────────────┘                                           │
   │          │ (when the model stops calling tools)              │
   │          ▼                                                   │
   │      final answer                                            │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ WHAT MAKES AGENTS WORK OR FAIL

   ✅ WORKS WHEN
     • Tools are few, well-described, and clearly distinct
     • ⭐ Tool descriptions are written for the MODEL, not
       copied from developer docs
     • Errors are returned as informative text the model can
       act on
     • The task decomposes into verifiable steps
     • ⭐ There's a step limit and a cost ceiling

   ⚠️ FAILS WHEN
     • Too many overlapping tools → the model picks wrong
     • ⭐ Errors are opaque, so the model retries identically
       forever
     • Long horizons compound small errors
     • ⚠️ No termination condition → infinite loops and
       runaway cost
```

```
   ⚠️⚠️ THE SECURITY MODEL IS THE HARD PART

   ⭐ PROMPT INJECTION: any untrusted content the model reads —
     a web page, an email, a retrieved document, a tool result —
     can contain instructions. The model cannot reliably
     distinguish data from instructions.

   ⭐ THEREFORE: DO NOT rely on the model to enforce security.

   ✅ CONTROLS THAT ACTUALLY WORK
     • ⭐ Least privilege on tools — an agent that can only read
       cannot be tricked into writing
     • ⭐ Human approval for consequential or irreversible
       actions
     • Authorization enforced in the TOOL, on the acting user's
       identity — never by the prompt
     • Sandboxed execution, egress restrictions
     • ⭐ Treat all retrieved and tool-returned content as
       untrusted input, exactly like user input
     • Spend and step limits
     • Full audit logging of every tool call
```

---

## 13. Fine-Tuning vs RAG vs Prompting

```
   ⭐ THE DECISION TREE — in the order you should try them

   ┌──────────────────────────────────────────────────────────────┐
   │ ① PROMPTING + FEW-SHOT                                       │
   │    ⭐ ALWAYS START HERE. Fastest, cheapest, most flexible.    │
   │    Solves more than people expect.                           │
   ├──────────────────────────────────────────────────────────────┤
   │ ② RAG                                                        │
   │    ⭐ When the model needs KNOWLEDGE it doesn't have:         │
   │    your documents, current data, private information.        │
   │    ✅ Updates instantly · ✅ citable · ✅ access-controllable   │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ FINE-TUNING                                                │
   │    ⭐ When you need a consistent BEHAVIOUR, FORMAT, STYLE,    │
   │    or a domain-specific pattern — not new facts.             │
   │    ✅ Shorter prompts, lower latency, lower per-call cost     │
   │    ⚠️ Needs quality labelled data · a training pipeline ·     │
   │      re-doing it on model upgrades · ⚠️ knowledge goes STALE  │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ CONTINUED PRETRAINING                                      │
   │    Rarely justified — a genuinely novel domain or language   │
   └──────────────────────────────────────────────────────────────┘

   ⭐⭐ THE MOST COMMON MISTAKE: fine-tuning to inject KNOWLEDGE.
     Fine-tuning teaches FORM, not FACTS. Facts baked into
     weights are stale the moment the data changes, can't be
     cited, and can't be permission-filtered. ⭐ Use RAG for
     knowledge; fine-tune for behaviour.

   ⭐ AND THEY COMPOSE: fine-tune for format and tone, RAG for
     the facts, prompting for the specific task.
```

---

## 14. LLM Evaluation

```
   ⚠️⚠️ EVALUATION IS THE HARDEST PART OF SHIPPING LLM FEATURES,
     and the part most often skipped.

   ⭐ WHY IT'S HARD
     • Output is open-ended — there's rarely one correct answer
     • Quality is multidimensional: correct, relevant, safe,
       well-formatted, appropriately concise
     • Outputs are non-deterministic
     • ⭐ Human evaluation is slow, expensive, and inconsistent
```

```
   ⭐ THE EVALUATION LADDER

   ┌──────────────────────────────────────────────────────────────┐
   │ ① ⭐ DETERMINISTIC CHECKS — cheap, fast, run on every change  │
   │    Valid JSON · schema conformance · required fields         │
   │    present · no forbidden content · length limits ·          │
   │    exact match where applicable                              │
   ├──────────────────────────────────────────────────────────────┤
   │ ② REFERENCE-BASED                                            │
   │    Compare to a gold answer: exact match, F1, semantic       │
   │    similarity. ⚠️ BLEU/ROUGE correlate poorly with quality    │
   │    for open-ended generation.                                │
   ├──────────────────────────────────────────────────────────────┤
   │ ③ ⭐ LLM-AS-JUDGE                                             │
   │    A model grades outputs against a rubric.                  │
   │    ✅ Scales, correlates reasonably with human judgment       │
   │    ⚠️ Biases: prefers longer answers, prefers its own         │
   │      style, position bias in pairwise comparison             │
   │    ⭐ Mitigate: a specific rubric, few-shot graded examples,  │
   │      randomize position, and ⭐ VALIDATE the judge against    │
   │      human labels                                            │
   ├──────────────────────────────────────────────────────────────┤
   │ ④ HUMAN EVALUATION                                           │
   │    The ground truth. Expensive — reserve for calibrating     │
   │    the automated evaluators and for final release gates.     │
   ├──────────────────────────────────────────────────────────────┤
   │ ⑤ ⭐ ONLINE / PRODUCTION                                      │
   │    ⭐ THE ACTUAL TRUTH: task completion, thumbs up/down,      │
   │    retry rate, escalation to a human, business outcomes      │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ BUILD AN EVAL SET BEFORE YOU BUILD THE FEATURE

   • Start with 20-50 real examples — small is fine, and far
     better than none
   • ⭐ Include the hard and adversarial cases, not just the
     happy path
   • ⭐ GROW IT FROM PRODUCTION FAILURES. Every bug report
     becomes a permanent test case. This is what makes the
     suite valuable over time.
   • Run it on every prompt change, every model upgrade, and
     every retrieval change
   • ⭐ Version it alongside the code

   ⚠️ Without this, "did that prompt change help?" is
     unanswerable, and you're tuning by vibes.
```

```
   ⭐ RAG-SPECIFIC EVALUATION — measure the two stages SEPARATELY

   RETRIEVAL   ⭐ recall@k (is the answer in the retrieved set
               AT ALL?) · precision · MRR
   GENERATION  ⭐ FAITHFULNESS — is the answer supported by the
               retrieved context, or fabricated? ·
               relevance · completeness

   ⭐ THIS SEPARATION IS ESSENTIAL. A bad answer might mean
     retrieval never found the right document, or that
     generation ignored a document it was given. Those have
     completely different fixes, and a single end-to-end score
     tells you nothing about which.
```

---

## 15. LLM Production Concerns

```
   ⭐ COST CONTROL

   ┌──────────────────────────────────────────────────────────────┐
   │ ⭐ ROUTE BY DIFFICULTY  a small model handles most requests;  │
   │                        escalate only when needed             │
   │ ⭐ CACHE                exact-match and semantic caching.     │
   │                        Prompt caching for a shared prefix    │
   │                        cuts both cost and latency            │
   │ SHORTEN CONTEXT        ⭐ retrieve less, but better           │
   │ BATCH                  offline work at lower rates           │
   │ ⭐ SET LIMITS           per-user and per-tenant spend caps —  │
   │                        LLM features can generate unbounded   │
   │                        cost in a way most features cannot    │
   └──────────────────────────────────────────────────────────────┘
```

```
   ⭐ LATENCY

   • ⭐ STREAM the response — perceived latency drops
     dramatically even though total time is unchanged. This is
     the single biggest UX win.
   • Show intermediate progress for agents
   • Smaller or faster models for latency-sensitive paths
   • Parallelize independent calls
   • ⚠️ Time-to-first-token matters more than total time for
     interactive use
```

```
   ⭐ SAFETY AND RELIABILITY

   □ ⭐ Input and output filtering for harmful content
   □ ⭐ PROMPT INJECTION defenses: treat all retrieved and
     tool-returned content as untrusted; never let the prompt
     be the authorization boundary
   □ PII detection and redaction on both input and output
   □ ⭐ Groundedness checking for RAG — flag or suppress
     unsupported claims
   □ Rate limiting per user
   □ ⭐ Graceful degradation when the provider is down or slow
   □ Full logging of prompts and responses for debugging —
     ⚠️ with PII handling and retention policy
   □ ⭐ Human-in-the-loop for consequential decisions
```

```
   ⚠️ THE ISSUES THAT SURPRISE TEAMS IN PRODUCTION

   • ⭐ NON-DETERMINISM — the same input gives different output,
     which breaks naive testing and confuses users
   • ⭐ SILENT MODEL UPDATES change behaviour underneath you →
     PIN model versions and re-run evals on upgrade
   • Long-tail inputs nobody anticipated
   • ⭐ Users treating output as authoritative — the interface
     must convey uncertainty and cite sources
   • Cost scaling unexpectedly with usage patterns
   • ⚠️ Latency spikes from provider-side load
```

---

## 16. Interview Section

<details>
<summary><b>Q1. What's different about deploying ML versus normal software?</b></summary>

The failure mode. Normal software fails loudly — an exception, a 500. An ML system keeps returning confident predictions that are quietly wrong, and nothing errors.

Second, behaviour lives in data rather than code, so a deploy changes both the logic and the distributional assumptions. And it degrades when nothing changes at all — the world moves and the model doesn't.

The practical consequence is that monitoring and evaluation aren't extras, they're the primary engineering work, because without them you can't distinguish a working system from a broken one.

The specific things I'd build: input distribution monitoring, because it alerts immediately and doesn't need labels; output distribution monitoring; and business metrics, which are what actually matter. Accuracy monitoring is valuable but often delayed by days or months waiting for labels — for a churn model you learn whether you were right thirty days later.

I'd also emphasize shadow deployment. Running a new model against real production traffic while still serving the old one gives you real-world evaluation at zero user risk, and it's underused.
</details>

<details>
<summary><b>Q2. What is train/serve skew and how do you prevent it?</b></summary>

The model performs well offline and badly in production because features at serving time differ from those it trained on.

The usual cause is two implementations. Training features are computed in SQL or Spark; serving features are computed in application code. They start identical and drift — different null handling, a different aggregation window, a subtly different definition.

The fix is one definition used by both paths, which is exactly what a feature store provides: an offline store for point-in-time-correct training data, and an online store for low-latency serving, both derived from the same definition.

The subtler and more damaging variant is point-in-time incorrectness. If you're training a churn model and you join "customer's total lifetime orders" as of today onto a label from six months ago, the feature contains information from after the prediction moment. The model looks excellent offline and is useless in production.

The rule is that every feature value must be exactly what would have been known at prediction time. That requires as-of joins, and hand-rolled pipelines get it wrong at least once — it's the single subtlest bug in applied ML.
</details>

<details>
<summary><b>Q3. Your model has 99% accuracy. Are you happy?</b></summary>

Not without knowing the base rate. If 99% of transactions are legitimate, a model that always predicts "not fraud" achieves 99% accuracy and is completely worthless.

So for imbalanced problems I'd look at precision and recall, and PR-AUC rather than ROC-AUC, since ROC-AUC can look excellent on heavily imbalanced data even when the model isn't useful.

Then the more important question: which error is costlier? For fraud, a false negative means a loss; a false positive means a frustrated legitimate customer. Those have very different costs, and the decision threshold should reflect that. I'd frame it explicitly as a business conversation — "at this threshold we catch 80% of fraud and falsely flag 2% of legitimate transactions, is that the trade you want?" — rather than treating the threshold as a modelling detail.

I'd also check calibration. If the model says 0.7, does it happen 70% of the time? That matters whenever the probability feeds a downstream decision, and it's routinely ignored.

And finally: how does it compare to the existing baseline? A model that beats random but not the current rules-based system is a regression, and that goes unnoticed more often than you'd expect.
</details>

<details>
<summary><b>Q4. Explain RAG and where it typically fails.</b></summary>

Retrieval-augmented generation grounds a model in your own data. Offline you chunk documents, embed them, and index them. At query time you embed the question, retrieve relevant chunks, and include them in the prompt so the model answers from provided context rather than parametric memory.

Failures cluster in retrieval rather than generation.

Chunking is the most under-appreciated decision. Too small and chunks lack the context to be useful; too large and embeddings become diluted averages, so retrieval precision collapses. Structure-aware chunking — splitting on sections or function boundaries — beats fixed-size splitting substantially.

Pure vector search misses exact matches: product codes, error numbers, names. Embeddings capture semantics, not literals. So hybrid search combining dense and sparse BM25 retrieval is the reliable default.

And reranking is usually the single largest quality improvement available. Retrieve fifty candidates cheaply, then use a cross-encoder to select the top five. A cross-encoder sees query and document together rather than comparing independent embeddings, which is meaningfully more accurate.

The failure people miss entirely is permissions. If the vector store holds documents from across an organization and you retrieve by pure similarity, you leak confidential material — laundered through a fluent answer that gives no hint of its source's sensitivity. Permission filtering has to happen before or during retrieval, never after generation.
</details>

<details>
<summary><b>Q5. Fine-tune or use RAG?</b></summary>

They solve different problems, and conflating them is the most common mistake in this space.

RAG is for knowledge — facts the model doesn't have, or that change. Your documentation, current inventory, customer records. It updates instantly, supports citations, and can be permission-filtered.

Fine-tuning is for behaviour — consistent format, tone, a domain-specific pattern, or a task the model does poorly with prompting alone. It lets you shorten prompts, which reduces latency and per-call cost.

The mistake is fine-tuning to inject knowledge. Fine-tuning teaches form, not facts. Facts baked into weights are stale the moment the underlying data changes, can't be cited, and can't be access-controlled per user.

My order of attack: prompting with few-shot examples first, because it solves more than people expect and is fastest to iterate. Then RAG if the problem is missing knowledge. Then fine-tuning if you need consistent behaviour that prompting can't reliably produce.

And they compose — fine-tune for format and tone, RAG for the facts, prompt for the specific task.

The practical caveat on fine-tuning is the maintenance burden: you need quality labelled data, a training pipeline, and you redo it every time you want to move to a newer base model.
</details>

<details>
<summary><b>Q6. How do you evaluate an LLM feature?</b></summary>

Build the eval set before building the feature, because otherwise "did that prompt change help?" is unanswerable and you're tuning by vibes.

Start with twenty to fifty real examples including the hard and adversarial cases. Small is fine; none is not. Then grow it from production failures — every bug report becomes a permanent test case, which is what makes the suite valuable over time.

The ladder runs from cheap to expensive. Deterministic checks first: valid JSON, schema conformance, required fields, forbidden content, length. These run on every change and catch a surprising amount.

Then LLM-as-judge with a specific rubric, which scales and correlates reasonably with human judgment. But it has known biases — preferring longer answers, preferring its own style, position bias in pairwise comparisons — so randomize position and validate the judge against human labels rather than trusting it blindly.

Human evaluation is the ground truth, reserved for calibrating the automated evaluators and for release gates.

And production signals are the actual truth: task completion, retry rate, escalation to a human, and business outcomes.

For RAG specifically, evaluate retrieval and generation separately. Recall@k for retrieval, faithfulness for generation. A bad answer might mean retrieval never found the document, or that generation ignored a document it was given — completely different fixes, and a single end-to-end score can't distinguish them.
</details>

<details>
<summary><b>Q7. What are the risks of an LLM agent with tool access?</b></summary>

Prompt injection is the central one, and it's not fully solvable at the model layer. Any untrusted content the model reads — a web page, an email, a retrieved document, even a tool's return value — can contain instructions, and the model cannot reliably distinguish data from instructions.

So the design principle is that you never rely on the model to enforce security. Controls have to live outside it.

Concretely: least privilege on tools, so an agent that can only read cannot be tricked into writing. Authorization enforced inside the tool against the acting user's identity, never by the prompt. Human approval for consequential or irreversible actions. Sandboxed execution with egress restrictions. And treating all retrieved and tool-returned content as untrusted input, exactly like user input.

Beyond security there are reliability risks: infinite loops when there's no termination condition, runaway cost, and compounding errors over long horizons. Those need step limits, spend ceilings, and full audit logging of every tool call.

The design lesson is that a good agent has few, well-described, clearly distinct tools, and returns informative errors the model can act on — opaque errors cause the model to retry identically forever.
</details>

<details>
<summary><b>Q8. Design a recommendation system.</b></summary>

Two-stage funnel, because running an expensive model over millions of candidates at high QPS is computationally impossible.

Candidate generation cuts millions to hundreds using several cheap, high-recall sources in parallel: collaborative filtering, embedding similarity via approximate nearest neighbour search, content-based matching, trending items, and an exploration bucket. Diversity of sources is what creates diversity of results.

Ranking then applies an expensive model to that shortlist, predicting multiple objectives — engagement, dwell time, and importantly the probability of negative feedback, weighted heavily downward.

Then re-ranking for business rules: diversity so one source doesn't dominate, freshness, integrity filters.

Cold start needs explicit handling on both sides. New users get popular regional content, with the interest model updating in near-real-time as they interact — fast skips are as informative as engagement. New items get placed in embedding space from content features with zero interactions, then staged exposure: show to a small cohort and promote based on engagement rate within that cohort rather than absolute counts, so new creators compete fairly.

The risk I'd raise unprompted is the feedback loop. Optimizing engagement narrows the signal, which narrows recommendations, which narrows the signal further — converging on high short-term engagement and declining retention. Countermeasures have to be deliberate: a guaranteed exploration budget, diversity constraints, penalizing repetition even when predicted engagement is high, and including retention proxies in the objective rather than immediate engagement alone. The optimizer will never find those on its own.
</details>

---

## 17. Cheat Sheet

```
╔══════════════════════════════════════════════════════════════════════╗
║                  ML & LLM SYSTEMS — ONE PAGE                         ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ ML FAILS SILENTLY — confident wrong answers, no exception          ║
║   behaviour lives in DATA · degrades when NOTHING changes            ║
║   → monitoring and evaluation ARE the engineering work               ║
║ ⭐ IMPACT ORDER: is ML even right? > data > features > framing >      ║
║   architecture (least important, gets the most attention)            ║
║ ⭐ ALWAYS establish a BASELINE (majority class / existing rules)      ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐⭐ TRAIN/SERVE SKEW = #1 production bug → ONE feature definition     ║
║   for both paths (feature store)                                     ║
║ ⭐⭐ POINT-IN-TIME CORRECTNESS: every feature must be what was KNOWN   ║
║   at prediction time. As-of joins. Leakage makes offline metrics lie.║
║ ⚠️ TEMPORAL data needs a CHRONOLOGICAL split, never random           ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⚠️ accuracy is useless when imbalanced → precision/recall, PR-AUC     ║
║ ⭐ the THRESHOLD is a BUSINESS decision (cost of FP vs FN)            ║
║ ⭐ check CALIBRATION if the probability feeds a decision              ║
║ ⭐ offline metrics are a PROXY — A/B test decides what ships          ║
╠══════════════════════════════════════════════════════════════════════╣
║ SERVING: ⭐ START WITH BATCH — many "real-time" needs are met by      ║
║   precomputed predictions. SHADOW deploy first (zero risk).          ║
║ DRIFT: data P(X) · ⭐ concept P(Y|X) (dangerous — inputs look normal) ║
║   ⚠️ LABELS ARE DELAYED → input/output monitoring is your early warning║
╠══════════════════════════════════════════════════════════════════════╣
║ RECSYS: ⭐ two-stage — cheap RECALL then expensive PRECISION          ║
║   cold start: content embeddings + ⭐ STAGED EXPOSURE on              ║
║   engagement RATE (not absolute counts)                              ║
║   ⚠️ filter bubble is STRUCTURAL → exploration + diversity must be    ║
║     deliberately built in                                            ║
╠══════════════════════════════════════════════════════════════════════╣
║ LLM: predicts the next token — ⭐ context window IS the memory,       ║
║   confident fabrication is the EXPECTED failure mode                 ║
║   ⚠️ "lost in the middle" → put key content at the START or END      ║
║ ⭐ RAG: ⭐ CHUNKING is the biggest decision · HYBRID search (dense +   ║
║   BM25, since embeddings miss exact codes) · ⭐ RERANK with a         ║
║   cross-encoder = biggest single quality win                         ║
║   ⚠️⚠️ RETRIEVAL MUST RESPECT PERMISSIONS or you leak, fluently       ║
║ ⭐ prompting → RAG (knowledge) → fine-tune (BEHAVIOUR, not facts)     ║
║   ⚠️ fine-tuning for knowledge is the classic mistake                 ║
╠══════════════════════════════════════════════════════════════════════╣
║ ⭐ AGENTS: prompt injection is unsolvable at the model layer →        ║
║   least-privilege tools · authz IN THE TOOL · human approval for     ║
║   irreversible actions · step and spend limits · audit every call    ║
║ ⭐ EVAL: build the set BEFORE the feature · grow it from PRODUCTION   ║
║   FAILURES · deterministic → LLM-judge (validate it!) → human →      ║
║   production signals.  ⭐ Measure RETRIEVAL and GENERATION separately.║
║ PROD: ⭐ STREAM (biggest UX win) · route by difficulty · cache ·      ║
║   ⭐ PIN model versions (silent updates change behaviour) · spend caps║
╚══════════════════════════════════════════════════════════════════════╝
```

---

**Related:** [Data Engineering](data-engineering.md) · [System Design](../05-system-design/00-fundamentals.md) · [AppSec](../07-security/appsec.md)
