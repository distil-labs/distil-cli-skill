# Configuration File Reference

The configuration file controls the training pipeline through five main sections: `base`, `tuning`, `evaluation`, `synthgen`, and `trace_processing`.

**For most use cases, the defaults work well.** You only need to specify the task type. Customize other parameters only if you need to fine-tune the training process.

## File Format

- **CLI**: Use YAML format (`config.yaml`)
- **Webapp**: Use JSON format (`config.json`)

## Minimal Configuration

For most cases, this is all you need:

```yaml
base:
  task: classification
```

Or with a custom student model:

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-3B-Instruct
```

## Configuration Structure

```yaml
base:
  # Task and model selection (task is required)
  task: classification

tuning:
  # Fine-tuning parameters
  num_train_epochs: 4

evaluation:
  # Teacher evaluation parameters
  num_few_shot_examples: 1

synthgen:
  # Synthetic data generation parameters
  generation_target: 10000

trace_processing:
  # Trace processing parameters (for training from traces)
  relabel: true
```

---

## Base Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `task` | *required* | Task type (see below) |
| `student_model_name` | `Llama-3.2-1B-Instruct` | Model to fine-tune |
| `teacher_model_name` | `openai.gpt-oss-120b` | Model for synthetic data generation |
| `random_seed` | `123` | Random seed for reproducibility |

### Task Types

| Task | Value |
|------|-------|
| Question Answering | `question-answering` |
| Classification | `classification` |
| Tool Calling | `tool-calling-closed-book` |
| Open Book QA (RAG) | `question-answering-open-book` |
| Closed Book QA | `question-answering-closed-book` |
| Multi-Turn Tool Calling | `multi-turn-tool-calling-closed-book` |

### Student Models

| Model | Value |
|-------|-------|
| Llama 3.2 1B Instruct | `Llama-3.2-1B-Instruct` |
| Llama 3.2 3B Instruct | `Llama-3.2-3B-Instruct` |
| Llama 3.1 8B Instruct | `Llama-3.1-8B-Instruct` |
| SmolLM2 135M | `SmolLM2-135M-Instruct` |
| SmolLM2 1.7B | `SmolLM2-1.7B-Instruct` |
| Gemma 3 270M | `gemma-3-270m-it` |
| Gemma 3 1B | `gemma-3-1b-it` |
| Gemma 3 4B | `gemma-3-4b-it` |
| Qwen3 0.6B | `Qwen3-0.6B` |
| Qwen3 1.7B | `Qwen3-1.7B` |
| Qwen3 4B | `Qwen3-4B-Instruct-2507` |
| Qwen3 8B | `Qwen3-8B` |
| IBM Granite 3.1 8B | `granite-3.1-8b-instruct` |
| IBM Granite 3.3 8B | `granite-3.3-8b-instruct` |

### Teacher Models

| Model | Value |
|-------|-------|
| DeepSeek R1 | `deepseek.r1` |
| DeepSeek V3.1 | `deepseek.v3.1` |
| Qwen3 235B | `Qwen3-235B-A22B-Instruct-2507` |
| Llama 3.3 70B Instruct | `Llama-3.3-70B-Instruct` |
| GPT OSS 120B | `openai.gpt-oss-120b` |

---

## Tuning Configuration

Parameters controlling fine-tuning of the student model.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `learning_rate` | `5e-5` | Initial learning rate for AdamW optimizer |
| `learning_rate_scheduler` | `linear` | Scheduler type: `cosine`, `linear`, `constant` |
| `weight_decay` | `0.0` | Weight decay for regularization |
| `warmup_ratio` | `0.05` | Ratio of steps for learning rate warmup |
| `fp16` | `true` | Use 16-bit mixed precision training |
| `use_lora` | `true` | Use LoRA for efficient fine-tuning |
| `lora_r` | `64` | LoRA attention dimension (rank) |
| `lora_alpha_multiplier` | `1` | LoRA scaling factor |
| `per_device_train_batch_size` | `1` | Training batch size per device |
| `per_device_eval_batch_size` | `1` | Evaluation batch size per device |
| `num_train_epochs` | `4` | Total training epochs |
| `train_eval_split` | `0.2` | Fraction of data for evaluation |

---

## Evaluation Configuration

Parameters for teacher evaluation.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `num_few_shot_examples` | `1` | Few-shot examples for teacher evaluation |
| `llm_as_a_judge_model_name` | `openai.gpt-oss-120b` | Model for LLM-as-a-judge evaluation |

---

## Synthetic Generation Configuration

Parameters controlling synthetic data generation.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `generation_target` | `10000` | Target number of synthetic examples |
| `generation_in_single_call` | `4` | Examples per teacher invocation |
| `teacher_temperature` | `0.7` | Output creativity (0.0-1.0) |
| `validation_similarity_threshold` | `0.95` | Deduplication threshold |
| `match_generated_distribution_to_seed` | `false` | Match class distribution (classification only) |
| `num_distractor_context_blocks` | `0` | Distractor blocks for RAFT training |
| `output_is_json` | `false` | Enforce JSON outputs |

---

## Trace Processing Configuration

Parameters for the trace processing pipeline, which converts production traces into training and test data. These are used when training from traces via `distil model upload-traces`.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `relabel` | `true` | If true, use a committee of models to relabel trace examples. If false, use the original labels from traces. |
| `expand_multiturn_conversations` | `true` | If true, expand multi-turn conversations into multiple training examples. If false, each observation produces a single example using only the last assistant turn. |
| `max_examples` | `null` | Maximum number of training examples to use from traces. |
| `relevance_filtering_batch_size` | `32` | Number of examples scored per batch during relevance filtering. Controls early-stopping granularity against `max_examples`. |
| `num_traces_as_training_base` | `200` | Number of traces to use as the seed for generating training examples. Unused traces beyond this count are used as unstructured data. |
| `num_traces_as_testing_base` | `200` | Number of traces to use as the seed for generating testing examples. Unused traces beyond this count are used as unstructured data. Ignored if a test set is provided. |
| `min_generated_examples` | `20` | Minimum number of examples that trace processing must produce. Raises an error if fewer are generated, to prevent training with too few examples. |
| `max_unstructured` | `10000` | Maximum number of unstructured data examples to include. |
| `observation_format` | `openai_messages` | Format of trace observations in `traces.jsonl`. Options: `langfuse`, `openai_messages`, `unstructured_with_openai_messages`. |
| `remove_system_prompt_from_traces` | `false` | If true, remove the system prompt from traces before converting them to unstructured data. Useful when the system prompt is very large. |
| `compress_job_description` | `false` | If true, compress the job description using the teacher model before relevance filtering. Useful when the task description is very long. |
| `teacher_model_name` | `openai.gpt-oss-120b` | Teacher model used for relevance filtering and picking the best relabelled answer. |
| `relabelling_committee_models` | See below | Models that produce candidate relabels. Each model generates an output for every example; the teacher then picks the best. Only used when `relabel` is true. |

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
  teacher_temperature: 0.6
  validation_similarity_threshold: 0.9

trace_processing:
  relabel: true
  num_traces_as_training_base: 5000
  num_traces_as_testing_base: 100
  max_examples: 10000
```

---

## Model-Specific Notes

### DeepSeek R1
When using `deepseek.r1` as the teacher model, use temperature between **0.5 and 0.7**.

### Tool Calling
Only **Qwen3 and Llama 3-family** student models are supported for the `tool-calling-closed-book` task.
