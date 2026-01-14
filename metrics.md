# Understanding Evaluation Metrics

When evaluating teacher models or trained SLMs, Distil Labs uses different metrics depending on your task type. This guide explains how to interpret each metric.

## Text Generation Metrics

Used for question answering, classification, and other text generation tasks.

### LLM-as-a-Judge (Recommended)

A large language model acts as a human grader to evaluate if the answer is semantically correct. Scores reflect quality even when wording differs from the reference answer.

**Best for:** Tasks where many valid phrasings are possible.

### Exact-Match (Binary)

Returns 1 if the model output exactly matches the reference answer, 0 otherwise.

**Best for:** Facts with one correct phrasing. Harsh on synonyms or paraphrases.

### ROUGE-L

Measures the longest common subsequence (word overlap) between the answer and reference. Higher scores indicate more shared wording.

**Best for:** Summarization tasks. Favors longer answers that reuse reference phrases.

### METEOR

Balances precision and recall, rewards correct synonyms, and penalizes incoherent text. Often tracks human judgments better than pure overlap metrics.

**Best for:** Tasks where synonyms and paraphrasing are acceptable.

### Interpreting Text Generation Results

| Scenario | What it means |
|----------|---------------|
| Low Exact-Match, High LLM-as-a-Judge | Answers are correct but paraphrased — this is often fine |
| All metrics low | Task may be under-specified — revisit task description or add more context |
| High ROUGE-L, Low LLM-as-a-Judge | Model is copying words but missing the meaning |

---

## Tool Calling Metrics

Used for tool calling tasks.

### tool_call_equivalence (Recommended)

Compares prediction and reference with intelligent handling of default values. Parameters that weren't explicitly set are treated as having their default values.

**Returns:** 1 if tool calls are equivalent, 0 otherwise.

**Best for:** Real-world correctness where unset parameters fall back to defaults.

### binary_tool_call

Compares prediction and reference as exact dictionaries. All keys must be present with identical values (key order doesn't matter). Does not account for default parameter values.

**Returns:** 1 if exactly equivalent, 0 otherwise.

**Best for:** Strict validation where all parameters must be explicit.

### staged_tool_call

Evaluates predictions incrementally across four stages (useful for debugging):

| Stage | Score | What it checks |
|-------|-------|----------------|
| 1 | 0.25 | Valid JSON? |
| 2 | 0.50 | Correct function name? |
| 3 | 0.75 | Correct parameter keys? |
| 4 | 1.00 | Exact match? |

**Interpreting staged_tool_call:**
- **0.25** — Valid JSON but wrong function called
- **0.50** — Right function but wrong parameters
- **0.75** — Right function and keys but wrong values
- **1.00** — Perfect match

---

## Quick Reference

| Task Type | Recommended Metric |
|-----------|-------------------|
| Question Answering | LLM-as-a-Judge |
| Classification | LLM-as-a-Judge |
| Open Book QA (RAG) | LLM-as-a-Judge |
| Closed Book QA | LLM-as-a-Judge |
| Tool Calling | tool_call_equivalence |
