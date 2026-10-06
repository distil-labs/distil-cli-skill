# Configuration

`config.yaml` has six sections: `base`, `tuning`, `evaluation`, `synthgen`,
`trace_processing`, `traces_to_test_set`. Only `base.task` is required; every other parameter
has a default. Always set `task`, `student_model_name` and `teacher_model_name`.

## base

| Parameter | Default | Notes |
|---|---|---|
| `task` | required | See `task-types.md` |
| `visual_task` | `false` | Inputs carry images; QA tasks only, and every model must be vision-capable |
| `enable_thinking` | `false` | Reasoning student. See `reasoning-models.md` |
| `student_model_name` | `Llama-3.2-1B-Instruct` | See `model-catalog.md` |
| `teacher_model_name` | `openai.gpt-oss-120b` | See `model-catalog.md` |
| `random_seed` | `123` | Seeds sampling everywhere, including mutators |
| `llm_num_parallel_requests` | `4` | Parallel LLM calls across teacher/synthgen/judge; raising it helps only until per-call latency and between-batch validation dominate |
| `should_expand_dataset` | `"auto"` | `true`, `false` or `auto`. § Conversation expansion |

## tuning

| Parameter | Default | Notes |
|---|---|---|
| `learning_rate` | `5e-5` | AdamW |
| `learning_rate_scheduler` | `linear` | `cosine`, `linear`, `constant` |
| `weight_decay` | `0.0` | |
| `warmup_ratio` | `0.05` | |
| `bf16` | `true` | |
| `lora_r` | `64` | Rank of the LoRA adapter, which is the trained model. One of 8, 16, 32, 64, 128, 256, 320 or 512. alpha = `lora_r * lora_alpha_multiplier` |
| `lora_alpha_multiplier` | `1` | |
| `per_device_train_batch_size` | `1` | Higher uses more memory. At a fixed `num_train_epochs` it divides the optimizer-step count |
| `per_device_eval_batch_size` | `1` | |
| `num_train_epochs` | `4` | Steps ≈ rows x epochs / (batch x `gradient_accumulation_steps`) |
| `gradient_accumulation_steps` | `1` | Multiplies effective batch size |
| `num_few_shot_examples_student` | `0` | Few-shot for student eval/tuning. § Cross-field validation |
| `enable_trainer_internal_eval` | `false` | Per-epoch validation during training. Final metrics come from the post-training suite either way |
| `memory_optimized_training` | `false` | Only when training runs out of GPU memory; much slower |
| `use_qlora` | `false` | 4-bit NF4 base model. Linux-only bitsandbytes |
| `max_completion_length` | `2048` | Tokens the student may generate in evaluation and RLVR. Raise it when training warns it is below the longest training completion |

RLVR (optional RL stage after SFT, enabled when `rlvr_dataset_size > 0`):
`rlvr_dataset_size` 0.0, `rlvr_llm_as_a_judge_model_name` inherits `base.teacher_model_name`,
`rlvr_per_device_batch_size` 6 (must be a multiple of `rlvr_num_generations` 6),
`rlvr_num_train_epochs` 1.

## evaluation

| Parameter | Default | Notes |
|---|---|---|
| `num_few_shot_examples` | `1` | Teacher evaluation few-shot. At least one per class for classification. § Cross-field validation |
| `llm_as_a_judge_model_name` | inherits `base.teacher_model_name` | Set it once and keep it fixed across all runs. `model-catalog.md` § Defaults |

## synthgen

| Parameter | Default | Notes |
|---|---|---|
| `generation_target` | `10000` | A target, not an exact count: generation runs in `generation_iteration_size` batches until the target is met, so the result can exceed it by up to one batch, and validation losses shift where that boundary is |
| `generation_in_single_call` | `4` | Examples per teacher call |
| `generation_iteration_size` | `128` | Generate-validate batch size, and also the granularity `generation_target` rounds up to |
| `num_positive_exemplars_per_generation` | `1` | In-context examples per generation call (for classification, of the class being generated, and a per-class floor on train data). § Cross-field validation |
| `num_negative_exemplars_per_generation` | `1` | In-context examples for the classes NOT being generated. Classification only. § Cross-field validation |
| `num_unlabelled_exemplars_per_generation` | `1` | Unstructured dataset must be at least this size |
| `validation_max_total_length` | `30000` | Chars, question+answer+context; applies to uploaded data too |
| `validation_similarity_threshold` | `0.95` | Dedup vs seed data. Lower it if synthgen produces near-duplicates |
| `teacher_temperature` | `0.7` | § Cross-field validation for reasoning teachers |
| `teacher_max_tokens` | `32000` | |
| `match_generated_distribution_to_seed` | `false` | Classification only |
| `output_is_json` | `false` | QA only. Also forces answers in uploaded data to be valid JSON |
| `mutators` | `[]` | One entry per dimension to vary. `mutators.md` |
| `mutator_update_frequency` | `5` | Batches between two classifier runs of an adaptive mutator. `mutators.md` § Adaptive mutators |
| `max_tool_calls_per_turn` | `null` | Chat completion only: cap on tool calls per generated assistant turn. `data-preparation/chat-completion.md` § Tool calls per turn |
| `clean_training_targets` | `false` | Final teacher pass that minimally repairs corrupted/truncated training targets. Expansion follows `base.should_expand_dataset`; `false` cleans only supplied final-turn targets |

