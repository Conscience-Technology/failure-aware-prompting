# Can We Stop LLMs from Repeating the Same Mistakes?

*An experiment in accumulating failure cases and injecting them into prompts*

---

## The Recurring Problem

If you've spent any time using LLMs for real work, you've probably experienced a strange sense of deja vu. You saw the model make a particular type of mistake last week, and today it makes the exact same kind of error. The document says "may contribute to," but the model confidently restates it as "contributes to." A critical qualifier gets quietly dropped. The same blind spot, surfacing again and again.

A human would learn from this. After misreading a hedge expression once, you'd start paying closer attention to words like "may," "could," or "is expected to." But LLMs don't carry that lesson into the next conversation. Every session starts from zero.

So we asked:

> **If we collect an LLM's actual failures and show them back to it, will it make fewer mistakes of the same type on new problems?**

Intuitively, this seems plausible. But intuition isn't enough, so we ran an experiment. The short answer is yes, it works. But the mechanism was different from what we initially expected — and along the way, we caught ourselves making a flawed conclusion that we had to correct.

This post is the full record of that process.

---

## Experiment Design

### The Task: Document-Grounded Fact Verification

We chose a straightforward task: give the model a long Korean-language document (average 1,400 words — roughly 3 pages) and a claim, and ask whether the document supports the claim.

```
Input:  Document D (1,400 words) + Claim C (1-2 sentences)
Output: Supported | Not Supported
```

We picked this task for specific reasons. First, the binary output makes evaluation unambiguous. Second, the constraint that the model must judge based solely on the provided document eliminates reliance on external knowledge. Third, it's a task that shows up constantly in practice — report review, contract verification, research fact-checking.

### How Hard Did We Make It?

Easy problems don't tell you much. We deliberately crafted distortions subtle enough that **Claude Opus 4.6 would actually get them wrong.**

Here's a real example from the experiment:

```
Document: "...expected to capture approximately 700,000 tons of CO2 annually."
           (Korean: "포집할 수 있을 것으로 전망된다" — "could be expected to")

Claim:    "...expected to capture approximately 700,000 tons of CO2 annually."
           (Korean: "포집할 것으로 전망된다" — "is expected to")

Difference: Two characters deleted ("수 있을" = "could")
Label:      Not Supported (possibility escalated to certainty)
```

Can you spot the difference? In the Korean original, a two-character modal expression meaning "could" was deleted, turning a possibility into a definitive forecast. We generated distortions like this using 9 systematic strategies:

- **Near-miss numbers**: Change 12.0 billion to 12.2 billion — barely noticeable
- **Qualifier removal**: "Effective in patients aged 65+" becomes "Effective in patients" — the key condition silently disappears
- **Hedge removal**: "may have an effect" becomes "has an effect" — one word changes the certainty level
- **Plausible inference**: Revenue is up and R&D spending is up, so "the growth strategy is succeeding" — sounds reasonable, but the document never said that

We created 20 long documents across 5 domains (finance, medical, legal, tech, policy) and applied these strategies to generate 160 test items total.

### Error Taxonomy: 8 Types

We classified every claim into this hierarchy. Types 1-2 are Supported, 3-8 are Not Supported.

```
Supported (document backs the claim)
  1. Direct Match — Faithful restatement or synonym
  2. Deterministic Derivation — Mechanically calculable from document

Not Supported (document does NOT back the claim)
  3. Factual Mismatch — Numbers, entities, or dates differ
  4. Negation Flip — Positive ↔ negative reversed
  5. Condition/Intensity — Qualifier, scope, or degree altered
  6. Certainty/Status — Uncertain→certain, incomplete→complete
  7. Ungrounded Reasoning — Conclusion not stated in document
  8. Fabrication — No connection to any sentence in document
```

