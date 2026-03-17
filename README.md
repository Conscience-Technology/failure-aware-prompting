# Failure-Aware Prompting

**Can LLMs stop repeating the same mistakes if you show them their past failures?**

We ran an experiment: collect an LLM's actual failures, abstract them into reusable patterns, and inject them into prompts when it faces new problems. Accuracy went from 81.2% to 90.0% (p=0.032).

But the mechanism wasn't what we expected.

---

## Key Findings

### 1. Failure injection works.

Showing abstracted past failures improved document-grounded fact verification accuracy by +8.8 percentage points (p < 0.05). The improvement concentrated on the model's weak spots — subtle error types like qualifier removal and certainty escalation.

### 2. The content matters, not the retrieval.

We initially thought similarity-based retrieval from a vector DB was the key. **It wasn't.** Randomly selecting 3 failures from the bank worked just as well as targeted search (91.2% vs 90.0%, p=0.749). Generic warnings and longer prompts alone had no effect.

### 3. Why random works (for now).

The model's failures naturally concentrate on a few error types. With a small bank, random sampling almost always includes relevant cases. Retrieval may prove its value when the bank grows large and diverse — that's the next experiment.

---

## The Experiment at a Glance

```
Phase 1: Baseline Probe
  80 items → Opus judges → 13 failures collected (error rate: 16.2%)

Phase 2: Failure Learning
  13 failures → abstracted into pattern / signal / lesson → stored in vector DB

Phase 3: Effect Test (5-arm comparison on 80 NEW items, separate documents)
  A  Baseline                81.2%
  B  Generic warnings        82.5%   (+1.2%p)  n.s.
  D  Length control           82.5%   (+1.2%p)  n.s.
  B' Random real failures    91.2%  (+10.0%p)  ***
  C  Retrieved failures      90.0%   (+8.8%p)  ***
```

Probe and test sets use **completely separate documents** — no data leakage.

### Effect Decomposition

```
Total effect (A → C):  +8.8%p  ***

  Length/attention  (A → D):   +1.2%p   n.s.   → Not the cause
  Failure content   (D → B'):  +8.8%p          → The driver
  Retrieval         (B'→ C):  -1.2%p   n.s.   → No added value
```

### Per-Type Improvement

| Error Type | Baseline | With Failures | Delta |
|------------|----------|---------------|-------|
| Factual Mismatch | 100% | 100% | — |
| Negation Flip | 62% | 100% | +38%p |
| Condition/Intensity | 62% | 77% | +15%p |
| Certainty/Status | 46% | 77% | +31%p |
| Ungrounded Reasoning | 100% | 100% | — |

---

## How Failures Are Abstracted

Each failure is distilled into three domain-agnostic components:

```
Pattern:  What went wrong?
          "A conditional premise was dropped, turning a qualified
           statement into an absolute one"

Signal:   Where should you look?
          "A clause like 'subject to...' is missing from the claim"

Lesson:   How should you judge next time?
          "When a claim makes an affirmative judgment, always check
           whether the source attaches conditions or qualifiers"
```

This triad is what gets injected into the prompt. The specificity of real failure abstractions (vs. generic warnings) is what drives the effect.

[Full abstraction methodology →](methodology/failure_abstraction.md)

---

## Error Taxonomy

We classify every document-claim pair using an 8-type sequential taxonomy:

| # | Type | Question | Label |
|---|------|----------|-------|
| 1 | Direct Match | Faithful restatement? | Supported |
| 2 | Deterministic Derivation | Mechanically calculable? | Supported |
| 3 | Factual Mismatch | Facts (numbers/entities/dates) differ? | Not Supported |
| 4 | Negation Flip | Positive ↔ negative reversed? | Not Supported |
| 5 | Condition/Intensity Alteration | Qualifier/scope/degree changed? | Not Supported |
| 6 | Certainty/Status Escalation | Uncertain→certain, incomplete→complete? | Not Supported |
| 7 | Ungrounded Reasoning | Conclusion not in document? | Not Supported |
| 8 | Fabrication | No connection to document? | Not Supported |

