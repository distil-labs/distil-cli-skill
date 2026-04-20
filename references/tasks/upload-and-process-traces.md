# Upload and Process Traces

Train a model from production LLM interaction logs instead of manually curated datasets. The platform automatically filters, relabels, and splits your traces into training and test data.

## What Are Traces

Traces are logs of real interactions with an LLM in production. Instead of hand-labeling examples, you upload these logs and the platform processes them into structured training data automatically.

**Example use cases:**
- Bootstrapping a fine-tuned model from existing production chat logs
- Improving a deployed model by training on its own successful interactions
- Converting multi-turn conversation logs into individual training examples
- Rapidly creating training data without manual labeling effort

## Task Compatibility

Trace processing does **not** support contextual (open-book) tasks. The following task types work with `upload-traces`:

| Task type | Supported |
|-----------|-----------|
| `question-answering` | Yes |
| `classification` | Yes |
| `tool-calling-closed-book` | Yes |
| `multi-turn-tool-calling-closed-book` | Yes |
| `question-answering-closed-book` | Yes |
| `question-answering-open-book` | **No** — trace processing cannot separate context from question automatically. If your production traces contain RAG-style prompts where the retrieved context is embedded in the user message, use `question-answering` instead and put the full prompt (context + question) in the `question` field. |

## Upload Traces

The recommended approach is to place all required files in a directory and upload it:

```bash
distil model upload-traces <model-id> --data <directory>
```

The directory should contain:

| File | Required | Description |
|------|----------|-------------|
| `traces.jsonl` | Yes | Production traces in JSONL format |
| `job_description.json` | Yes | Task objectives and configuration |
| `config.yaml` or `config.json` | Yes | Training and trace processing parameters |
| `test.jsonl` or `test.csv` | No | Optional curated test set |

### Individual File Flags

As an alternative to directory mode, specify each file individually:

```bash
distil model upload-traces <model-id> \
  --traces <file> \
  --job-description <file> \
  --config <file> \
  [--test <file>]
```

| Flag | Required | Description |
|------|----------|-------------|
| `--data` | Yes* | Directory containing trace files. |
| `--traces` | Yes* | Path to traces file (`.jsonl`). |
| `--job-description` | Yes* | Path to job description file (`.json`). |
| `--config` | Yes* | Path to config file (`.json` or `.yaml`). |
| `--test` | No | Path to a curated test data file (`.jsonl` or `.csv`). |

\* Provide either `--data` or all three individual file flags (`--traces`, `--job-description`, `--config`), but not both.

## Trace Formats

The format of your `traces.jsonl` file is controlled by the `observation_format` parameter in the `trace_processing` section of your config. There are three supported formats:

### openai_messages (Default)

Each line contains a `messages` array of OpenAI chat completion messages:

```json
{"messages": [{"role": "system", "content": "You are a helpful assistant."}, {"role": "user", "content": "What is the capital of France?"}, {"role": "assistant", "content": "The capital of France is Paris."}]}
{"messages": [{"role": "user", "content": "Translate 'hello' to Spanish."}, {"role": "assistant", "content": "Hola"}]}
```

> **Gotcha: Use ASCII-safe JSON.** When writing `traces.jsonl`, use `ensure_ascii=True` in Python (or equivalent in other languages). The platform's JSON parser treats Unicode LINE SEPARATOR (`U+2028`) and PARAGRAPH SEPARATOR (`U+2029`) as line terminators and will silently truncate records that contain them. These characters can appear in scraped web content and are valid JSON per the spec, so standard JSON validators won't catch the issue. If you can't use `ensure_ascii=True`, explicitly replace `\u2028` and `\u2029` with their escaped forms before writing.

### langfuse

Langfuse observation objects with `id`, `input`, and `output` fields:

```json
{"id": "obs-1", "input": [{"role": "user", "content": "What is 2+2?"}], "output": {"role": "assistant", "content": "4"}}
```

### unstructured_with_openai_messages

Unstructured data combined with OpenAI messages format. Use this when your traces contain a mix of structured conversations and freeform text.

## Trace Processing Pipeline

When you upload traces, the platform runs four stages:

1. **Filtering** — drop traces not relevant to the task.
2. **Relabelling** — a committee of teachers rewrites labels; the teacher picks the best.
3. **Splitting** — split into train + test; leftover traces become unstructured context.
4. **Multi-turn handling** — `convert_to_single_turn: true` splits by assistant turn; `false` preserves conversations.

For the end-to-end flow diagram and the `trace_processing` parameters gating each stage, see `references/platform-overview.md`. Per-parameter semantics live in the `Trace Processing Configuration` section below and in `references/configuration.md`.

## Test Set Behavior

There are two options for the test set used in evaluation:

**Option 1: Provide your own test set.** Include a `test.jsonl` or `test.csv` file in your upload directory (or pass via `--test`). This test set is used as-is for teacher evaluation and training evaluation, without any processing.

**Option 2: Let the platform create one (default).** If you do not provide a test set, the platform creates one automatically as part of trace processing. It selects traces, runs them through the same filtering and relabelling pipeline, and uses the result as the test set. The platform also evaluates the original model (the one that generated the traces) on this test set, giving you a baseline to compare against your trained SLM.

