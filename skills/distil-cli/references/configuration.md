# Configuration

`config.yaml` has five sections: `base`, `tuning`, `evaluation`, `synthgen`,
`trace_processing`. Only `base.task` is required; every other parameter has a default. Always
set `task`, `student_model_name` and `teacher_model_name`.

## base

| Parameter | Default | Notes |
|---|---|---|
| `task` | required | The task the student solves: `question-answering`, `classification`, `chat-completion` or `chat-completion-agentic`. See `task-types.md` |
| `visual_task` | `false` | The task inputs carry images. Question answering only; every model must be vision-capable. Image traces use the `openai_messages_with_images` observation format |
| `enable_thinking` | `false` | The student is trained, evaluated and served with its chat template's thinking mode on. Synthetic data generation then writes the reasoning: the teacher produces it with each assistant message it generates, and a final backfill pass fills in what is still missing. Reasoning students only. See `reasoning-models.md` |
| `variable_system_prompt` | `false` | `false`: every example is trained, evaluated and served with the `task_description` as its system message, which replaces a system message found in a row or trace. `true`: every row and relabeled trace keeps its own system message (a row without one gets the `task_description`), and the served model reads the system message each request sends. See `variable-system-prompts.md` |
| `student_model_name` | `Llama-3.2-1B-Instruct` | The base model that is fine-tuned into your student. See `model-catalog.md` |
| `teacher_model_name` | `openai.gpt-oss-120b` | The model that generates the synthetic data and that the student learns from. See `model-catalog.md` |
| `random_seed` | `123` | Seed for random sampling across the pipeline, mutators included |
| `llm_num_parallel_requests` | `4` | Maximum number of LLM requests sent in parallel, across the teacher, synthetic generation and the judge. `1` turns parallelism off |
| `should_expand_dataset` | `"auto"` | Whether a conversation becomes one example per assistant turn: `true`, `false` or `auto`. § Conversation expansion |

## tuning