[Full taxonomy with examples →](methodology/error_taxonomy.md)

---

## What We Got Wrong (and How We Fixed It)

Our first experiment (v1) concluded that similarity-based retrieval was the key driver. **This was wrong.** The control condition used low-information toy examples (129 chars each) while the dynamic condition used rich abstractions (605 chars each). The comparison was unfair.

We added two controls in v2 — random real failures (same format/length, no retrieval) and a length-matched irrelevant text condition — and discovered that retrieval added no value. The failure content itself was what mattered.

We scaled the failure bank from 13 to 25 cases and re-ran. Same conclusion (p=0.500).

[Full v1 → v2 analysis →](docs/report.md)

---

## Limitations (Honestly)

We want to be upfront about what this experiment cannot tell you.

**NS priming bias.** Every failure in the bank is a "this should have been Not Supported" case. The model may be biased toward answering NS more often rather than genuinely understanding the error pattern. NS Precision dropped from 1.000 to 0.964 (2 new false positives).

**Same-model loop.** Claude Opus generated the documents, generated the claims, judged them, and abstracted the failures. Validation on external human-labeled datasets (FEVER, SciFact, etc.) is needed.

**Small scale.** 80 test items, 13-25 failure cases. Per-type statistical power is limited.

**Easy Supported cases.** All Supported test items were simple paraphrases. The real false positive risk may be underestimated.

---

## Next Steps

- Scale the failure bank to 100+ cases with diverse error types, then re-evaluate whether retrieval adds value
- Validate on external datasets (FEVER, SciFact, CLIMATE-FEVER)
- Test across multiple models (Sonnet, Haiku, GPT-4o)
- Balance the failure bank with both FP and FN cases to address priming bias
- Measure the learning curve: how does performance change with 1, 5, 10, 20 accumulated failures?

---

## Practical Takeaway

```
MVP:  Collect failures per user → abstract them → inject recent N into prompt.
      No retrieval needed. Simple is enough.

Later: Add vector search when you have hundreds of failures
       across many different task types.
```

---

## Repository Structure

```
failure-aware-prompting/
├── README.md                          ← You are here
├── methodology/
│   ├── error_taxonomy.md              ← 8-type error classification system
│   ├── failure_abstraction.md         ← Pattern/signal/lesson framework
│   ├── experiment_design.md           ← 5-arm design rationale
│   └── prompts.md                     ← Prompt templates (system, injection format)
├── results/
│   └── summary.json                   ← All numerical results
├── docs/
│   ├── report.md                      ← Full experiment report (v2)
│   └── blog.md                        ← Blog post version
└── LICENSE
```

---

## Technical Details

| Component | Detail |
|-----------|--------|
| Base LLM | Claude Opus 4.6 via Claude Agent SDK |
| Embedding | Google Gemini (gemini-embedding-001, 768d) |
| Vector DB | ChromaDB (dual indexing + Reciprocal Rank Fusion) |
| Statistics | Paired Permutation Test (10k perms) + BCa Bootstrap 95% CI |
| Data | 5 domains, 20 long documents (avg 1,424 words), 160 claims |
| Separation | Document-level split, zero overlap between probe and test |

---

## References

- Reflexion (NeurIPS 2023) — Verbal reinforcement learning from failure memory
- LEMA (2024) — Learning from mistakes makes LLM better reasoner
- "Can LLMs Learn from Previous Mistakes?" (ACL 2024) — Error-driven prompting and fine-tuning
- MiniCheck (EMNLP 2024) — D2C synthetic data generation for fact-checking
- FActScore (EMNLP 2023) — Atomic fact decomposition for verification
- FAVA (2024) — Fine-grained hallucination taxonomy
- "When +1% Is Not Enough" (2025) — Paired bootstrap evaluation protocol

---

*This is an early-stage exploration on synthetic data. Results are encouraging but cannot be generalized without reproduction on external datasets and multiple models. We've tried to clearly separate confirmed findings from open questions.*

**Built by [Conscience Technology](https://github.com/Conscience-Technology)**
