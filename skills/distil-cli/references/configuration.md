# Configuration

`config.yaml` has five sections: `base`, `tuning`, `evaluation`, `synthgen`,
`trace_processing`. Only `base.task` is required; every other parameter has a default. Always
set `task`, `student_model_name` and `teacher_model_name`.

A config written for the previous platform still loads: `synthgen.generation_target` is an
alias of `train_generation_target`, `trace_processing.num_traces_as_training_base` moves to
`num_train_relabelled`, `traces_to_test_set.num_traces_to_relabel` to `num_test_relabelled`
and `traces_to_test_set.num_synthetic_examples` to `synthgen.test_generation_target`; the rest
of the removed keys are dropped (`migrating-old-entities.md`).

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
| `num_few_shot_examples` | `1` | Teacher evaluation few-shot, drawn from the train split. § Cross-field validation |
| `llm_as_a_judge_model_name` | inherits `base.teacher_model_name` | Set it once and keep it fixed across all runs. `model-catalog.md` § Defaults |

## synthgen

| Parameter | Default | Notes |
|---|---|---|
| `train_generation_target` | `10000` | Synthetic rows one `generate-synthetic-data-train` run adds to the train split; `generation_target` is an alias. A target, not an exact count: generation runs in `generation_iteration_size` batches until the target is met, so the result can exceed it by up to one batch, and validation losses shift where that boundary is |
| `test_generation_target` | `0` | Synthetic rows one `generate-synthetic-data-test` run adds to the test split. Same rounding |
| `use_traces_as_context` | `true` | The traces in the Dataset serve as context for generation and the used ones leave the Dataset: `T + min(T, 1000)` for a target of T, or all that are left. Below `min(T / 4, 10)` traces, or with `false`, the job generates without context and leaves the traces alone. `platform.md` § The expand operations |
| `generation_in_single_call` | `4` | Examples per teacher call |
| `generation_iteration_size` | `128` | Generate-validate batch size, and also the granularity a generation target rounds up to |
| `num_positive_exemplars_per_generation` | `1` | In-context examples per generation call (for classification, of the class being generated). § Cross-field validation |
| `num_negative_exemplars_per_generation` | `1` | In-context examples for the classes NOT being generated. Classification only. § Cross-field validation |
| `num_unlabelled_exemplars_per_generation` | `1` | Context traces shown per generation call, when the run has context |
| `validation_max_total_length` | `30000` | Chars, question+answer+context; applies to uploaded data too |
| `validation_similarity_threshold` | `0.95` | Dedup against the rows already in the split. Lower it if synthgen produces near-duplicates |
| `teacher_temperature` | `0.7` | § Cross-field validation for reasoning teachers |
| `teacher_max_tokens` | `32000` | |
| `match_generated_distribution_to_seed` | `false` | Classification only |
| `output_is_json` | `false` | QA only. Also forces answers in uploaded data to be valid JSON |
| `mutators` | `[]` | One entry per dimension to vary. `mutators.md` |
| `mutator_update_frequency` | `5` | Batches between two classifier runs of an adaptive mutator. `mutators.md` § Adaptive mutators |
| `max_tool_calls_per_turn` | `null` (1 with tools declared, 0 without) | Chat completion only: cap on tool calls per generated assistant turn. `data-preparation/chat-completion.md` § Tool calls per turn |
| `clean_training_targets` | `false` | Final teacher pass that minimally repairs corrupted/truncated training targets. Expansion follows `base.should_expand_dataset`; `false` cleans only supplied final-turn targets |

## trace_processing

Read by `relabel-traces-train` and `relabel-traces-test` (`../stages/relabel-traces.md`);
`observation_format` is also read when a Dataset is created from traces.

| Parameter | Default | Notes |
|---|---|---|
| `relabel` | `true` | Teacher/committee rewrites labels. `false` keeps the original trace labels |
| `relevance_filtering` | `false` | `true` has an LLM score traces and drop low relevance/coherence, at one LLM pass over every trace |
| `relevance_filtering_batch_size` | `32` | |
| `min_relevance_score` | `4` | 1-5 |
| `min_coherence_score` | `3` | 1-5. Lower lets corrupted traces through for committee repair |
| `num_train_relabelled` | `200` | Traces one `relabel-traces-train` run turns into train rows. The Dataset must hold at least this many distinct traces or the job refuses to run; fewer rows can come out, because filtering drops some. The used traces leave the Dataset |
| `num_test_relabelled` | `200` | The same for `relabel-traces-test` and the test split |
| `observation_format` | `openai_messages` | See `data-preparation/traces.md` |
| `remove_system_prompt_from_traces` | `true` | `data-preparation/traces.md` § Conversion guidance |
| `compress_job_description` | `false` | For very long task descriptions |
| `teacher_model_name` | inherits `base.teacher_model_name` | Does filtering and relabel arbitration. `model-catalog.md` § Defaults |
| `relabelling_committee_models` | `[]` | Non-empty list enables committee relabeling |
| `committee_max_input_length` | `250000` | Chars. Traces whose projected committee-aggregator input exceeds this skip the committee (direct teacher edit) |

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
- `base.enable_thinking: true` is refused by relabelling (`../stages/relabel-traces.md`).

The four in-context exemplar counts (`synthgen.num_positive_exemplars_per_generation`,
`synthgen.num_negative_exemplars_per_generation`, `evaluation.num_few_shot_examples`,
`tuning.num_few_shot_examples_student`) are not validated: every job lowers them to the number
of rows the split it draws from holds, down to zero on an empty split. Read the counts a job ran
with from its config.

Inert: `evaluation.batch_size`, `synthgen.validation_max_answer_length`,
`synthgen.parallel_llm_calls`, `synthgen.basic_mutators_to_use`,
`tuning.awq_quantize_tuned_model`, `tuning.train_eval_split`. They appear in every expanded
config the platform returns and have no effect. Leave them untouched in a config you override, and do not
add them to one you write.

## Conversation expansion

`base.should_expand_dataset` decides whether a conversation becomes one example per assistant
turn. It applies to training, to evaluation, and to the expansion before
`synthgen.clean_training_targets` and the reasoning backfill.

| Value | Effect |
|---|---|
| `auto` (default) | Expands multi-turn tasks, unless at least half of the examples share history prefixes with other examples (data that is already split per turn) |
| `false` | Each supplied row stays one example, with only its final assistant turn as the target; earlier assistant turns are history. Evaluation scores each test row's final turn only. Use it for pre-split rows whose final responses were selected or rewritten |
| `true` | Expands every conversation, pre-split data included. Valid only for the chat completion tasks |