## Trace Processing Configuration

The `trace_processing` section in your config controls how traces are processed:

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-3B-Instruct
  teacher_model_name: openai.gpt-oss-120b

trace_processing:
  relabel: true
  convert_to_single_turn: true
  num_traces_as_training_base: 200
  num_traces_as_testing_base: 200
  observation_format: openai_messages
```

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `relabel` | `true` | Improve label quality via a committee of teacher models. |
| `convert_to_single_turn` | `true` for single-turn tasks, `false` for multi-turn tasks | Splits multi-turn conversations into individual training examples. Keep `true` for single-turn tasks (QA, classification, single-turn tool calling). **Set to `false` for `multi-turn-tool-calling-closed-book`** — the model needs to see conversations whole, so splitting them into single-turn examples destroys the signal you want to train on. When `false`, conversations are preserved with committee-based rewriting. |
| `num_traces_as_training_base` | `200` | Number of traces to use as the training base. Recommended: set equal to `num_traces_as_testing_base`. |
| `num_traces_as_testing_base` | `200` | Number of traces to use as the testing base. Recommended: set equal to `num_traces_as_training_base` (keeps train/test seeded from comparable trace volumes). |
| `observation_format` | `openai_messages` | Trace format: `openai_messages`, `langfuse`, or `unstructured_with_openai_messages`. |
| `remove_system_prompt_from_traces` | `false` | Remove large system prompts from traces. |
| `compress_job_description` | `false` | Compress long job descriptions before relevance filtering. |

## Reprocessing Traces

After the initial upload, you can try different processing parameters without re-uploading the trace files:

```bash
# Using a trace processing config file (only trace_processing parameters)
distil model reprocess-traces <model-id> --trace-processing-config <file>

# Or using a full config file (only the trace_processing section is used)
distil model reprocess-traces <model-id> --config <file>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--trace-processing-config` | `-t` | Path to trace processing config file (`.json` or `.yaml`). |
| `--config` | `-c` | Path to full config file -- only the `trace_processing` section is used. |

Provide either `--trace-processing-config` or `--config`, but not both. You must have uploaded traces with `upload-traces` first.

## Checking Status

Check the upload and processing status:

```bash
distil model upload-status <model-id>
```

For machine-readable output:

```bash
distil model upload-status <model-id> --output json
```

Once processing completes, continue with teacher evaluation and training:

```bash
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

## Common Gotchas

1. **`validation_max_total_length` applies to traces too** — The default limit of 10,000 characters applies to both uploaded traces/test data and generated synthetic data, not just synthgen output. If your production traces contain long inputs (e.g., full documents, injected schemas), you will hit this limit during trace processing. Increase it in your config:
   ```yaml
   synthgen:
     validation_max_total_length: 30000
   ```

2. **Updating `job_description.json` requires re-uploading** — There is no way to update the job description in place. If you need to change it (e.g., to add a missing required field), you must re-run `upload-traces`, which triggers full trace processing including committee relabelling. Plan your job description carefully before uploading.

3. **Markdown fences in relabeled JSON answers** — When `synthgen.output_is_json: true`, committee relabeling models sometimes wrap JSON in ```` ```json ... ``` ```` fences. Teacher evaluation will then fail JSON validation. Workaround: download the relabeled train/test, strip markdown fences from `answer` fields, validate every `answer` parses as JSON, and re-upload as a regular dataset with `distil model upload-data`. Then proceed to teacher evaluation.

4. **`input_description` must be self-contained** — The synthgen model that generates training data does NOT see `task_description`. It only sees `input_description`. So `input_description` must fully describe the input structure on its own — markers, sections, examples, formatting. If you only put input details in `task_description`, synthgen will produce poor data even if teacher evaluation looks fine.

## Tips

1. **Provide enough traces** -- Hundreds to thousands of traces is ideal for good results.
2. **Keep relabelling enabled** -- Use `relabel: true` (the default) to improve label quality via a committee of teacher models. The committee approach produces more consistent and accurate labels than any single model.
3. **Iterate with reprocess** -- If the processed data does not look right, use `reprocess-traces` to try different parameters without re-uploading.
4. **Multi-turn conversations** -- Match `convert_to_single_turn` to the target task. For single-turn tasks (QA, classification, single-turn tool calling), keep `true` (the default) to convert multi-turn conversations into individual training examples. **For `multi-turn-tool-calling-closed-book`, set `convert_to_single_turn: false`** — preserving full conversations with committee-based rewriting is required to retain the conversational context the multi-turn model learns from.
5. **Cap large trace sets** -- Tune `num_traces_as_training_base` and `num_traces_as_testing_base` to control how many traces feed into seed generation. Unused traces become unstructured context. Set the two to the **same value** so train and test are seeded from comparable trace volumes; diverging them skews the train/test distribution.
6. **Strip large system prompts** -- If your traces have very large system prompts, set `remove_system_prompt_from_traces: true` to prevent them from dominating the data.
7. **Compress long job descriptions** -- If your job description is very long, set `compress_job_description: true` to compress it before relevance filtering.
8. **Provide your own test set** -- If you have a curated test set, include `test.jsonl` or `test.csv` in your data directory (or pass it via `--test` with individual file flags) to use it instead of the automatically generated test split.