## trace_processing

Read by both trace jobs: trace processing, and test set from traces for how it relabels.

| Parameter | Default | Notes |
|---|---|---|
| `relabel` | `true` | Teacher/committee rewrites labels. `false` keeps the original trace labels |
| `relevance_filtering` | `false` | `true` has an LLM score traces and drop low relevance/coherence, at one LLM pass over every trace |
| `relevance_filtering_batch_size` | `32` | |
| `min_relevance_score` | `4` | 1-5 |
| `min_coherence_score` | `3` | 1-5. Lower lets corrupted traces through for committee repair |
| `num_traces_as_training_base` | `200` | Traces relabeled into the train split. Both trace jobs refuse a trace set smaller than this, test set from traces included, since both parse the traces with the same validation. Leftover traces become unstructured data |
| `max_unstructured` | `10000` | |
| `observation_format` | `openai_messages` | See `data-preparation/traces.md` |
| `remove_system_prompt_from_traces` | `true` | `data-preparation/traces.md` § Conversion guidance |
| `compress_job_description` | `false` | For very long task descriptions |
| `teacher_model_name` | inherits `base.teacher_model_name` | Does filtering and relabel arbitration. `model-catalog.md` § Defaults |
| `relabelling_committee_models` | `[]` | Non-empty list enables committee relabeling |
| `committee_max_input_length` | `250000` | Chars. Traces whose projected committee-aggregator input exceeds this skip the committee (direct teacher edit) |

## traces_to_test_set

Read by the test set from traces job (`../stages/test-set-from-traces.md`).

| Parameter | Default | Notes |
|---|---|---|
| `num_traces_to_relabel` | `200` | Traces relabeled into test examples, with the `trace_processing` relabeling settings. They leave the trace set |
| `num_synthetic_examples` | `0` | Synthetic test examples generated from the relabeled ones with the `synthgen` section. The next `2 * num_synthetic_examples` traces are the generation context and also leave the trace set. A floor: generation runs in `synthgen.generation_iteration_size` batches, so more rows can come back |
| `evaluate_original_model` | `false` | Scores the original model's answers on the relabeled traces with the student's metric suite. `false` writes null metrics. `evaluation-metrics.md` § Verdicts |
| `min_relabelled_examples` | `0` | Fails the job when fewer relabeled test examples survive. Best kept at the default 0, which sets no floor |

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
- No in-context exemplar count may exceed the number of train rows. The four counts are
  `synthgen.num_positive_exemplars_per_generation`,
  `synthgen.num_negative_exemplars_per_generation`, `evaluation.num_few_shot_examples` and
  `tuning.num_few_shot_examples_student`. Every job rejects a count above the train row count,
  except trace processing, which lowers the counts to fit the train split it produced
  (`../stages/trace-processing.md`):

  ```
  {'synthgen.num_positive_exemplars_per_generation': 3} asks for more in-context exemplars
  than the training dataset holds (1). Lower the counts, or provide more training data.
  ```

Inert: `evaluation.batch_size`, `synthgen.validation_max_answer_length`,
`synthgen.parallel_llm_calls`, `synthgen.basic_mutators_to_use`,
`tuning.awq_quantize_tuned_model`, `tuning.train_eval_split`,
`trace_processing.num_traces_as_testing_base`, `trace_processing.min_generated_examples`,
`trace_processing.evaluate_original_model`. They appear in every config the platform returns
and have no effect. Leave them untouched in a config you override, and do not add them to one
you write.

## Conversation expansion

`base.should_expand_dataset` decides whether a conversation becomes one example per assistant
turn. It applies to training, to evaluation, and to the expansion before
`synthgen.clean_training_targets` and the reasoning backfill.

| Value | Effect |
|---|---|
| `auto` (default) | Expands multi-turn tasks, unless at least half of the examples share history prefixes with other examples (data that is already split per turn) |
| `false` | Each supplied row stays one example, with only its final assistant turn as the target; earlier assistant turns are history. Evaluation scores each test row's final turn only. Use it for pre-split rows whose final responses were selected or rewritten |
| `true` | Expands every conversation, pre-split data included. Valid only for the chat completion tasks |
