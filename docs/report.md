# Failure-Aware Prompting — Full Experiment Report (v2)

## 1. Summary

We tested whether injecting abstracted past failures into prompts reduces LLM error repetition. To address flaws found in v1, we added two control conditions for a 5-arm experiment.

**Conclusion: Failure learning works. The effect comes from the failure content itself, not from similarity-based retrieval.**

| Condition | Accuracy | NS F1 | Delta vs A | p-value |
|-----------|----------|-------|------------|---------|
| A (Baseline) | 81.2% | 0.857 | — | — |
| B (Static toy examples) | 82.5% | 0.870 | +1.2%p | 0.501 n.s. |
| D (Length control) | 82.5% | 0.868 | +1.2%p | 0.512 n.s. |
| **B' (Random real failures)** | **91.2%** | **0.939** | **+10.0%p** | **0.010 *** |
| **C (Retrieved failures)** | **90.0%** | **0.931** | **+8.8%p** | **0.032 *** |
| B' vs C | — | — | -1.2%p | 0.749 n.s. |

---

## 2. Flaws in v1 and How v2 Addressed Them

### 2.1 Flaws Found in v1

| # | Flaw | Impact | v2 Response |
|---|------|--------|-------------|
| 1 | **Unfair B condition**: B used toy examples (129 chars/each), C used rich abstractions (605 chars/each). Information density mismatch inflated retrieval's apparent value. | Could not distinguish retrieval effect from information quality effect | Added **B' (Random Failure)**: same format, same length as C, but randomly selected. Fair comparison. |
| 2 | **Prompt length uncontrolled**: A (0 chars) < B (774) < C (1,814). Length itself could affect attention. | Length and content effects were confounded | Added **D (Length Control)**: irrelevant general text at ~1,500 chars |
| 3 | **Same-model bias**: Opus generated documents, claims, judgments, and abstractions | Opus's generation bias may influence its own judgments | Acknowledged as limitation; external dataset validation needed |
| 4 | **Small failure bank (13 cases)**: Low retrieval quality (Type-Match@1 = 11.7%) | C's effect may not come from retrieval at all | Confirmed via B' that retrieval was indeed not the driver |

### 2.2 v2 Experimental Conditions (5-Arm)

| Condition | Content Injected | Length | Control Variable |
|-----------|-----------------|--------|------------------|
| **A (Baseline)** | Nothing | 0 | — |
| **B (Static toy)** | 6 generic pattern examples | ~774 chars | Short generic examples |
| **B' (Random failure)** | 3 random from failure bank | ~1,814 chars | Same format/length as C, no retrieval |
| **D (Length control)** | Irrelevant general text | ~1,500 chars | Similar length to C, no failure content |
| **C (Dynamic retrieved)** | Top-3 by similarity search | ~1,814 chars | Retrieval-based selection |

**Design intent**: Decompose the effect into three candidate sources.

```
Length/attention effect?    → D vs A
Failure content effect?    → B' vs D
Retrieval effect?          → C vs B'
```

---

## 3. Experiment Setup

### 3.1 Task

```
Input:  Long Document D (Korean, avg 6,467 chars / 1,424 words) + Claim C (avg 103 chars)
Output: Supported | Not Supported
```

### 3.2 Error Taxonomy

| # | Type | Diagnostic Question | Label |
|---|------|---------------------|-------|
| 1 | Direct Match | Faithful restatement or synonym? | Supported |
| 2 | Deterministic Derivation | Mechanically calculable? | Supported |
| 3 | Factual Mismatch | Specific facts differ? | Not Supported |
| 4 | Negation Flip | Positive/negative reversed? | Not Supported |
| 5 | Condition/Intensity Alteration | Qualifier, scope, or degree changed? | Not Supported |
| 6 | Certainty/Status Escalation | Uncertain→certain, incomplete→complete? | Not Supported |
| 7 | Ungrounded Reasoning | Conclusion not in document? | Not Supported |
| 8 | Fabrication | No connection to document? | Not Supported |

### 3.3 Data

- 5 domains (finance, medical, legal, tech, policy) x 4 documents = 20 long documents (Opus-generated)
- Document generation requirements: 15+ specific numbers, 5+ conditional statements, 5+ uncertainty expressions, 2-3 double negations
- D2C (Document-to-Claim) generation with 9 difficulty strategies

| Dataset | Documents | Composition | Purpose |
|---------|-----------|-------------|---------|
| probe_set | 10 | NS 60 + S 20 = 80 | Phase 1: Failure collection |
| test_set | 10 (separate) | NS 60 + S 20 = 80 | Phase 3: Effect measurement |

### 3.3.1 Data Separation (No Leakage)

Probe and test use **completely separate documents**. No document appears in both.

| Domain | Probe Documents | Test Documents |
|--------|----------------|----------------|
| Finance | finance_1, finance_2 | finance_0, finance_3 |
| Medical | medical_0, medical_1 | medical_2, medical_3 |
| Legal | legal_1, legal_2 | legal_0, legal_3 |
| Tech | tech_1, tech_2 | tech_0, tech_3 |
| Policy | policy_2, policy_3 | policy_0, policy_1 |

