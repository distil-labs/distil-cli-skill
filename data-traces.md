# Training from Traces Data Preparation

**Note:** Training from traces is an experimental feature. The API and behavior may change in future releases.

Use training from traces when you have production logs of real interactions with an LLM and want to automatically prepare training and test data from them, instead of manually creating structured datasets. Traces are uploaded, filtered for relevance, relabelled by a committee of teacher models, and split into training and test sets — all automatically.

**Example use cases:**
- Bootstrapping a fine-tuned model from existing production chat logs
- Improving a deployed model by training on its own successful interactions
- Converting multi-turn conversation logs into individual training examples
- Rapidly creating training data without manual labelling effort

## How It Works

Training from traces is a two-step process:

1. **Upload** — Traces are uploaded and stored as a `PreparedTraces` resource
2. **Process** — Traces are filtered, relabelled, split, and expanded into training and test data (an `Upload`)

The CLI's `upload-traces` command performs both steps in one go. To re-run only the processing step with different parameters (e.g. adjusting filtering or relabelling), use `reprocess-traces`.

The trace processing pipeline will automatically:
- Filter traces for relevance to your task
- Relabel examples using a committee of teacher models
- Split traces into training and test sets
- Convert multi-turn conversations to single-turn training examples, or preserve them as full conversations with committee-based rewriting

## Required Files

### 1. traces.jsonl

JSONL file containing production traces. Each line is a JSON object. There are three supported formats, controlled by the `observation_format` parameter in your config:

**`openai_messages`** (default) — Each line contains a `messages` array of OpenAI chat completion messages:

```json
{"messages": [{"role": "system", "content": "You are a helpful assistant."}, {"role": "user", "content": "What is the capital of France?"}, {"role": "assistant", "content": "The capital of France is Paris."}]}
{"messages": [{"role": "user", "content": "Translate 'hello' to Spanish."}, {"role": "assistant", "content": "Hola"}]}
```

**`langfuse`** — Langfuse observation objects with `id`, `input`, and `output` fields:

```json
{"id": "obs-1", "input": [{"role": "user", "content": "What is 2+2?"}], "output": {"role": "assistant", "content": "4"}}
```

**`unstructured_with_openai_messages`** — Unstructured data with OpenAI messages format.

### 2. job_description.json

Same format as the structured data approach — describes the task the model should perform:

```json
{
  "task_description": "Answer customer questions about our product based on the conversation history."
}
```

**Fields:**
- `task_description`: Explains what the model should do and the expected output format

### 3. config.yaml (or config.json)

Must include a `base` section with the task type. Optionally include a `trace_processing` section to control how traces are processed. For all available trace processing parameters, see `config.md`.

**Minimal example:**

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-1B-Instruct

trace_processing:
  relabel: true
```

**Full example with common trace processing parameters:**

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-3B-Instruct
  teacher_model_name: openai.gpt-oss-120b

trace_processing:
  relabel: true
  expand_multiturn_conversations: true
  num_traces_as_training_base: 200
  num_traces_as_testing_base: 200
  max_examples: null
  observation_format: openai_messages
```

**Trace processing parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `relabel` | `true` | Improve label quality via a committee of teacher models |
| `expand_multiturn_conversations` | `true` | Split multi-turn conversations into individual training examples |
| `num_traces_as_training_base` | `200` | Number of traces to use as the training base |
| `num_traces_as_testing_base` | `200` | Number of traces to use as the testing base |
| `max_examples` | `null` | Cap the number of training examples (null = no limit) |
| `observation_format` | `openai_messages` | Trace format: `openai_messages`, `langfuse`, or `unstructured_with_openai_messages` |
| `remove_system_prompt_from_traces` | `false` | Remove large system prompts from traces |
| `compress_job_description` | `false` | Compress long job descriptions before relevance filtering |

### 4. test.jsonl or test.csv (Optional)

An optional held-out test set. If provided, it will be used as the test data instead of the platform automatically splitting traces into training and test sets. The format should match the task type (same as structured data uploads — see the relevant data preparation guide for your task).

When using `--data` (directory upload), place the file in the directory as `test.jsonl` or `test.csv`. When using individual file flags, pass it with `--test`.

## Upload and Train

**Upload traces** — provide either a directory or individual files:

```bash
# Option 1: Directory (expects traces.jsonl, job_description.json, config.yaml/config.json, and optionally test.jsonl/test.csv)
distil model upload-traces <model-id> --data ./my-traces

# Option 2: Individual files
distil model upload-traces <model-id> --traces traces.jsonl --job-description job_description.json --config config.yaml

# Option 3: Individual files with optional test data
distil model upload-traces <model-id> --traces traces.jsonl --job-description job_description.json --config config.yaml --test test.jsonl
```

You must provide either `--data` or the three required individual file flags (`--traces`, `--job-description`, `--config`), but not both. The `--test` flag is optional with individual file flags; when using `--data`, place `test.jsonl` or `test.csv` in the directory instead.

**Reprocess traces** — after the initial upload, try different processing parameters without re-uploading:

```bash
# Using a trace processing config (only trace_processing parameters)
distil model reprocess-traces <model-id> --trace-processing-config new-config.yaml

# Or using a full config file (only the trace_processing section is used)
distil model reprocess-traces <model-id> --config new-config.yaml
```

**Check upload status:**

```bash
distil model upload-status <model-id>
```

**Continue with the normal workflow** once processing is complete:

```bash
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

## Tips

1. **Provide enough traces** — Hundreds to thousands of traces is ideal for good results
2. **Keep relabelling enabled** — Use `relabel: true` (the default) to improve label quality via a committee of teacher models
3. **Iterate with reprocess** — If you're not happy with the processed data, use `reprocess-traces` to try different parameters without re-uploading the traces
4. **Multi-turn conversations** — Set `expand_multiturn_conversations: true` (the default) to convert multi-turn conversations into individual training examples, or set it to `false` to preserve them as full conversations with committee-based rewriting
5. **Cap large trace sets** — Use `max_examples` to limit the number of training examples if you have a very large trace set
6. **Strip large system prompts** — If your traces have very large system prompts, set `remove_system_prompt_from_traces: true` to prevent them from dominating the data
7. **Compress long job descriptions** — If your job description is very long, set `compress_job_description: true` to compress it before relevance filtering
8. **Provide your own test set** — If you have a curated test set, include `test.jsonl` or `test.csv` in your data directory, or pass it via `--test` when using individual file flags, to use it instead of the automatically generated test split