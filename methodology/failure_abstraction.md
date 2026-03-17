# Failure Abstraction: Pattern / Signal / Lesson

## Why Abstract?

Raw failures are tied to specific documents and domains. A mistake in a carbon policy report and a mistake in a clinical trial paper may share the same underlying structure — a conditional clause was dropped, turning a qualified statement into an absolute one.

By abstracting failures into domain-agnostic components, we make them transferable across contexts.

## The Three Components

### Pattern (What went wrong?)

A structural description of the error, independent of domain.

> "A conditional premise was removed from the claim, converting a qualified possibility into an unconditional assertion."

This should be recognizable regardless of whether the original document was about finance, medicine, or law.

### Signal (Where should you look?)

The specific linguistic or structural cue that distinguishes the document from the claim.

> "The document's clause 'subject to green hydrogen costs falling below $3.0/kg' is completely absent from the claim."

This tells the model exactly what kind of discrepancy to scan for.

### Lesson (How should you judge?)

An actionable instruction for future judgment.

> "When a claim contains an affirmative judgment, always verify whether the source document attaches conditions, qualifiers, or prerequisites. If any are missing from the claim, judge Not Supported."

## Real Examples from Our Experiment

### Example 1: Condition/Intensity Alteration

| Field | Content |
|-------|---------|
| **Pattern** | Conditional premise removed from claim, converting a qualified plan into an unconditional commitment |
| **Signal** | Document states the plan is "subject to green hydrogen costs falling below $3.0/kg" — this prerequisite is absent from the claim |
| **Lesson** | Even when the conclusion matches, verify that all qualifying conditions are preserved. Missing conditions → Not Supported |

### Example 2: Certainty/Status Escalation

| Field | Content |
|-------|---------|
| **Pattern** | Modal hedging expression removed, promoting an uncertain forecast to a definitive statement |
| **Signal** | Document's "could be expected to" (possibility) reduced to "is expected to" (certainty) — the modal "could" was deleted |
| **Lesson** | Compare the certainty level of the document's language against the claim's. If hedge words (may, could, possibly) are missing, judge Not Supported |

### Example 3: Ungrounded Reasoning

| Field | Content |
|-------|---------|
| **Pattern** | Two independently stated facts combined to draw a causal conclusion not present in the document |
| **Signal** | Document states A and B separately; claim asserts A caused B without the document making this connection |
| **Lesson** | Co-occurrence in a document does not imply causation. If the document doesn't explicitly link two facts, a causal claim is Not Supported |

## Injection Format

When injected into the prompt, an abstracted failure looks like this:

```
[Warning] A similar situation led to a misjudgment in the past.

=== Past Misjudgment 1 ===
Error type: 5-Condition/Intensity Alteration
Pattern: Conditional premise removed, making a qualified statement absolute
Signal: "subject to green hydrogen cost falling below $3.0/kg" was deleted
Document excerpt: "...commercialization plan, subject to green hydrogen costs..."
Incorrect Claim: "announced a plan to complete commercialization by 2030."
Lesson: Always verify whether qualifying conditions are preserved in the claim
```

## Why This Works Better Than Generic Warnings

Our experiment showed a stark difference:

- **Generic warning** ("Check certainty levels carefully"): +1.2%p (not significant)
- **Abstracted real failure** (pattern + signal + lesson from an actual mistake): +10.0%p (p=0.010)

The specificity of real failure cases — showing which sentence, which word changed, and what went wrong — focuses the model's attention on exactly the right thing. A generic warning is too vague to change behavior.
