# Configuration

`config.yaml` has five sections: `base`, `tuning`, `evaluation`, `synthgen`,
`trace_processing`. Only `base.task` is required. Every other parameter has a sensible
default.

Tiers: always set `task`, `student_model_name`, `teacher_model_name`. Revisit `synthgen`
mutator and target parameters when iterating (see `mutators.md`). Treat the rest as
expert-only.

## base

| Parameter | Default | Notes |
|---|---|---|
| `task` | required | See `task-types.md` |
| `visual_task` | `false` | Inputs carry images; QA tasks only, and every model must be vision-capable |
| `student_model_name` | `Llama-3.2-1B-Instruct` | See `model-catalog.md` |
| `teacher_model_name` | `openai.gpt-oss-120b` | See `model-catalog.md` |
| `random_seed` | `123` | Seeds sampling everywhere, including mutators |
| `llm_num_parallel_requests` | `4` | Parallel LLM calls across teacher/synthgen/judge; raising it helps only until per-call latency and between-batch validation dominate |

## tuning

| Parameter | Default | Notes |
|---|---|---|
| `learning_rate` | `5e-5` | AdamW |
| `learning_rate_scheduler` | `linear` | `cosine`, `linear`, `constant` |
| `weight_decay` | `0.0` | |
| `warmup_ratio` | `0.05` | |
| `bf16` | `true` | |
| `use_lora` | `true` | |
| `lora_r` | `64` | alpha = `lora_r * lora_alpha_multiplier` |
| `lora_alpha_multiplier` | `1` | |
| `per_device_train_batch_size` | `1` | **Not just a memory/speed knob.** At a fixed `num_train_epochs` it divides the optimizer-step count, so raising it trains the model less. Raise `num_train_epochs` proportionally when you raise it |
| `per_device_eval_batch_size` | `1` | |
| `num_train_epochs` | `4` | Steps ≈ rows x epochs / (batch x `gradient_accumulation_steps`). Keep that product stable when changing batch size |
| `train_eval_split` | `0.2` | Held out to pick the best checkpoint. Must be in (0, 1) |
| `gradient_accumulation_steps` | `1` | Multiplies effective batch size |
| `num_few_shot_examples_student` | `0` | Few-shot for student eval/tuning |
| `enable_trainer_internal_eval` | `false` | Per-epoch validation during training. Final metrics come from the post-training suite either way |
| `memory_optimized_training` | `false` | Only when training runs out of GPU memory; much slower |
| `use_qlora` | `false` | 4-bit NF4 base model. Needs `use_lora`, Linux-only bitsandbytes |

RLVR (optional RL stage after SFT, enabled when `rlvr_dataset_size > 0`):
`rlvr_dataset_size` 0.0, `rlvr_llm_as_a_judge_model_name` inherits `base.teacher_model_name`,
`rlvr_per_device_batch_size` 6 (must be a multiple of `rlvr_num_generations` 6),
`rlvr_num_train_epochs` 1.

## evaluation

| Parameter | Default | Notes |
|---|---|---|
| `num_few_shot_examples` | `1` | Teacher evaluation few-shot. At least one per class for classification |
| `llm_as_a_judge_model_name` | inherits `base.teacher_model_name` | Set only to judge with a different model than the teacher |

## synthgen

| Parameter | Default | Notes |
|---|---|---|
| `generation_target` | `10000` | A target, not an exact count: generation runs in `generation_iteration_size` batches until the target is met, so the result can exceed it by up to one batch, and validation losses shift where that boundary lands. Ignored for closed-book QA when `generation_per_unstructured_context` is set |
| `generation_in_single_call` | `4` | Examples per teacher call |
| `generation_iteration_size` | `128` | Generate-validate batch size, and also the granularity `generation_target` rounds up to |
| `generation_per_unstructured_context` | `null` | Closed-book QA only; target becomes this x len(unstructured) |
| `num_positive_exemplars_per_generation` | `2` | Also a per-class floor on train data (see data-preparation) |
| `num_negative_exemplars_per_generation` | `2` | Classification only |
| `num_unlabelled_exemplars_per_generation` | `1` | Unstructured dataset must be at least this size |
| `validation_max_total_length` | `30000` | Chars, question+answer+context; applies to uploaded data too |
| `validation_similarity_threshold` | `0.95` | Dedup vs seed data. Lower it if synthgen produces near-duplicates |
| `teacher_temperature` | `0.7` | Reasoning teachers require 0.5-0.7 (validation error otherwise) |
| `teacher_max_tokens` | `32000` | |
| `match_generated_distribution_to_seed` | `false` | Classification and tool calling |
| `num_distractor_context_blocks` | `0` | Above zero enables RAFT (open-book) |
| `output_is_json` | `false` | QA only. Also forces answers in uploaded data to be valid JSON |
| `basic_mutators_to_use` | `["complexity"]` | See `mutators.md` |
| `mutation_topics` | `[]` | See `mutators.md` |
| `clean_training_targets` | `false` | Final teacher pass that minimally repairs corrupted/truncated training targets. Multi-turn data is expanded into per-turn examples first, so every turn is covered |

## trace_processing

| Parameter | Default | Notes |
|---|---|---|
| `relabel` | `true` | Teacher/committee rewrites labels. `false` keeps original trace labels |
| `relevance_filtering` | `false` | `true` has an LLM score traces and drop low relevance/coherence, at one LLM pass over every trace |
| `relevance_filtering_batch_size` | `32` | |
| `min_relevance_score` | `4` | 1-5 |
| `min_coherence_score` | `3` | 1-5. Lower lets corrupted traces through for committee repair |
| `num_traces_as_training_base` | `200` | Leftover traces become unstructured data |
| `num_traces_as_testing_base` | `200` | Must be ≥ 1. Ignored when a test set is provided. Keep equal to the training base |
| `min_generated_examples` | `1` | Floor checked per split, after filtering and relabeling. Keep it below the smaller of the two base counts |
| `evaluate_original_model` | `true` | Judges the original model on the test split. `false` skips it and writes null metrics. This is the stage's LLM-judge cost, so turn it off for smokes |
| `max_unstructured` | `10000` | |
| `observation_format` | `openai_messages` | See `data-preparation/traces.md` |
| `remove_system_prompt_from_traces` | `true` | The system prompt belongs in the job description instead |
| `compress_job_description` | `false` | For very long task descriptions |
| `teacher_model_name` | inherits `base.teacher_model_name` | Does filtering and relabel arbitration. Set only to differ from the base teacher |
| `relabelling_committee_models` | `[]` | Non-empty list enables committee relabeling |
| `committee_max_input_length` | `250000` | Chars. Traces whose projected committee-aggregator input exceeds this skip the committee (direct teacher edit) |

## Cross-field validation (fails at config load)

- Reasoning teacher (every teacher except `Qwen2.5-VL-72B-Instruct`,
  `Qwen3-235B-A22B-Instruct-2507`, `Qwen3-480B-A35B-Coder`) requires
  `synthgen.teacher_temperature` in [0.5, 0.7].
- Tool-calling tasks require a supported student. Multi-turn additionally requires a supported
  teacher. See `model-catalog.md`.

Inert: `evaluation.batch_size`, `synthgen.validation_max_answer_length`,
`synthgen.parallel_llm_calls`, `tuning.awq_quantize_tuned_model`. They carry defaults and
appear in every config the platform returns, but have no effect. Leave them untouched in a
config you override. Do not add them to one you author.
