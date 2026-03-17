# Experiment Design: 5-Arm Controlled Comparison

## Why 5 Conditions?

We didn't start with 5. We started with 3 (Baseline, Static, Dynamic) and got a misleading result. Adding 2 control conditions in v2 revealed that our initial conclusion was wrong.

This document explains the rationale for each condition.

## The 5 Conditions

| Condition | What's Injected | Length | Purpose |
|-----------|----------------|--------|---------|
| **A (Baseline)** | Nothing | 0 | Baseline performance |
| **B (Static toy)** | 6 generic pattern examples | ~774 chars | Does any warning help? |
| **D (Length control)** | Irrelevant general text | ~1,500 chars | Does prompt length help? |
| **B' (Random failure)** | 3 random from failure bank | ~1,814 chars | Does failure content help? |
| **C (Dynamic retrieved)** | 3 retrieved by similarity | ~1,814 chars | Does retrieval add value? |

## What Each Comparison Isolates

```
A vs B:   Generic warnings vs nothing
A vs D:   Longer prompt vs nothing
A vs B':  Real failure content vs nothing
A vs C:   Retrieved failures vs nothing
B' vs C:  Retrieval vs random selection (THE KEY COMPARISON)
D vs B':  Failure content vs irrelevant text of same length
```

## Why B' Is the Critical Addition

In v1, we compared B (generic, short) against C (specific, long) and attributed the difference to retrieval. But B and C differed in **two ways simultaneously**: information quality AND information quantity.

B' controls for this by using the **same format, same length, same real failures** as C — just without similarity-based selection. This isolates the retrieval component.

## Data Separation

Probe and test sets use entirely separate documents. No document appears in both.

| Domain | Probe Documents | Test Documents |
|--------|----------------|----------------|
| Finance | finance_1, finance_2 | finance_0, finance_3 |
| Medical | medical_0, medical_1 | medical_2, medical_3 |
| Legal | legal_1, legal_2 | legal_0, legal_3 |
| Tech | tech_1, tech_2 | tech_0, tech_3 |
| Policy | policy_2, policy_3 | policy_0, policy_1 |

Document overlap: **zero** (verified).

This ensures we're testing whether the model handles the same **type** of new problem better — not whether it can re-answer problems it's seen before.

## Statistical Method

- **Paired Permutation Test** (10,000 permutations): Non-parametric, minimal distributional assumptions
- **BCa Bootstrap 95% CI** (5,000 samples): Bias-corrected confidence intervals
- **Significance criterion**: CI entirely above 0 AND p < 0.05

Based on the protocol recommended by "When +1% Is Not Enough" (2025).

## The v1 → v2 Correction

| | v1 | v2 |
|--|-----|-----|
| Conditions | 3 (A, B, C) | 5 (A, B, B', D, C) |
| Conclusion | "Retrieval is the key" | "Failure content is the key; retrieval adds nothing in this setting" |
| Why v1 was wrong | B was unfairly weak (129 chars vs 605 chars per example) | B' provides fair comparison at equal information density |
