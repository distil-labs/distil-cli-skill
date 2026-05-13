# Configuration Reference

The `config.yaml` file controls the training pipeline through five sections: `base`, `tuning`, `evaluation`, `synthgen`, and `trace_processing`.

For most use cases the defaults work well. You only need to specify the task type.

## Parameter Tiers

When preparing `config.yaml` for a user, only include parameters from the tier that matches their situation. Don't expose expert-level params unless the user asks.

- **Always set:** `task` (required), `student_model_name`, `teacher_model_name`
- **Optional on the first run, often revisited when iterating:** `basic_mutators_to_use`, `mutation_topics`, `generation_target`, `num_generations_per_llm_call`. You don't *need* to set these on the first training run — the defaults are reasonable. But if the user's task description names specific patterns/scenarios/length characteristics to cover (e.g., "short/medium/long conversations", "billing vs. cancellation vs. tech support"), it's worth translating them into `mutation_topics` / `basic_mutators_to_use` upfront rather than waiting for iteration.
- **Expert only (leave defaults):** everything else (LoRA rank, batch sizes, warmup ratio, learning rate, RLVR, etc.)

## File Format

- **CLI**: YAML format (`config.yaml`)
- **Webapp**: JSON format (`config.json`)

## Minimal Configuration

```yaml
base:
  task: classification
```

With a custom student model:

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-3B-Instruct
```

With a custom teacher model:

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-3B-Instruct
  teacher_model_name: openai.gpt-oss-120b
```

## Configuration Structure

```yaml
base:
  task: classification

tuning:
  num_train_epochs: 4

evaluation:
  num_few_shot_examples: 1

synthgen:
  generation_target: 10000

trace_processing:
  relabel: true
```

---

## 1. Base Configuration

General parameters for task and model selection.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `task` | `string` | *required* | Type of NLP task to solve. See supported task types below. |
| `student_model_name` | `string` | `Llama-3.2-1B-Instruct` | Base model to fine-tune for the use case. |
| `teacher_model_name` | `string` | `openai.gpt-oss-120b` | Teacher model used for synthetic data generation and knowledge distillation. |
| `random_seed` | `integer \| null` | `123` | Random seed for reproducible sampling across the pipeline. |

### Supported Task Types

| Task | Value |
|------|-------|
| Question Answering | `question-answering` |
| Classification | `classification` |
| Tool Calling | `tool-calling-closed-book` |
| Multi-Turn Tool Calling | `multi-turn-tool-calling-closed-book` |
| Open Book QA (RAG) | `question-answering-open-book` |
| Closed Book QA | `question-answering-closed-book` |

### Supported Models

Student and teacher model lists, task compatibility, and default recommendations live in `references/model-catalog.md`. Read that file before recommending a specific model or judging whether a config value is valid.

---

## 2. Tuning Configuration

Parameters controlling fine-tuning of the student model.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `learning_rate` | `float` | `5e-5` | Initial learning rate for the AdamW optimizer. |
| `learning_rate_scheduler` | `string` | `linear` | Scheduler type. Options: `cosine`, `linear`, `constant`. |
| `weight_decay` | `float` | `0.0` | Weight decay applied to all layers except bias and LayerNorm weights in the AdamW optimizer. |
| `warmup_ratio` | `float` | `0.05` | Ratio of total training steps used for linear warmup from 0 to `learning_rate`. |
| `bf16` | `boolean` | `true` | Use bf16 16-bit (mixed) precision training instead of 32-bit training. |
| `use_lora` | `boolean` | `true` | Use LoRA for student training. |
| `lora_r` | `integer` | `64` | LoRA attention dimension (rank). Only used if `use_lora` is true. |
| `lora_alpha_multiplier` | `integer` | `1` | LoRA scaling factor. Alpha is computed as `lora_r * lora_alpha_multiplier`. Only used if `use_lora` is true. |
| `per_device_train_batch_size` | `integer` | `1` | Batch size per GPU/device for training. |
| `per_device_eval_batch_size` | `integer` | `1` | Batch size per GPU/device for evaluation. |
| `num_train_epochs` | `integer` | `4` | Total number of training epochs. |
| `train_eval_split` | `float` | `0.2` | Fraction of training data used for evaluation. Must be between 0 and 1 (exclusive). |
| `gradient_accumulation_steps` | `integer` | `1` | Number of update steps to accumulate gradients before performing a backward/update pass. Effectively multiplies batch size by this factor without increasing memory usage. |
| `num_few_shot_examples_student` | `integer` | `0` | Number of few-shot examples for student evaluation and tuning. If above 0, at least one example per class is used for classification tasks. |

### RLVR (Reinforcement Learning with Verifiable Rewards)

