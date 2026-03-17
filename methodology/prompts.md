# Prompt Templates

## System Prompt (All Conditions)

Used identically across all 5 experimental conditions.

```
You are an expert in document-based fact verification.
Judge whether the Claim is supported by the given Document alone.

Criteria:
- Supported: Directly stated in the Document or mechanically derivable from it
- Not Supported: All other cases

Important: Do not use any external knowledge beyond the Document.

Respond in this JSON format only:
{"judgment": "Supported" or "Not Supported", "reason": "(one sentence)"}
```

Note: The actual experiment used Korean-language prompts. The above is a faithful translation.

## User Prompt (All Conditions)

```
Document:
{full document text}

Claim:
{claim text}
```

## Dynamic Failure Injection (Condition C)

Appended to the system prompt, before the user prompt.

```
[Warning] A similar situation led to a misjudgment in the past.
Do not repeat the same mistake.

=== Past Misjudgment 1 ===
Error type: {error_type}
Pattern: {abstract_pattern}
Signal: {signal}
Document excerpt: "{document_excerpt}"
Incorrect Claim: "{claim}"
Previous misjudgment: {model_prediction} → Correct answer: {gold_label}
Lesson: {correct_behavior}

=== Past Misjudgment 2 ===
...

=== Past Misjudgment 3 ===
...
```

## Static Few-Shot (Condition B)

6 fixed, generic examples — one per error type (3-8). These are simple, short descriptions unrelated to any specific document.

```
[Warning] Below are common misjudgment patterns. Keep them in mind.

=== Pattern 1: Factual Mismatch ===
Pattern: Specific number replaced with a different value
Signal: Numbers in document and claim don't match
Example document: "Company revenue was $12 billion."
Example claim: "Company revenue was $15 billion."
Correct judgment: Not Supported — compare numbers precisely

=== Pattern 2: Negation Flip ===
...

(6 patterns total, ~129 chars each)
```

## Random Failure Injection (Condition B')

Same format as Condition C, but failures are randomly sampled from the bank instead of retrieved by similarity. This is the critical control — it matches C in format, length, and content richness, but without retrieval.

## Length Control (Condition D)

General text about fact verification methodology (~1,500 chars), matched to C's injection length. Contains no failure cases or error-specific guidance.