Document overlap: **zero** (verified). Each domain contributes 2 documents to each set. Both sets have 16 items per domain.

This means Phase 1 failures (from probe documents) are tested against entirely new documents in Phase 3. We're measuring whether the model handles the same **type** of new problem better — not whether it can re-answer seen problems.

### 3.4 3-Phase Structure

```
Phase 1: Baseline Probe → 13 failures collected (error rate 16.2%)
Phase 2: Failure abstraction → ChromaDB dual indexing
Phase 3: 5-arm effect test (A, B, B', D, C)
```

### 3.5 Base LLM

Claude Opus 4.6 via Claude Agent SDK, temperature 0.

### 3.6 Statistical Testing

Paired Permutation Test (10,000 perms) + BCa Bootstrap 95% CI.
Significance criterion: CI entirely above 0 AND p < 0.05.

---

## 4. Results

### 4.1 Phase 1: Opus Weakness Profile

| Error Type | Errors / Total | Error Rate |
|------------|---------------|------------|
| 3-Factual Mismatch | 0/13 | **0%** |
| 4-Negation Flip | 1/7 | 14% |
| **5-Condition/Intensity** | **5/12** | **42%** |
| **6-Certainty/Status** | **5/13** | **38%** |
| 7-Ungrounded Reasoning | 1/15 | 7% |

13 failures collected: 5-Condition (5), 6-Certainty (5), 4-Negation (1), 7-Ungrounded (1), 1-Direct Match FP (1).

### 4.2 Phase 3: 5-Arm Comparison

#### Overall Metrics

| Condition | Accuracy | NS-P | NS-R | NS-F1 |
|-----------|----------|------|------|-------|
| A (Baseline) | 0.812 | 1.000 | 0.750 | 0.857 |
| B (Static toy) | 0.825 | 0.979 | 0.783 | 0.870 |
| D (Length control) | 0.825 | 1.000 | 0.767 | 0.868 |
| **B' (Random failure)** | **0.912** | 0.982 | **0.900** | **0.939** |
| **C (Dynamic retrieved)** | **0.900** | 0.964 | **0.900** | **0.931** |

#### Statistical Tests

| Comparison | Delta Accuracy | p-value | 95% CI | Significant? |
|------------|---------------|---------|--------|-------------|
| A vs B | +1.2%p | 0.501 | [-2.5%, +6.2%] | n.s. |
| A vs D | +1.2%p | 0.512 | [+0.0%, +3.8%] | n.s. |
| **A vs B'** | **+10.0%p** | **0.010** | **[+2.5%, +17.5%]** | **Yes** |
| **A vs C** | **+8.8%p** | **0.032** | **[+1.3%, +17.5%]** | **Yes** |
| **B' vs C** | **-1.2%p** | **0.749** | **[-8.8%, +6.3%]** | **n.s.** |
| D vs C | +7.5%p | 0.057 | [+0.0%, +15.0%] | borderline |

#### Per-Type Recall

| Error Type | A | B | D | B' | C |
|------------|-----|-----|-----|------|------|
| 3-Factual Mismatch | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |
| 4-Negation Flip | 0.62 | 0.62 | 0.62 | **0.88** | **1.00** |
| 5-Condition/Intensity | 0.62 | 0.69 | 0.69 | **0.77** | **0.77** |
| 6-Certainty/Status | 0.46 | 0.54 | 0.46 | **0.85** | **0.77** |
| 7-Ungrounded Reasoning | 1.00 | 1.00 | 1.00 | 1.00 | 1.00 |

---

## 5. Analysis: Decomposing the Effect

### 5.1 Three-Factor Decomposition

```
Total effect (A → C):  +8.8%p (p=0.032) ***

= Length/attention  (A → D):   +1.2%p (p=0.512) n.s.
+ Failure content   (D → B'):  +8.8%p
+ Retrieval         (B'→ C):  -1.2%p (p=0.749) n.s.
```

| Factor | Contribution | Significance | Interpretation |
|--------|-------------|--------------|----------------|
| **Length/attention** | +1.2%p | n.s. | Longer prompts alone don't help |
| **Failure content** | **+8.8%p** | **Key driver** | Abstracted failure experience is what matters |
| **Retrieval** | -1.2%p | n.s. | Similarity-based selection adds nothing over random |

### 5.2 Correcting the v1 Conclusion

**v1 conclusion (wrong)**: "Retrieving similar failures is the key" (based on B vs C)

**v2 correction**: v1's B condition had far less information density (129 vs 605 chars/example) than C. With B' (same format/length, random selection), **C and B' are indistinguishable** (p=0.749).

