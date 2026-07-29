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

Trace processing does **not** support contextual (open-book) tasks. The following task types work with traces:

| Task type | Supported |
|-----------|-----------|
| `question-answering` | Yes |
| `classification` | Yes |
| `tool-calling-closed-book` | Yes |
| `multi-turn-tool-calling-closed-book` | Yes |
| `question-answering-closed-book` | Yes |
| `question-answering-open-book` | **No** — trace processing cannot separate context from question automatically. If your production traces contain RAG-style prompts where the retrieved context is embedded in the user message, use `question-answering` instead and keep the full prompt (context + question) in the `user` turn's content. |

## The Four Commands

Traces reach a trainable dataset in four steps. Run them in order; each prints the ID the next one needs.

```bash
# 1. Store the trace files -> <traces-id>
distil traces upload --data <directory>

# 2. Start processing -> <upload-id>
distil upload create-from-traces <traces-id>

# 3. Poll until JOB_SUCCESS (several minutes)
distil upload status <upload-id> --output json | jq -r '.status'

# 4. Fetch the processed data and hand it to the model
distil upload download <upload-id> --data-destination ./processed
distil model upload-data <model-id> --data ./processed
```

Step 4 exists because steps 1-3 work in terms of trace and upload IDs, while `run-teacher-evaluation` and `run-training` read the data uploaded by `upload-data`. `distil upload download` writes exactly the filenames `--data` expects (`train.jsonl`, `test.jsonl`, `unstructured.jsonl`, `config.yaml`, `job_description.json`), so the two commands chain with no editing in between.

`distil model upload-traces` and `distil model reprocess-traces` have been **removed** — their API endpoint no longer exists. Both print the replacement steps and exit non-zero.

### Step 1: Upload the trace files

The recommended approach is to place all required files in a directory:

```bash
distil traces upload --data <directory>
# Output: Prepared traces created. ID: <traces-id>
```

The directory should contain:

| File | Required | Description |
|------|----------|-------------|
| `traces.jsonl` | Yes | Production traces in JSONL format |
| `job_description.json` | Yes | Task objectives and configuration |
| `config.yaml` | Yes | Training and trace processing parameters |
| `test.jsonl` | No | Optional curated test set |

As an alternative to directory mode, specify each file individually:

```bash
distil traces upload \
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
| `--config` | Yes* | Path to config file (`.yaml` or `.yml`). |
| `--test` | No | Path to a curated test data file (`.jsonl` only). |

\* Provide either `--data` or all three individual file flags (`--traces`, `--job-description`, `--config`), but not both.

This command only stores the files — nothing is processed yet, and `distil traces status <traces-id>` reports success for any set that exists. Trace validity surfaces in step 2.

`distil traces list --output json | jq -r '.[0].id'` recovers the ID if you lose it; `distil traces download <traces-id>` pulls the files back.

### Step 2: Process the traces

```bash
distil upload create-from-traces <traces-id>
# Output: Processing started. Upload ID: <upload-id>
```

| Flag | Alias | Description |
|------|-------|-------------|
| `--config` | `-c` | Config file (`.yaml`/`.yml`) merged over the prepared traces' own config on top-level keys. |
| `--job-description` | | Job description file (`.json`) that replaces the prepared traces' own outright. |

Both are optional — omit them to reuse what you uploaded in step 1. Because the merge is per top-level key, a config containing only a `trace_processing` section leaves the rest of the original config untouched.

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

When you upload traces, the platform runs these stages:

1. **Splitting** — deduplicate and split into train + test seed sets; leftover traces become unstructured context.
2. **Filtering** — score each seed trace for relevance and coherence; drop low-scoring traces.
3. **Relabelling** — a committee of teachers rewrites each conversation as a whole; the teacher picks the best.
4. **Validation** — validate rewritten conversations against the task and repair where needed.

Every trace is processed as a multi-turn conversation — a simple single-exchange trace is just a two-turn conversation. For the end-to-end flow diagram and the `trace_processing` parameters gating each stage, see `references/platform-overview.md`. Per-parameter semantics live in the `Trace Processing Configuration` section below and in `references/configuration.md`.

## Test Set Behavior

There are two options for the test set used in evaluation:

**Option 1: Provide your own test set.** Include a `test.jsonl` file in your upload directory (or pass via `--test`). This test set is used as-is for teacher evaluation and training evaluation, without any processing.

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
  num_traces_as_training_base: 200
  num_traces_as_testing_base: 200
  observation_format: openai_messages
```

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `relabel` | `true` | Improve label quality via a committee of teacher models. |
| `num_traces_as_training_base` | `200` | Number of traces to use as the training base. Recommended: set equal to `num_traces_as_testing_base`. |
| `num_traces_as_testing_base` | `200` | Number of traces to use as the testing base. Recommended: set equal to `num_traces_as_training_base` (keeps train/test seeded from comparable trace volumes). |
| `observation_format` | `openai_messages` | Trace format: `openai_messages`, `langfuse`, `openai_messages_with_images`, or `unstructured_with_openai_messages`. |
| `remove_system_prompt_from_traces` | `true` | Strip leading system messages from traces and processed examples (default `true`). |
| `compress_job_description` | `false` | Compress long job descriptions before relevance filtering. |

