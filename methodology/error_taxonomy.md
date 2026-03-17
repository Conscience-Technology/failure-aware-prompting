# Error Taxonomy: 8-Type Sequential Classification

## Overview

We classify every (Document, Claim) pair by applying these questions **in order**. The first matching type becomes the final label.

## Supported Types (1-2)

### 1. Direct Match
**Question**: Does the claim restate the document faithfully, using the same words or synonyms?

Sub-tags: Direct Statement, Paraphrase

**Example**:
- Document: "The company reported revenue of $12 billion in 2024."
- Claim: "In 2024, the company's revenue reached $12 billion."
- Label: **Supported**

### 2. Deterministic Derivation
**Question**: Can the claim be mechanically derived from the document's information (e.g., arithmetic)?

**Example**:
- Document: "Revenue was $12B and costs were $8B."
- Claim: "The profit margin was approximately 33%."
- Label: **Supported** (12-8=4, 4/12 ≈ 33%)

---

## Not Supported Types (3-8)

### 3. Factual Mismatch
**Question**: Does the claim differ from the document on specific facts (entities, numbers, dates)?

Sub-tags: Entity Mismatch, Number Mismatch, Temporal Mismatch

**Example**:
- Document: "Company A's 2024 revenue was $12.0 billion."
- Claim: "Company A's 2024 revenue was $12.2 billion."
- Label: **Not Supported** — Near-miss number change

### 4. Negation Flip
**Question**: Has a positive/negative been reversed?

**Example**:
- Document: "The system cannot detect anomalies in real-time."
- Claim: "The system can detect anomalies in real-time."
- Label: **Not Supported**

### 5. Condition/Intensity Alteration
**Question**: Has a qualifier, scope, or degree been changed so the meaning differs from the original?

Sub-tags: Qualifier/Scope Shift, Degree/Intensity Shift, Soft Generalization, Mild Degree Shift

**Example**:
- Document: "The treatment is effective in patients aged 65 and older."
- Claim: "The treatment is effective in patients."
- Label: **Not Supported** — Age qualifier removed, scope expanded

### 6. Certainty/Status Escalation
**Question**: Has something uncertain/incomplete/correlational been presented as certain/complete/causal?

Sub-tags: Probabilistic Leap, Causal Overattribution, Tense/Modality Shift, Hedged Probabilistic

**Example**:
- Document: "Early results suggest the therapy may be effective."
- Claim: "The therapy has been proven effective."
- Label: **Not Supported** — "may be" escalated to "proven"

### 7. Ungrounded Reasoning
**Question**: Does the claim draw a conclusion the document does not directly state?

Sub-tags: Inference, Speculative Multi-hop, Common-sense Completion

**Example**:
- Document: "Revenue increased 20%. R&D spending increased 15%."
- Claim: "The company's growth strategy is succeeding."
- Label: **Not Supported** — Plausible but not stated

### 8. Fabrication
**Question**: Can the claim not be connected to any sentence in the document?

**Example**:
- Document: "Python is a popular programming language."
- Claim: "Python's creator announced retirement in 2024."
- Label: **Not Supported** — No connection to the document

---

## Application Rules

1. **Sequential**: Apply questions in order (1 through 8). The first match is the final label.
2. **Boundary between 5 and 6**: Ask "Was the change about condition/scope/degree, or about certainty/completion/causality?"
3. **Supported threshold**: Only types 1 and 2. Everything else is Not Supported — even if the claim seems "mostly right."
