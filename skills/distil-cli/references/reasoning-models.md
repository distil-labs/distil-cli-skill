# Reasoning Models

A reasoning student writes its reasoning in a thinking block (`<think>…</think>` for Qwen3)
before the answer. One config key controls it: `base.enable_thinking` (default `false`). With it
on, synthetic data generation writes the reasoning, and the student is trained, evaluated and
served with its chat template's thinking mode on. It is allowed for every task type.

## When to propose it

- Propose it when reaching the answer takes several steps (a calculation, a rule applied to
  facts, a choice between close options) and a normal student is short of the teacher.
- It costs output tokens on every request, so latency goes up. Metrics score only the answer, so
  the reasoning must pay for itself in the primary metric.
- Train the same student with and without thinking on the same data and compare. Do not make it
  the default.
- The user's teacher choice is separate: a `-thinking` teacher (`model-catalog.md`) does not make
  the student a reasoning student.

## Supported students

`Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-8B`, `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`,
`Qwen3.5-9B`, `Qwen3.6-35B-A3B`, `Nemotron-3.5-Lightning-30B-A3B`. Any other student fails
config load with `base.enable_thinking=true is not available for student model '<name>'`. This
includes `Qwen3-4B-Instruct-2507`, so for a 4B-class reasoning student use `Qwen3.5-4B`.

## Where the reasoning comes from

1. **Generation.** The teacher writes `reasoning_content` next to every assistant message it
   generates.
2. **Backfill.** After generation, a teacher pass visits every training row, seed rows
   included, and writes reasoning on the final assistant message when it has none. It changes
   only the reasoning, never `content` or `tool_calls`. It costs one teacher call per row
   without reasoning. Rows still without reasoning after a retry are dropped, and the log says
   `Reasoning backfill: kept X of Y`.

The teacher writes in the first person, works the answer out instead of explaining it, rules
out the nearest alternative, and invents no facts. There is no length parameter. To steer the
length or style, say it in `task_description` (`job-description.md`).

## Data format

- V1 rows: `reasoning_content` on the assistant message. For tool calls, on the assistant
  message that carries `tool_calls`.
  ```json
  {"messages": [{"role": "user", "content": "..."}, {"role": "assistant", "content": "5", "reasoning_content": "..."}]}
  ```
- A `reasoning` key, or any key at the row's top level, is dropped without an error. Check the
  key name when the user supplies their own reasoning.
- Reasoning is optional per row. The backfill fills the gaps.
- Multi-turn rows are split into one row per assistant turn, and only the final turn keeps its
  reasoning. History reasoning is removed, because a served model never sees it.
- With `enable_thinking: false`, reasoning in the data is removed on read. The same data serves
  both kinds of student.
- Legacy format: a top-level `reasoning` next to `question`/`answer`. Legacy multi-turn tool
  calling cannot carry reasoning.
- `test.jsonl` needs no reasoning.

## Stage order

The config is inherited: seed dataset → training dataset → model. Set `enable_thinking` at the
point where the training data is generated.

- **From a dataset:** set `base.enable_thinking: true` in the seed dataset's `config.yaml`.
  Teacher evaluation, synthgen and training then inherit it.
- **From traces:** trace processing rejects the flag (`Trace processing does not support
  base.enable_thinking=true`). Process the traces with it off. Then submit synthgen as an
  override on the resulting seed dataset with `base.enable_thinking: true`
  (`platform.md` § Overrides: read the parent config back, edit that one field, send it whole).
- **Adding reasoning to an existing labelled set:** `synthgen.generation_target: 0` skips
  generation but still runs the backfill. Not for `question-answering-closed-book`.
- **Never turn it on only at training time** on a training dataset generated without it. The
  rows have no reasoning, nothing adds it at training time, and the student learns an empty
  thinking block without any error. Generate a new training dataset instead.

## Length budget

- `tuning.max_completion_length` (default 2048 tokens) caps what the student generates in
  evaluation and RLVR, reasoning included. A completion that is cut off scores near zero. If the
  longest training completion exceeds it, training logs `tuning.max_completion_length (X) is
  below the longest training completion (Y)`. Raise it when that warning appears or when student
  scores are near zero with long reasoning.
- `synthgen.validation_max_total_length` (default 30000 chars) is also checked after the
  backfill. Rows that grew past it are dropped.
- After synthgen, compare the train row count with the target. Backfill and length drops can
  remove rows, and they are not replaced.

## Evaluation

Metrics read only `content` and `tool_calls`. There is no reasoning metric. The student is
evaluated with thinking on, at the vendor's thinking sampling settings (Qwen3: temperature 0.6,
top_p 0.95, top_k 20; Qwen3.5/Qwen3.6: temperature 1.0, top_p 0.95, top_k 20). To judge the
reasoning itself, read it in the evaluation output rows.

## Deployment

- `model_client.py` sends `chat_template_kwargs: {"enable_thinking": true}` with
  `temperature=0`. Use it, as for any model (`deployment.md`).
- Hosted deployment supports reasoning students with no extra setup.
- Local vLLM needs the reasoning parser for the family, or the thinking block stays at the start
  of `content`:

  | Student | vLLM flags |
  |---|---|
  | Qwen3 | `--enable-auto-tool-choice --tool-call-parser hermes --reasoning-parser qwen3` |
  | Qwen3.5, Qwen3.6 | `--enable-auto-tool-choice --tool-call-parser qwen3_xml --reasoning-parser qwen3` |
  | Nemotron | `--enable-auto-tool-choice --tool-call-parser qwen3_xml --reasoning-parser nemotron_v3` |

- With the parser, the reasoning comes back in `reasoning_content` (or `reasoning` in newer
  vLLM) and `content` holds only the answer.
- Reasoning text from a reasoning student is expected. It is a smoke-test failure only for a
  model trained with `enable_thinking: false`.
