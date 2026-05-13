# Data Preparation Overview

Preparing your dataset is the first step in training a small language model with Distil. This guide covers the shared structure and requirements that apply across all task types.

## Directory Structure

Your data directory should contain the following files:

| File | Format | Required | Description |
|------|--------|----------|-------------|
| `job_description.json` | JSON | Yes | Task description defining what the model should do |
| `train.csv` or `train.jsonl` | CSV or JSONL | Yes | 20+ labeled training examples |
| `test.csv` or `test.jsonl` | CSV or JSONL | Yes | Held-out evaluation set (20+ examples) |
| `config.yaml` | YAML | Yes | Task type and training hyperparameters |
| `unstructured.csv` or `unstructured.jsonl` | CSV or JSONL | No | Domain-relevant text for synthetic data generation |

## job_description.json

The job description tells the platform what the model should do. Think of it as an LLM prompt that guides both the teacher model and the student model.

### Common fields (all task types)

- **`task_description`** (required) -- Describes what the model should do and how it should format its outputs.
- **`llm_as_a_judge_instructions`** (optional) -- Instructions for the LLM evaluator that judges model outputs. The judge outputs binary values (`good`/`bad`).

### Task-specific fields

| Task Type | Additional Fields |
|-----------|-------------------|
| `classification` | `classes_description` -- A map from class names to their descriptions |
| `tool-calling-closed-book` | `tools` -- List of tool schemas in OpenAI function calling format |
| `multi-turn-tool-calling-closed-book` | `tools` -- List of tool schemas in OpenAI function calling format |
| `question-answering` | `input_description` (required) -- Describes what the input data looks like |
| `question-answering-open-book` | None beyond common fields |
| `question-answering-closed-book` | None beyond common fields |

## Supported Formats

Both **CSV** and **JSONL** are supported for train, test, and unstructured files. You can use either format -- the platform handles both identically.

- **CSV**: Standard comma-separated values with a header row
- **JSONL**: One JSON object per line, each containing the required column keys

The file extension determines the parser used, so use `.csv` for CSV files and `.jsonl` for JSONL files.

## Minimum Requirements

- **Training data**: At least 20 labeled examples in `train.csv`/`train.jsonl`
- **Test data**: At least 20 labeled examples in `test.csv`/`test.jsonl`
- The platform generates synthetic data from your examples, so a few dozen high-quality examples is enough to start

## Unstructured Data (unstructured.csv)

The unstructured dataset provides domain-relevant text that the teacher model uses to generate diverse, domain-specific synthetic training data. It has a single column: `context`.

**JSONL format:**
```json
{"context": "Your domain-specific text here..."}
{"context": "Another passage of relevant content..."}
```

**CSV format:**

| context |
|---------|
| Your domain-specific text here... |
| Another passage of relevant content... |

The role of unstructured data varies by task type:

| Task Type | Role of Unstructured Data |
|-----------|---------------------------|
| `question-answering` | Sample documents for generating diverse QA pairs |
| `classification` | Unlabelled examples or domain documentation for new example generation |
| `tool-calling-closed-book` | Domain-specific scenarios for generating tool call examples |
| `multi-turn-tool-calling-closed-book` | Domain-specific scenarios for generating conversation examples |
| `question-answering-open-book` | Context passages for generating grounded QA pairs |
| `question-answering-closed-book` | **Critical** -- This is how knowledge gets embedded into the model. The system generates QA pairs from these contexts |

## config.yaml

The minimal configuration specifies the task type:

```yaml
base:
  task: <task-type>
```

For advanced options (model selection, training parameters, etc.), see the configuration reference.

## Choosing a Task-Type Format

Once your directory structure is ready, follow the task-specific guide for your data columns and job_description.json format:

| Task Type | Config Value | Data Guide |
|-----------|-------------|------------|
| Question Answering | `question-answering` | [question-answering.md](./question-answering.md) |
| Classification | `classification` | [classification.md](./classification.md) |
| Tool Calling | `tool-calling-closed-book` | [tool-calling.md](./tool-calling.md) |
| Multi-Turn Tool Calling | `multi-turn-tool-calling-closed-book` | [multi-turn-tool-calling.md](./multi-turn-tool-calling.md) |
| Open Book QA (RAG) | `question-answering-open-book` | [open-book-qa.md](./open-book-qa.md) |
| Closed Book QA | `question-answering-closed-book` | [closed-book-qa.md](./closed-book-qa.md) |

## Validation Checklist

Run these deterministic checks before uploading. In Claude Code, verify these programmatically by reading the files.

> **An exception is a failure.** If a validation script raises any exception (TypeError, KeyError, JSONDecodeError, etc.) — that's a failure, not a passing run. Read the traceback, fix the underlying issue, and re-run the validator. Do not declare data "correct" if the script errored out before completing. Past sessions have moved on after a `TypeError: object of type 'NoneType' has no len()` and only caught the data issue much later in the workflow.

- [ ] Train file has ≥ 20 rows
- [ ] Test file has ≥ 20 rows
- [ ] All required columns present (check the task-specific guide for column names)
- [ ] No empty values in required columns
- [ ] For classification: every class in `classes_description` appears at least once in train data
- [ ] For tool calling: every `answer` value parses as valid JSON
- [ ] For multi-turn tool calling: every `question` value parses as a valid JSON array
- [ ] File extension matches format (`.csv` for CSV, `.jsonl` for JSONL)
- [ ] `job_description.json` is valid JSON
- [ ] `config.yaml` is valid YAML with at least `base.task` set

## Quality Assessment (Judgment Required)

After passing the mechanical checks above, assess these qualitatively:

1. **Representative examples** -- Do the examples cover the variety of inputs expected in production? Include different phrasings, edge cases, and difficulty levels.
2. **Consistent formatting** -- Is the answer format consistent across examples? If answers should be short phrases, there should not be paragraph-length responses mixed in.
3. **Accurate labels** -- Does every example have a clearly correct answer? Ambiguous or contradictory labels degrade model quality.
4. **Production-like inputs** -- Does the input text resemble what the model will see in production (similar length, vocabulary, formatting)?
5. **Quality over quantity** -- A few dozen high-quality, representative examples outperform hundreds of noisy ones. The platform generates synthetic data to expand your dataset.