RLVR is an optional reinforcement learning stage that runs after SFT fine-tuning. It uses reward signals from an LLM-as-a-judge to further improve model performance. Set `rlvr_dataset_size` to a value greater than 0 to enable it.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `rlvr_dataset_size` | `float` | `0.0` | Proportion of the dataset to use for the RLVR split. Must be between 0.0 and 1.0. `0.0` means RLVR is disabled. |
| `rlvr_llm_as_a_judge_model_name` | `string` | `openai.gpt-oss-20b` | Model used for the LLM-as-a-judge reward signals in RLVR. |
| `rlvr_per_device_batch_size` | `integer` | `6` | Batch size per GPU/device for RLVR training and evaluation. Must be a multiple of `rlvr_num_generations`. |
| `rlvr_num_generations` | `integer` | `6` | Number of generations per prompt during RLVR training. |
| `rlvr_num_train_epochs` | `integer` | `1` | Number of training epochs for RLVR fine-tuning. |

---

## 3. Evaluation Configuration

Parameters used in teacher evaluation.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `num_few_shot_examples` | `integer` | `1` | Number of few-shot examples for teacher evaluation. If above 0, at least one example per class is used for classification tasks. |
| `llm_as_a_judge_model_name` | `string` | `openai.gpt-oss-120b` | Model used to power the LLM-as-a-judge evaluation. |
| `expand_tool_calling_turns` | `boolean` | `true` | If true, each line in multi-turn tool calling test files is expanded into multiple evaluation lines, each ending at a tool call. |
| `batch_size` | `integer` | `4` | *(Deprecated)* Batch size for model evaluation. |

---

## 4. Synthgen Configuration