This classification matters because it lets us analyze later which specific types of errors get better (and which don't).

---

## Experiment Structure: 3 Phases

```
┌─────────────────────────────────────────────────────────┐
│  Phase 1: Baseline Probe                                │
│  "Let it fail first"                                    │
│                                                         │
│  probe_set (80 items) → Opus judges → 13 failures       │
│  collected (error rate: 16.2%)                          │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  Phase 2: Failure Learning                              │
│  "Abstract the failures"                                │
│                                                         │
│  13 failures → distilled into pattern/signal/lesson     │
│             → stored in vector DB                       │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│  Phase 3: Effect Test                                   │
│  "Measure the effect on new problems"                   │
│                                                         │
│  test_set (80 items, entirely separate docs)            │
│  × 5 conditions                                        │
└─────────────────────────────────────────────────────────┘
```

One point to emphasize: the probe_set (where we collect failures) and the test_set (where we measure the effect) use **completely different documents.** No overlap at all. We're not testing whether the model can re-answer the same question — we're testing whether it handles the same **type** of new problem better.

---

## Phase 1: Where Does Opus Fail?

The goal of Phase 1 is simple: have Opus judge 80 items and observe what it gets wrong.

The results were striking — not because Opus performed well, but because its weaknesses were so clearly defined.

```
Error rate by type (Not Supported items only)

3-Factual Mismatch        ████████████████████ 0%      ← Perfect
4-Negation Flip           ██████████████████   14%
5-Condition/Intensity     ███████████          42%     ← Misses nearly half
6-Certainty/Status        ████████████         38%     ← Misses nearly half
7-Ungrounded Reasoning    ██████████████████   7%
```

Explicit factual mismatches (type 3) and baseless inferences (type 7) are caught almost perfectly. But when a **qualifier is quietly dropped (type 5)** or **a possibility is subtly promoted to a certainty (type 6)**, Opus misses about 40%.

This makes intuitive sense. A wrong number jumps out when you compare it against the source. But detecting that a modal verb was removed, or that a condition clause was silently dropped, requires meticulous comparison of tone and scope across a 1,400-word document. That's hard even for humans.

---

## Phase 2: Abstracting Failures

We didn't just show the raw failures. We took each one and converted it into a **domain-agnostic abstract pattern.** Whether the mistake happened in a carbon policy document or a clinical trial report, the underlying error structure is the same.

Each failure was distilled into three components:

```
┌─ Abstracted Failure Case ─────────────────────────────┐
│                                                       │
│  Pattern:  What went wrong?                           │
│            "A conditional premise was dropped,         │
│             turning a qualified claim into an          │
│             absolute one"                             │
│                                                       │
│  Signal:   Where should you look?                     │
│            "A clause like 'subject to...' or          │
│             'under the condition that...' is missing   │
│             from the claim"                           │
│                                                       │
│  Lesson:   How should you judge next time?            │
│            "When a claim makes an affirmative          │
│             judgment, check whether the source         │
│             attaches conditions or qualifiers"         │
│                                                       │
└───────────────────────────────────────────────────────┘
```

This triad — pattern (what), signal (where), lesson (how) — turned out to matter a lot, as the results will show.

---

## Phase 3: The First Attempt, and What We Got Wrong

### We Started with 3 Conditions

In our first experiment (v1), we compared:

- **A (Baseline)**: No assistance
- **B (Static)**: 6 generic error pattern examples always shown
- **C (Dynamic)**: 3 similar failure cases retrieved from vector DB

The results looked clean: A (81.2%) < B (82.5%) < C (90.0%). We concluded: "Retrieving similar failures is the key!"

**That conclusion was wrong.**

### What Was Flawed

When we dug deeper, an uncomfortable fact emerged. Condition B's examples were 129 characters each — brief, generic descriptions. Condition C's retrieved failures were 605 characters each — rich, detailed abstractions. **The information density differed by more than 4x.**

With that gap, we couldn't tell whether B < C reflected the value of retrieval or simply the value of showing more detailed text.

There was another confound too. The total injected text was 0 chars for A, 774 for B, and 1,814 for C. Maybe longer prompts just make the model read more carefully?

### Adding Controls: v2

So we added two more conditions:

```
Condition     Content injected                       Length       Tests whether...
─────────     ────────────────                       ──────       ──────────────
A             Nothing                                0 chars      Baseline
B             Generic toy examples ×6                ~774 chars   Generic warnings help
D (new)       Irrelevant general text                ~1,500 chars Length/attention matters
B'(new)       Random 3 from failure bank             ~1,814 chars Failure content itself helps
C             Top-3 by similarity search             ~1,814 chars Retrieval adds value
```

B' is the critical addition. It uses the **exact same format, same length, same failure cases** as C — but selected randomly instead of by similarity. If B' equals C, retrieval is unnecessary. If C beats B', retrieval adds genuine value.

---

## Results: It Works, But Not How We Expected

### Overall Accuracy

```
A  Baseline             ██████████████████████████████████████████   81.2%
B  Static (toy)         ███████████████████████████████████████████  82.5%  (+1.2%p)
D  Length Control        ███████████████████████████████████████████  82.5%  (+1.2%p)
B' Random Failure       ██████████████████████████████████████████████████  91.2%  (+10.0%p) ***
C  Dynamic Retrieved    █████████████████████████████████████████████████   90.0%  (+8.8%p)  ***

*** = Statistically significant (p < 0.05, Paired Permutation Test)
```

Only B' and C are significantly better than baseline. And B' and C are **statistically indistinguishable** (p=0.749).

### Decomposing the Effect

The 5-arm design lets us cleanly separate three possible drivers:

```
Total effect (A → C):  +8.8%p  ***

  ┌──────────────────────────────────────────────┐
  │  Length/attention   (A → D):  +1.2%p   n.s.  │  Longer prompt?    → No
  │  Failure content    (D → B'): +8.8%p         │  Failure examples? → This is it
  │  Retrieval          (B'→ C): -1.2%p   n.s.  │  Smart selection?  → No added value
  └──────────────────────────────────────────────┘
```

Three takeaways:

1. Longer prompts and generic warnings alone don't help (D, B ≈ A)
2. **Showing abstracted real failure experiences helps substantially** (B', C >> A)
3. Similarity-based retrieval isn't necessary — random selection works just as well (B' ≈ C)

Point 3 surprised us. Why would random selection match targeted retrieval?

### Why Random Selection Works

The answer is simple once you see it. Out of 13 failures in the bank, 10 (77%) are types 5 and 6. When you randomly pick 3, the probability of getting **zero** type-5 or type-6 failures is just 1.2%.

Opus's failures naturally concentrate on its weak spots. So random sampling almost always includes relevant cases. We scaled the bank to 25 failures — still 76% types 5+6, and B' vs C remained identical (p=0.500).

This might actually be good news. It means you don't need a complex retrieval system. **Just showing the most recent N failures is likely enough.** Retrieval becomes worthwhile only when failure types become truly diverse — e.g., hundreds of failures across many different task types.

### Which Error Types Improved?

```
                          Baseline(A)    With failures(C)    Delta
3-Factual Mismatch           100%            100%              -
4-Negation Flip               62%            100%           +38%p
5-Condition/Intensity         62%             77%           +15%p
6-Certainty/Status            46%             77%           +31%p
7-Ungrounded Reasoning       100%            100%              -
```

No change on types Opus already handles perfectly (3, 7) — there's no room to improve from 100%. The gains concentrate exactly on its **weak spots (4, 5, 6)**. Type 6 (certainty escalation) went from catching less than half to catching three-quarters.

---

## Why Don't Generic Warnings Work?

You might wonder: doesn't a generic instruction like "check certainty levels carefully" convey the same idea? Why does B (generic examples) give only +1.2%p while B' (real failure abstractions) gives +10%p?

We believe the difference comes down to **specificity.**

Condition B looks like this:
```
Pattern: Possibility expression escalated to certainty
Signal: "may" → "does" transformation
Example: "may have an effect" → "effect has been proven"
```

Condition B' looks like this:
```
Pattern: Modal hedging removed, promoting uncertain forecast to definitive statement
Signal: "could be expected to" reduced to "is expected to" — modal "could" deleted
Document excerpt: "...subject to green hydrogen costs falling below $3.0/kg..."
Incorrect Claim: "announced a plan to complete commercialization by 2030."
Lesson: Always verify whether qualifying conditions are preserved in the claim
```

Both point at the same error type. But the latter shows a **concrete case** — which sentence, which word changed, and what went wrong as a result. This specificity appears to focus the model's attention on exactly the right thing.

That said, this is our interpretation, not an experimentally confirmed mechanism.

---

## What We Still Don't Know

We don't want to oversell these results. Several issues remain unresolved, and some could challenge the conclusions themselves.

**1. We can't tell if the model "understands" the pattern or just answers "Not Supported" more often.**

Every failure in the bank is a Not Supported case. The injected context always says "this should have been NS." The model might simply be biased toward answering NS rather than genuinely understanding the error pattern. In fact, NS Precision dropped from 1.000 to 0.964 — 2 new false positives appeared where Supported items were incorrectly flagged as Not Supported.

In real-world usage, both directions of failure (FP and FN) would accumulate, which should naturally counterbalance this bias. But we haven't verified that experimentally.

**2. The same model generated and judged everything.**

Claude Opus created the documents, generated the claims, made the judgments, and abstracted the failures. It's a closed loop. Validation on external, human-labeled datasets (FEVER, SciFact, etc.) is needed before these results can be trusted as general findings.

**3. Small scale.**

80 test items, 13-25 failure cases. With only 8-15 items per error type, per-type statistical power is limited.

**4. The Supported test cases are too easy.**

All 20 Supported items in the test set are simple paraphrases that Opus gets 100% right. We're likely underestimating the real false positive risk.

---

## Summary

### What We Confirmed

- **Injecting abstracted failure cases into prompts improves accuracy** — 81.2% to 90%+, statistically significant (p < 0.05)
- The effect comes from **failure content itself**, not prompt length or generic warnings
- **Concrete failure experience is far more effective than generic pattern examples** — +10%p vs +1.2%p for the same error types
- Improvement concentrates on the model's **existing weak spots**

### What We Haven't Confirmed

- Whether similarity-based retrieval adds value (not observed here; needs a larger, more diverse failure bank)
- Whether the effect is genuine pattern understanding or just NS-direction priming
- Reproducibility across other models and external datasets
- Effectiveness in real-world (non-synthetic) settings

### The Practical Takeaway

If there's one thing to take from this experiment:

```
┌───────────────────────────────────────────────────────────┐
│                                                           │
│   MVP: Collect user failures, abstract them,              │
│   inject the most recent N into the prompt. That's it.    │
│                                                           │
│   No retrieval system needed up front.                    │
│   Add vector search later, when you have hundreds of      │
│   failures across many different task types.              │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## Technical Details

| Component | Detail |
|-----------|--------|
| Base LLM | Claude Opus 4.6 via Claude Agent SDK |
| Embedding | Google Gemini (gemini-embedding-001, 768d) |
| Vector DB | ChromaDB (dual indexing + RRF) |
| Statistics | Paired Permutation Test (10,000 perms) + BCa Bootstrap 95% CI |
| Domains | finance, medical, legal, tech, policy |
| Data scale | probe 80 + test 80 (document-level separation, zero overlap) |
| Total LLM calls | ~700 |

---

*This experiment is an early exploration on small-scale synthetic data. The results are encouraging, but cannot be generalized without reproduction on external datasets and multiple models. We've tried to clearly separate what we confirmed from what we haven't.*