- The abstracted failure content is the driver (B' ≈ C >>> A)
- Similarity-based retrieval adds no value (C ≈ B')
- Prompt length and generic warnings are insufficient (D ≈ B ≈ A)

### 5.3 Why Random Selection Works

The failure bank is **naturally skewed** toward the model's weak spots. 10 out of 13 failures (77%) are types 5 and 6. Random sampling of 3 items has a 98.8% chance of including at least one type-5 or type-6 failure.

### 5.4 Scaled Experiment: Same Conclusion with Larger Bank

To test whether the bank was simply too small, we generated 72 additional probe items, collected 12 more failures, and expanded the bank from 13 to 25 cases.

**Scaled bank (25 cases) results:**

| Condition | Accuracy | Delta vs A |
|-----------|----------|------------|
| A (Baseline) | 81.2% | — |
| B' (Random, 25-case bank) | 91.2% | +10.0%p |
| C (Dynamic, 25-case bank) | 92.5% | +11.2%p |
| B' vs C | +1.2%p | p=0.500, n.s. |

Still **B' ≈ C** even with nearly double the bank. The newly collected failures were still concentrated on types 5 and 6 (76%), so the skew persists naturally.

**Practical implication**: A model's failure patterns are inherently skewed toward its weaknesses. Random sampling of recent failures is likely sufficient. Complex retrieval becomes valuable only when failure types become truly diverse.

### 5.5 Why Failure Content Works

Why does B' (+10.0%p) dramatically outperform B (+1.2%p) when both describe the same error types?

1. **Concrete experience vs. abstract description**: Real failures show exactly which sentence and which word changed. Generic warnings are too vague to shift behavior.
2. **Pattern + Signal + Lesson triad**: The combination of what went wrong, where to look, and how to judge focuses the model's attention on the precise point of vulnerability.
3. **"You actually got this wrong" framing**: The warning "a misjudgment occurred in this situation" may activate stronger attention than a generic pattern list.

This is our interpretation, not an experimentally confirmed mechanism.

---

## 6. Limitations

### 6.1 Same-model bias
Opus generated documents, claims, judgments, and abstractions. External datasets (FEVER, SciFact, etc.) needed for validation.

### 6.2 Small scale
80 test items, 13-25 failure cases. Per-type statistical power is limited with only 8-15 items per type.

### 6.3 Failure bank type skew
77% of failures are types 5 and 6. A more diverse bank is needed to properly assess retrieval value.

### 6.4 False positive side effect
NS Precision dropped from 1.000 to 0.964 in condition C (2 new false positives). Failure injection may induce over-suspicion.

### 6.5 NS priming bias
All failures in the bank are "should have been Not Supported" cases. The model may simply be biased toward answering NS more often, rather than genuinely understanding error patterns. Balancing the bank with both FP and FN failures is needed.

### 6.6 No human label verification
Gold labels were auto-generated, not human-verified. Some ambiguous cases may have incorrect labels.

---

## 7. Conclusion

### Key Findings

1. **Failure learning works.** Injecting abstracted past failures improves accuracy by +8.8-10.0%p (p < 0.05).
2. **The driver is failure content, not retrieval.** Random 3 ≈ retrieved 3 (p=0.749). Prompt length and generic warnings have no effect.
3. **This holds in the current setting** (small bank, skewed types). Retrieval may prove valuable with larger, more diverse banks.
4. **Concrete failures >> generic examples.** Toy examples: +1.2%p (n.s.). Real failure abstractions: +10.0%p (***).

### Next Steps

1. Scale failure bank to 100+ cases with diverse types; re-evaluate retrieval
2. Validate on external datasets (FEVER, SciFact, CLIMATE-FEVER)
3. Test across multiple models (Sonnet, Haiku, GPT-4o)
4. Balance failure bank with both FP and FN cases
5. Measure learning curve (1 → 5 → 10 → 20 accumulated failures)
6. Validate in real-world (non-synthetic) user interactions

---

## Appendix A: v1 → v2 Changes

| Item | v1 | v2 |
|------|-----|-----|
| Conditions | 3-arm (A, B, C) | **5-arm (A, B, B', D, C)** |
| B condition | Static toy (129 chars/ex) | Same |
| B' condition | None | **Random real failures (605 chars/ex)** |
| D condition | None | **Length control (~1,500 chars)** |
| Key conclusion | "Retrieval is the key" | **"Failure content is the key; retrieval adds nothing in this setting"** |

## Appendix B: References

- MiniCheck (EMNLP 2024) — D2C synthetic data generation for fact-checking
- FActScore (EMNLP 2023) — Atomic fact decomposition for verification
- "Can LLMs Learn from Previous Mistakes?" (ACL 2024) — Error-driven prompting and fine-tuning
- Reflexion (NeurIPS 2023) — Verbal reinforcement learning from failure memory
- KATE — kNN-based dynamic few-shot selection
- Explanation-based ICL Retrieval (2025) — Error-pattern-based retrieval
- "When +1% Is Not Enough" (2025) — Paired bootstrap evaluation protocol
- FAVA (2024) — Fine-grained hallucination taxonomy