## Reprocessing Traces

To try different processing parameters, re-run step 2 against the same `<traces-id>` with a new config. The trace files do not need re-uploading:

```bash
distil upload create-from-traces <traces-id> --config ./config.yaml
```

Each run produces a **new** upload ID, so earlier attempts stay intact for comparison — `distil upload list` shows them newest first. There is no `reprocess-traces` command any more; this is it.

## Checking Status

Poll the upload produced in step 2. Copy the canonical loop from `references/tasks/polling-jobs.md`; extract the status with jq rather than grepping text output:

```bash
distil upload status <upload-id> --output json | jq -r '.status'
```

When a run fails, read the processing job's output:

```bash
distil upload logs <upload-id>
```

Once it reports `JOB_SUCCESS`, the base model's scores on the generated test set are available:

```bash
distil upload metrics <upload-id>
```

Then complete step 4 and continue with teacher evaluation and training:

```bash
distil upload download <upload-id> --data-destination ./processed
distil model upload-data <model-id> --data ./processed
distil model run-teacher-evaluation <model-id>
distil model run-training <model-id>
```

`distil upload download` errors while the upload is still processing, so do not run it before the status is terminal.

## Common Gotchas

1. **`validation_max_total_length` applies to traces too** — The default limit of 10,000 characters applies to both uploaded traces/test data and generated synthetic data, not just synthgen output. Over-length content is **silently truncated, not rejected**, so long traces lose content with no warning; surface long traces to the user before uploading. If your production traces contain long inputs (e.g., full documents, injected schemas), increase it in your config:
   ```yaml
   synthgen:
     validation_max_total_length: 30000
   ```

2. **Changing `job_description.json` re-triggers full processing** — There is no way to update it in place. Pass a corrected one to `distil upload create-from-traces <traces-id> --job-description <file>`; that re-runs the whole pipeline including committee relabelling. You do not need to re-upload the trace files, but the processing cost is the same as a first run, so plan the job description carefully.

3. **Markdown fences in relabeled JSON answers** — When `synthgen.output_is_json: true`, committee relabeling models sometimes wrap JSON in ```` ```json ... ``` ```` fences. Teacher evaluation will then fail JSON validation. Workaround: download the relabeled train/test, strip markdown fences from the `assistant` turns' content, validate every `assistant` content parses as JSON, and re-upload as a regular dataset with `distil model upload-data`. Then proceed to teacher evaluation.

4. **(`question-answering` only) `input_description` must be self-contained** — For the `question-answering` task type, the synthgen model that generates training inputs does NOT see `task_description`. It only sees `input_description`. So `input_description` must fully describe the input structure on its own — markers, sections, examples, formatting. If you only put input details in `task_description`, synthgen will produce poor inputs even if teacher evaluation looks fine. This does NOT apply to other task types (`classification`, tool calling, closed-book QA): synthgen does not read `input_description` for them — see `references/job-description-guide.md` for what synthgen reads per task.

## Tips

1. **Provide enough traces** -- Hundreds to thousands of traces is ideal for good results.
2. **Keep relabelling enabled** -- Use `relabel: true` (the default) to improve label quality via a committee of teacher models. The committee approach produces more consistent and accurate labels than any single model.
3. **Iterate by re-processing** -- If the processed data does not look right, re-run `distil upload create-from-traces <traces-id> --config <file>` with different parameters. The trace files stay put; each run yields a new upload ID.
4. **Multi-turn conversations** -- Every trace is processed as a multi-turn conversation and rewritten as a whole; a simple single-exchange trace is just a two-turn conversation. Conversations are preserved automatically for tasks like `multi-turn-tool-calling-closed-book` — no configuration needed.
5. **Cap large trace sets** -- Tune `num_traces_as_training_base` and `num_traces_as_testing_base` to control how many traces feed into seed generation. Unused traces become unstructured context. Set the two to the **same value** so train and test are seeded from comparable trace volumes; diverging them skews the train/test distribution.
6. **Strip large system prompts** -- `remove_system_prompt_from_traces` is `true` by default, stripping leading system messages so large prompts don't dominate the data. Set it to `false` only if you need to keep system prompts.
7. **Compress long job descriptions** -- If your job description is very long, set `compress_job_description: true` to compress it before relevance filtering.
8. **Provide your own test set** -- If you have a curated test set, include `test.jsonl` in your data directory (or pass it via `--test` with individual file flags) to use it instead of the automatically generated test split.