Parameters for fine-grained control over synthetic data generation.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `generation_target` | `integer` | `10000` | Target number of synthetic examples to generate. For Closed-Book QA, calculated as `len(unstructured_data) * generation_per_unstructured_context`. |
| `generation_in_single_call` | `integer` | `4` | Number of examples to generate per teacher/LLM invocation. |
| `generation_iteration_size` | `integer` | `128` | Batch size for the generate-validate cycle. |
| `generation_per_unstructured_context` | `integer \| null` | `null` | Examples to generate per unstructured context. Only used with `question-answering-closed-book` task. Overwrites `generation_target` when set. |
| `num_positive_exemplars_per_generation` | `integer` | `2` | Number of in-context examples for the class/task being generated. |
| `num_negative_exemplars_per_generation` | `integer` | `2` | Number of in-context examples for classes not being generated. Only used for classification tasks. |
| `num_unlabelled_exemplars_per_generation` | `integer` | `2` | Number of unlabelled examples provided during each teacher invocation. |
| `validation_max_total_length` | `integer` | `10000` | Maximum total length (input + output) of examples in characters. Applied to both uploaded traces/test data and generated synthetic data. Increase this if your production inputs are long (e.g., full documents, injected schemas). |
| `validation_similarity_threshold` | `float` | `0.95` | Similarity threshold for deduplication. Generated data with similarity above this threshold to seed data are removed. |
| `validation_max_answer_length` | `integer` | `8192` | *(Deprecated)* Use `validation_max_total_length` instead. |
| `teacher_temperature` | `float` | `0.7` | Temperature for teacher output. Controls the balance between predictability and creativity. Must be between 0.0 and 1.0. |
| `teacher_max_tokens` | `integer \| null` | `null` | Maximum tokens in the generated response. |
| `match_generated_distribution_to_seed` | `boolean` | `false` | Match generated data class distribution to seed data. Only used for classification tasks. |
| `num_distractor_context_blocks` | `integer` | `0` | Number of distractor context blocks per example. Setting above zero enables [RAFT training](https://arxiv.org/pdf/2403.10131). |
| `output_is_json` | `boolean` | `false` | Only generate synthetic data with valid JSON outputs. Only relevant for QA tasks. |
| `basic_mutators_to_use` | `list[string]` | `["complexity"]` | List of basic mutators for data generation. Supported options: `complexity`, `length`, `specificity`. |
| `mutation_topics` | `list[list[string]] \| list[string]` | `[]` | Topics to sample from to guide the generation process. |
| `parallel_llm_calls` | `boolean` | `false` | If true, call the LLM in parallel during data generation and evaluation. |

---

## 5. Trace Processing Configuration

Parameters for the trace processing pipeline, which converts production traces into training and testing data. These are used when training from traces via `distil model upload-traces`.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `relabel` | `boolean` | `true` | If true, use a committee of models to relabel trace examples. If false, use the original labels from traces. |
| `convert_to_single_turn` | `boolean` | `true` (but see note) | If true, split multi-turn conversations into independent single-turn examples, relabeling each turn separately. If false, keep conversations intact as multi-turn examples and rewrite them as a whole. **Task-specific default:** keep `true` for single-turn tasks (QA, classification, single-turn tool calling). Set to `false` when training a **multi-turn** task (`multi-turn-tool-calling-closed-book`) — otherwise you'd split the conversations you need to preserve as seed data into isolated single-turn examples. |
| `relevance_filtering_batch_size` | `integer` | `32` | Number of examples scored per batch during relevance filtering. |
| `num_traces_as_training_base` | `integer` | `200` | Number of traces to use as the seed for generating training examples. Unused traces beyond this count are used as unstructured data. **Recommended:** set equal to `num_traces_as_testing_base`. Keep `min_generated_examples` low enough that the examples derived from this count aren't rejected by the floor. |
| `num_traces_as_testing_base` | `integer` | `200` | Number of traces to use as the seed for generating testing examples. Unused traces beyond this count are used as unstructured data. Ignored if a test set is provided. **Recommended:** set equal to `num_traces_as_training_base` — keeping the two in sync avoids train/test distribution skew where one side is seeded from far more traces than the other. |
| `min_generated_examples` | `integer` | `20` | Minimum number of examples that trace processing must produce. Raises an error if fewer are generated, to prevent training with too few examples. **Keep this low enough** that the examples produced from `num_traces_as_training_base` / `num_traces_as_testing_base` clear the floor — filtering and relabelling typically drop a significant fraction of traces, so setting `min_generated_examples` close to the trace count will cause spurious errors. |
| `max_unstructured` | `integer` | `10000` | Maximum number of unstructured data examples to include. |
| `observation_format` | `string` | `openai_messages` | Format of trace observations in `traces.jsonl`. Options: `langfuse` (Langfuse observation objects with id, input, output), `openai_messages` (objects with a `messages` array of chat completion messages), `unstructured_with_openai_messages` (unstructured data with OpenAI messages). |
| `remove_system_prompt_from_traces` | `boolean` | `false` | If true, remove the system prompt from traces before converting them to unstructured data. Useful when the system prompt is very large and would dominate the unstructured context. |
| `compress_job_description` | `boolean` | `false` | If true, compress the job description using the teacher model before relevance filtering. Useful when the task description is very long and would overwhelm the filtering LLM. |
| `teacher_model_name` | `string` | `openai.gpt-oss-120b` | Teacher model used for relevance filtering and picking the best relabelled answer from the committee. |
| `relabelling_committee_models` | `list[string]` | See below | Models that produce candidate relabels. Each model generates an output for every example; the teacher then picks the best. Only used when `relabel` is true. |

**Default relabelling committee:** `["zai.glm-5", "Qwen3-235B-A22B-Instruct-2507", "openai.gpt-oss-120b-thinking", "deepseek.v3.2"]`

---

## Full Configuration Example

```yaml
base:
  task: question-answering-open-book
  student_model_name: Qwen3-1.7B
  teacher_model_name: openai.gpt-oss-120b
  random_seed: 42

tuning:
  learning_rate: 1e-4
  learning_rate_scheduler: cosine
  use_lora: true
  lora_r: 32
  num_train_epochs: 3
  train_eval_split: 0.15

evaluation:
  num_few_shot_examples: 2

synthgen:
  generation_target: 5000
  generation_in_single_call: 8
  teacher_temperature: 0.6
  validation_similarity_threshold: 0.9

trace_processing:
  relabel: true
  num_traces_as_training_base: 5000
  num_traces_as_testing_base: 5000
```

---

## Notes

### Common configurations

**Classification with balanced generation:**
```yaml
base:
  task: classification
synthgen:
  match_generated_distribution_to_seed: true
```

**Open Book QA with RAFT (distractor contexts):**
```yaml
base:
  task: question-answering-open-book
synthgen:
  num_distractor_context_blocks: 3
```

**Training from traces with relabelling disabled:**
```yaml
base:
  task: question-answering
trace_processing:
  relabel: false
```

**Enabling RLVR after SFT:**
```yaml
tuning:
  rlvr_dataset_size: 0.3
  rlvr_num_train_epochs: 1
```

### Model-specific notes

Model compatibility constraints (tool calling student restrictions, multi-turn tool calling teacher restrictions, etc.) live in `references/model-catalog.md`. Notes here only cover config-parameter behavior.

- **GPT OSS 120B Thinking**: The `openai.gpt-oss-120b-thinking` model uses a `medium` reasoning effort setting by default for enhanced chain-of-thought capabilities.
- **Trace processing teacher model**: The `trace_processing.teacher_model_name` is independent from `base.teacher_model_name`. The trace processing teacher handles relevance filtering and relabel arbitration, while the base teacher handles synthetic data generation.
- **`generation_per_unstructured_context`**: Only applies to `question-answering-closed-book`. When set, it overwrites `generation_target`. See `references/tasks/prepare-data/closed-book-qa.md` for how this interacts with unstructured data.
- **`rlvr_per_device_batch_size`**: Must be a multiple of `rlvr_num_generations`.
- **`train_eval_split`**: Must be strictly between 0 and 1 (exclusive). A value of 0.2 means 20% of training data is held out for evaluation during training.
