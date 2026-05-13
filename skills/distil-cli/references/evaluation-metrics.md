# Evaluation Metrics

When evaluating teacher models or trained SLMs, Distil Labs uses different metrics depending on your task type.

## Text Generation Metrics

Used for question answering, classification, open book QA, closed book QA, and other text generation tasks.

### LLM-as-a-Judge (Recommended)

A large language model acts as a human grader to evaluate whether the answer is semantically correct. Scores reflect quality even when wording differs from the reference answer.

- **What it measures:** Semantic correctness -- does the answer mean the right thing, regardless of phrasing?
- **When to use:** Most text generation tasks. Best for tasks where many valid phrasings are possible.
- **Returns:** A quality score reflecting how well the answer matches the reference meaning.

### Exact-Match (Binary)

Checks whether the model output exactly matches the reference answer, character for character.

- **What it measures:** Exact textual equivalence.
- **When to use:** Facts with one correct phrasing, classification labels, or tasks requiring a specific output string.
- **Returns:** 1 if the output exactly matches the reference, 0 otherwise.
- **Limitation:** Harsh on synonyms and paraphrases. A correct answer worded differently scores 0.

### ROUGE-L

Measures the longest common subsequence (word overlap) between the model output and the reference answer.

- **What it measures:** How much wording is shared between the two texts.
- **When to use:** Summarization tasks and cases where reusing reference phrasing matters.
- **Returns:** A score from 0 to 1. Higher values indicate more shared wording.
- **Limitation:** Favors longer answers that reuse reference phrases. Does not capture semantic equivalence.

### METEOR

Balances precision and recall while rewarding correct synonyms and penalizing incoherent text.

- **What it measures:** Word overlap with credit for synonyms and word stems, plus a fluency penalty.
- **When to use:** Tasks where synonyms and paraphrasing are acceptable. Often tracks human judgments better than pure overlap metrics.
- **Returns:** A score from 0 to 1.

## Tool Calling Metrics

Used for tool calling and multi-turn tool calling tasks.

### tool_call_equivalence (Recommended)

Compares the prediction and reference with intelligent handling of default values. Parameters that were not explicitly set are treated as having their default values.

- **What it measures:** Whether the predicted tool call is functionally equivalent to the reference, accounting for defaults.
- **When to use:** Most tool calling evaluation. Best for real-world correctness where unset parameters fall back to defaults.
- **Returns:** 1 if the tool calls are equivalent, 0 otherwise.

### binary_tool_call

Compares the prediction and reference as exact dictionaries. All keys must be present with identical values. Key order does not matter. Does not account for default parameter values.

- **What it measures:** Strict dictionary equivalence between prediction and reference.
- **When to use:** Strict validation where all parameters must be explicitly provided and exactly correct.
- **Returns:** 1 if exactly equivalent, 0 otherwise.

### staged_tool_call

Evaluates predictions incrementally across four stages. Useful during development to understand where the model is failing.

| Stage | Score | What it checks |
|-------|-------|----------------|
| 1 | 0.25 | Is the output valid JSON? |
| 2 | 0.50 | Is the function name correct? |
| 3 | 0.75 | Are the parameter keys correct? |
| 4 | 1.00 | Is the full prediction an exact match? |

**Interpreting staged_tool_call scores:**

- **0.25** -- Valid JSON but wrong function called.
- **0.50** -- Right function but wrong parameter keys or values.
- **0.75** -- Right function and parameter keys but wrong values.
- **1.00** -- Perfect match.

## Interpretation Guide

### Text Generation Scorecards

| Scenario | What it means | Action |
|----------|---------------|--------|
| High LLM-as-a-Judge, low Exact-Match | Answers are semantically correct but worded differently from the reference. | This is usually fine. Consider adding alternate phrasings to your reference set if exact wording matters. |
| High ROUGE-L, low LLM-as-a-Judge | Model is copying words from the reference but missing the actual meaning. | Revisit your task description and training examples for clarity. |
| All metrics low | The task may be under-specified or the data quality may be insufficient. | Revisit the task description, add more context, improve data quality, or check for inconsistencies in the dataset. |
| High LLM-as-a-Judge, high ROUGE-L | Model is both semantically correct and uses similar wording to the reference. | Strong performance. Proceed with confidence. |

### Tool Calling Scorecards

| Scenario | What it means | Action |
|----------|---------------|--------|
| High tool_call_equivalence, low binary_tool_call | Model produces correct calls but omits parameters that have defaults. | This is usually fine -- the calls are functionally equivalent. |
| Low staged_tool_call (around 0.25) | Model produces valid JSON but calls the wrong function. | Check that your tool descriptions are distinct enough and training examples cover each tool. |
| Low staged_tool_call (around 0.50) | Model picks the right function but gets parameters wrong. | Improve parameter descriptions in your tool schemas and add more varied training examples. |

## Quick Reference: Recommended Metrics by Task Type

| Task Type | Recommended Metric | Other Available Metrics |
|-----------|--------------------|------------------------|
| Question Answering | LLM-as-a-Judge | Exact-Match, ROUGE-L, METEOR |
| Classification | LLM-as-a-Judge | Exact-Match, ROUGE-L, METEOR |
| Open Book QA (RAG) | LLM-as-a-Judge | Exact-Match, ROUGE-L, METEOR |
| Closed Book QA | LLM-as-a-Judge | Exact-Match, ROUGE-L, METEOR |
| Tool Calling | tool_call_equivalence | binary_tool_call, staged_tool_call |
| Multi-Turn Tool Calling | tool_call_equivalence | binary_tool_call, staged_tool_call |