| Parameter | Default | Notes |
|---|---|---|
| `learning_rate` | `5e-5` | Initial learning rate of the AdamW optimizer |
| `learning_rate_scheduler` | `linear` | `cosine`, `linear` or `constant`. See the Hugging Face [`SchedulerType`](https://huggingface.co/docs/transformers/en/main_classes/optimizer_schedules) for all values |
| `weight_decay` | `0.0` | AdamW weight decay, applied to all layers except the bias and LayerNorm weights |
| `warmup_ratio` | `0.05` | Fraction of the training steps used for a linear warmup from 0 to `learning_rate` |
| `bf16` | `true` | bf16 mixed-precision training instead of 32-bit training |
| `lora_r` | `64` | Rank of the LoRA adapter; students are always trained with LoRA. One of the ranks vLLM serves LoRA adapters at: 8, 16, 32, 64, 128, 256, 320 or 512 |
| `lora_alpha_multiplier` | `1` | LoRA alpha = `lora_r * lora_alpha_multiplier` |
| `per_device_train_batch_size` | `1` | Batch size per device for fine-tuning |
| `per_device_eval_batch_size` | `1` | Batch size per device of the trainer's evaluation on the validation set. Used only with `enable_trainer_internal_eval: true`, never by the post-training evaluation |
| `num_train_epochs` | `4` | Number of fine-tuning epochs |
| `gradient_accumulation_steps` | `1` | Steps whose gradients accumulate before one optimizer update. Multiplies the effective batch size without using more memory |
| `enable_trainer_internal_eval` | `false` | Fine-tuning and RLVR evaluate on the validation set, which is the test split, before training and at the end of every epoch. Final metrics come from the post-training evaluation either way; `false` trains faster and with less memory |
| `logging_steps` | `0.01` | How often fine-tuning and RLVR log their metrics, as a fraction of all optimizer steps (`0.1` logs every 10%). Above 0, at most 1 |
| `save_steps` | `0.25` | How often fine-tuning and RLVR save a checkpoint, as a fraction of all optimizer steps (`0.1` saves every 10%). Above 0, at most 1 |
| `memory_optimized_training` | `0` | `0`, `1` or `2`; raise it only when training runs out of GPU memory. `1`: gradient checkpointing, which recomputes activations in the backward pass instead of storing them, and is slower (RLVR also puts vLLM to sleep between generations). `2`: `1` plus activation offloading to CPU RAM, much slower. `true` reads as `2` and `false` as `0`. `../stages/model-training.md` |
| `use_qlora` | `false` | Loads the base model in 4-bit NF4 (QLoRA) for fine-tuning and attaches the LoRA adapter in higher precision. The base model uses about a third of the GPU memory, and training is slightly slower. Requires bitsandbytes (Linux only) |
| `max_completion_length` | `2048` | Tokens the student may generate: caps generation in student evaluation and the rollouts of RLVR. Teachers use `synthgen.teacher_max_tokens` instead. Raise it when training warns it is below the longest training completion |
| `rlvr_dataset_size` | `0.0` | Fraction of the dataset in the RLVR split, from 0 to 1. RLVR, an optional reinforcement learning stage after fine-tuning, runs when this is above 0 |
| `rlvr_llm_as_a_judge_model_name` | inherits `base.teacher_model_name` | The judge that scores RLVR rollouts |
| `rlvr_reward_metric` | `llm-judge-reference-free` | How the judge scores a rollout. `llm-judge-reference-free`: against the problem alone. `llm-judge-reference`: the judge also sees the reference answer. `llm-judge-reference-without-context`: the judge sees the reference but not the problem, which is much cheaper on long problems and enough when the reference fixes the answer exactly |
| `rlvr_per_device_batch_size` | `6` | Completions per forward and backward pass in RLVR training and evaluation. `rlvr_per_device_batch_size * rlvr_gradient_accumulation_steps` must be a multiple of `rlvr_num_generations`, so the gradients from one prompt stay in one optimizer step |
| `rlvr_num_generations` | `6` | Completions sampled per prompt in RLVR training |
| `rlvr_gradient_accumulation_steps` | `1` | Micro-batches per RLVR optimizer step, separate from `gradient_accumulation_steps` |
| `rlvr_num_train_epochs` | `1` | Number of RLVR epochs |
| `rlvr_learning_rate` | `5e-5` | Initial AdamW learning rate of RLVR |
| `rlvr_weight_decay` | `0.0` | AdamW weight decay of RLVR, applied to all layers except the bias and LayerNorm weights |
| `rlvr_learning_rate_scheduler` | `constant` | Learning rate scheduler of RLVR; `learning_rate_scheduler` does not apply to it |
| `rlvr_scale_rewards` | `batch` | How GRPO turns the centred rewards into advantages. `group` divides by the standard deviation of each prompt's group, which turns judge noise in an otherwise uniform group into a full-size advantage. `batch` divides by the standard deviation of the whole batch, so the advantage stays proportional to the reward gap. `none` leaves the centred rewards unscaled |
| `rlvr_vllm_importance_sampling_mode` | `token_truncate` | How GRPO corrects for vLLM and the trainer giving slightly different log-probabilities to the same rollout. `token_*` uses one ratio per token, `sequence_*` one ratio per completion (the product over its tokens, which drifts far from 1 on long completions). `*_truncate` clips the ratio at `rlvr_vllm_importance_sampling_clip_max`, `*_mask` zeroes the loss above it |
| `rlvr_vllm_importance_sampling_clip_max` | `2.0` | Upper bound of the vLLM importance-sampling ratio; `null` leaves it unbounded |
| `rlvr_steps_per_generation` | `null` | Micro-batches of `rlvr_per_device_batch_size` rollouts generated in one go. `null` means `rlvr_gradient_accumulation_steps`: each generation feeds exactly one optimizer step, so training is on-policy and `rlvr_epsilon` and `rlvr_epsilon_high` never bind. A multiple k of `rlvr_gradient_accumulation_steps` spreads one generation over k optimizer steps (GSPO): the steps after the first train on rollouts from a slightly older policy, and the clip bounds how far each of them moves it |
| `rlvr_importance_sampling_level` | `sequence` | Level of the ratio between the current policy and the policy that generated the rollouts, which `rlvr_epsilon` and `rlvr_epsilon_high` clip. `token`: one ratio per token (GRPO). `sequence`: one length-normalised ratio per completion (GSPO). Matters only when `rlvr_steps_per_generation` makes training off-policy |
| `rlvr_epsilon` | `3e-4` | Lower clip of that ratio: an update may lower a rollout's ratio to at most 1 - `rlvr_epsilon`. The default suits sequence-level ratios (GSPO); token-level GRPO usually uses 0.2 |
| `rlvr_epsilon_high` | `4e-4` | Upper clip of that ratio: an update may raise a rollout's ratio to at most 1 + `rlvr_epsilon_high`. The default suits sequence-level ratios (GSPO); token-level GRPO usually uses 0.28 |
| `rlvr_overlong_punishment_cache` | `null` | Length penalty for RLVR rollouts (DAPO soft overlong punishment): the number of tokens before `max_completion_length` over which the penalty ramps up. A rollout of up to `max_completion_length - rlvr_overlong_punishment_cache` tokens has no penalty; above that the penalty falls linearly from 0 to -1 at `max_completion_length`. For example, `max_completion_length` 6144 and `rlvr_overlong_punishment_cache` 2048 give no penalty up to 4096 tokens, -0.5 at 5120 and -1 at 6144. The penalty is added to the judge score and also applies when the judge could not score the rollout. Must be below `max_completion_length`; `null` turns it off |
| `rlvr_vllm_gpu_memory_utilization` | `0.3` | Fraction of the GPU the vLLM engine that generates the rollouts may use for its copy of the student and its KV cache. The training copy of the student uses the rest, so a student whose weights alone exceed this fraction cannot start its vLLM engine |

## evaluation

| Parameter | Default | Notes |
|---|---|---|
| `llm_as_a_judge_model_name` | inherits `base.teacher_model_name` | The model behind the LLM-as-a-judge. Scores from different judges can't be compared. Set it once and keep it fixed across all runs. `model-catalog.md` § Defaults |

## synthgen

| Parameter | Default | Notes |
|---|---|---|
| `train_generation_target` | `10000` | Synthetic rows one `generate-synthetic-data-train` run adds to the train split; `0` generates nothing. A target, not an exact count: generation runs in `generation_iteration_size` batches until the target is met, so the result can exceed it by up to one batch |
| `test_generation_target` | `0` | Synthetic rows one `generate-synthetic-data-test` run adds to the test split; `0` generates nothing. Same rounding |
| `use_traces_as_context` | `true` | The traces in the dataset serve as context for generation and the used ones leave the dataset: `T + min(T, 1000)` for a target of T, or all that are left. Below `min(T / 4, 10)` traces the job logs a warning and generates without context; with `false` it generates without context. Either way it leaves the traces alone. `platform.md` § The expand operations |
| `generation_in_single_call` | `4` | Examples generated per teacher call |
| `generation_iteration_size` | `128` | Examples per generation batch: generation and validation run in batches of this size |
| `num_positive_exemplars_per_generation` | `1` | In-context examples per generation call of the class or task being generated |
| `num_negative_exemplars_per_generation` | `1` | In-context examples of the classes not being generated. Classification only |
| `num_unlabelled_exemplars_per_generation` | `1` | Unlabeled examples (context traces) shown per generation call, when the run has context |
| `validation_max_total_length` | `30000` | Maximum total length of an example, input plus output, in characters. Applies to uploaded data too |
| `validation_similarity_threshold` | `0.95` | Generated rows whose similarity to the rows already in the dataset is above this threshold are removed. Lower it if generation produces near-duplicates |
| `validation_instructions` | `null` | Instructions for an LLM validator of synthetic examples. The evaluation judge (by default the teacher) checks each complete example, system prompt included, against these instructions and the job description, after the structural and diversity checks. Rejected examples and invalid judge replies are dropped. `null` turns it off and makes no validation LLM calls. § LLM validation |
| `teacher_temperature` | `0.7` | Sampling temperature of the teacher, from 0 to 1: higher gives more varied output. § Cross-field validation for reasoning teachers |
| `teacher_max_tokens` | `32000` | Maximum tokens in one teacher response. Kept well below typical context limits, so the reserved output does not crowd out large (for example multi-image) prompts |
| `match_generated_distribution_to_seed` | `false` | Generated data keeps the class distribution of the seed rows. Classification only |
| `output_is_json` | `false` | Question answering only: only synthetic rows whose output is valid JSON are generated. Answers in uploaded data must also be valid JSON |
| `mutators` | `[]` | Mutators applied during synthetic generation, one per dimension to vary. `mutators.md` |
| `mutator_update_frequency` | `5` | Generation batches between two runs of the topic classifier, for every adaptive mutator. The batches in between are not classified; their counts are estimated from the classified ones. Classifying costs teacher calls, so a higher number is cheaper and coarser. `mutators.md` § Adaptive mutators |
| `max_tool_calls_per_turn` | `null` (1 with tools declared, 0 without) | Chat completion only: the most tool calls one assistant message may contain. `unlimited` allows any number of parallel calls; `0` disallows tool calls. A job description without tools needs `0` or `null`. `data-preparation/chat-completion.md` § Tool calls per turn |
| `clean_training_targets` | `false` | A final teacher pass that minimally repairs corrupted or truncated final assistant messages (the training targets). Expansion before it follows `base.should_expand_dataset`; with `false` only the supplied final-turn targets are cleaned |

### LLM validation

Set `synthgen.validation_instructions` to a non-empty prompt to enable these checks.
The criteria do not have to appear in the job description. The judge receives the
instructions, the job description and the complete example, including its system
prompt, tool calls and images.

Validation uses `evaluation.llm_as_a_judge_model_name`, falling back to
`base.teacher_model_name`. It runs only during synthetic generation, after structural
and diversity checks, with one judge call per surviving example and the configured
request parallelism. Rejected examples and malformed judgments are dropped without
rewriting. Omit the field or set it to `null` to disable it; an empty string is invalid.
This check does not run during relabeling, target cleanup or dataset loading.

## trace_processing

Read by `relabel-traces-train` and `relabel-traces-test`
(`../stages/relabel-traces.md`); `observation_format` is also read
when a dataset is created from traces.

| Parameter | Default | Notes |
|---|---|---|
| `relabel` | `true` | The teacher (or committee) relabels the traces. `false` keeps the original trace labels |
| `relevance_filtering` | `false` | An LLM scores each trace and drops those below `min_relevance_score` or `min_coherence_score`. `false` skips the step and keeps every trace |
| `relevance_filtering_batch_size` | `32` | Traces scored per batch during relevance filtering |
| `min_relevance_score` | `4` | Minimum relevance score, 1-5, for a trace to pass relevance filtering |
| `min_coherence_score` | `3` | Minimum coherence score, 1-5, for a trace to pass. A lower value lets more corrupted traces through for committee repair |
| `num_train_relabelled` | `200` | Traces one `relabel-traces-train` run turns into train rows. The dataset must hold at least this many distinct traces or the job refuses to run; fewer rows can come out, because relabeling drops some traces. The used traces leave the dataset |
| `num_test_relabelled` | `200` | The same for `relabel-traces-test` and the test split |
| `observation_format` | `openai_messages` | Format of the traces in `traces.jsonl`. `openai_messages`: objects with a `messages` array of chat completion messages. `langfuse`: Langfuse observation objects (`id`, `input`, `output`). See `data-preparation/traces.md` |
| `remove_system_prompt_from_traces` | `true` | Deprecated, no effect. `base.variable_system_prompt` decides whether traces keep their system message |
| `compress_job_description` | `false` | The teacher compresses the job description before relevance filtering. For a `task_description` so long it would overwhelm the filtering LLM |
| `teacher_model_name` | inherits `base.teacher_model_name` | Filters the traces and picks the best committee answer |
| `relabelling_committee_models` | `[]` | A non-empty list turns on committee relabeling: every model writes a candidate relabel of every trace, and `teacher_model_name` combines them into the final one |
| `committee_max_input_length` | `250000` | Characters. A trace whose combining input would exceed this skips the committee and is edited by the teacher directly. That input is the trace's length times (1 + committee size): the original trace plus one full rewrite per member |

## Cross-field validation

These fail at config load.

- A reasoning teacher (every teacher except `Qwen3-235B-A22B-Instruct-2507`,
  `Qwen3-480B-A35B-Coder` and `Qwen2.5-VL-72B-Instruct`) requires `synthgen.teacher_temperature`
  in [0.5, 0.7].
- `base.enable_thinking: true` requires a reasoning student (`reasoning-models.md`).
- `visual_task: true` requires a vision-capable model in every teacher and judge role, and a
  vision-capable student. See `model-catalog.md`.
- Chat completion with tools declared restricts the student and teacher
  (`model-catalog.md` § Tool-calling compatibility).
- `synthgen.max_tool_calls_per_turn` must match whether the job description declares tools
  (`data-preparation/chat-completion.md` § Tool calls per turn).

The two in-context exemplar counts (`synthgen.num_positive_exemplars_per_generation`,
`synthgen.num_negative_exemplars_per_generation`) are not validated: every generation job lowers
them to the number of rows the split it draws from holds, down to zero on an empty split.

Inert: `evaluation.batch_size`, `synthgen.validation_max_answer_length`,
`synthgen.parallel_llm_calls`, `synthgen.basic_mutators_to_use`,
`tuning.use_lora`, `tuning.awq_quantize_tuned_model`, `tuning.train_eval_split`. They appear in every expanded
config the platform returns and have no effect. Leave them untouched in a config you override, and do not
add them to one you write.

Also without effect: `evaluation.num_few_shot_examples` and
`tuning.num_few_shot_examples_student`. Teacher evaluation, training and student evaluation use
each row as it is, with no examples added. To show the model examples, put them in the system
prompt: `task_description`, or each row's system message with `base.variable_system_prompt` on
(`variable-system-prompts.md`).

## Conversation expansion

`base.should_expand_dataset` decides whether a conversation becomes one example per assistant
turn. It applies to training, to evaluation, and to the expansion before
`synthgen.clean_training_targets` and the reasoning backfill.

| Value | Effect |
|---|---|
| `auto` (default) | Expands multi-turn tasks, unless at least half of the examples share history prefixes with other examples (data that is already split per turn) |
| `false` | Each supplied row stays one example, with only its final assistant turn as the target; earlier assistant turns are history. Evaluation scores each test row's final turn only. Use it for pre-split rows whose final responses were selected or rewritten |
| `true` | Expands every conversation, pre-split data included. Valid only for the chat completion tasks |

## Deprecated keys

These keys are not part of the config. Each one has a replacement or no equivalent:

| Deprecated key | Use instead |
|---|---|
| `synthgen.generation_target` | `synthgen.train_generation_target` |
| `trace_processing.num_traces_as_training_base` | `trace_processing.num_train_relabelled` |
| `traces_to_test_set.num_traces_to_relabel` | `trace_processing.num_test_relabelled` |
| `traces_to_test_set.num_synthetic_examples` | `synthgen.test_generation_target` |
| `traces_to_test_set.evaluate_original_model` | a teacher evaluation with the production model as `base.teacher_model_name` |
| `traces_to_test_set.min_relabelled_examples` | none |
| `trace_processing.max_unstructured`, `num_traces_as_testing_base`, `min_generated_examples`, `evaluate_original_model`, `convert_to_single_turn` | none |
| `synthgen.generation_per_unstructured_context`, `num_distractor_context_blocks` | none |
