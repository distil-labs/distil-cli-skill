# Reasoning Models

Everything about reasoning students. A reasoning student writes its reasoning in a thinking
block (`<think>…</think>` for Qwen3) before the answer. One config key controls it:
`base.enable_thinking` (default `false`). With it on, synthetic data generation writes the
reasoning, and the student is trained, evaluated and served with thinking on. It is allowed for
every task type.

## When to propose it

- When reaching the answer takes several steps (a calculation, a rule applied to facts, a
  choice between close options) and a normal student is short of the teacher.
- It adds output tokens to every request, so latency goes up, and metrics score only the
  answer. Train the same student with and without thinking on the same data and compare. Do not
  make it the default.
- A `-thinking` teacher (`model-catalog.md`) does not make the student a reasoning student.

## Supported students

`Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-8B`, `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`,
`Qwen3.5-9B`, `Qwen3.8-27B`, `Qwen3.6-35B-A3B`. Any other student fails config load with
`base.enable_thinking=true is not available for student model '<name>'`. For a 4B reasoning
student use `Qwen3.5-4B`, not `Qwen3-4B-Instruct-2507`.

## Where to set it

`base.enable_thinking: true` goes in the config of the synthetic data generation run into the
train split, as an override on the Dataset it runs on (`platform.md` § Overrides). The Dataset
it writes carries the setting, so training inherits it and needs no override.

- Relabelling rejects `true`: traces carry no reasoning and the job writes none. Build the test
  set and relabel the train rows with it off, then turn it on for generation.
- Turning it on only at training, on a Dataset generated without it, trains an empty thinking
  block without an error. Generate again with it on instead.
- `synthgen.train_generation_target: 0` skips generation and still writes reasoning onto the
  existing train rows, which adds reasoning to a labelled set without generating new rows.
- A run with it on expands multi-turn rows per turn, so a later top-up cannot use that
  Dataset's train split as examples (`../stages/build-a-train-set.md` § Topping Up).

## Where the reasoning comes from

1. **Generation.** The teacher writes `reasoning_content` next to every assistant message it
   generates.
2. **Backfill.** After generation, a teacher pass writes reasoning on the final assistant message
   of every training row that has none, uploaded and relabelled rows included. It changes only the reasoning,
   costs one teacher call per row, and drops rows still without reasoning after a retry. The log
   says `Reasoning backfill: kept X of Y`.

The teacher writes in the first person, works the answer out rather than explaining it, rules
out the nearest alternative, and invents no facts. Instructions on the style and length of the
reasoning go in `synthetic_data_generation_instructions` (`job-description.md`), for example
"Keep the reasoning under 250 tokens: one short line per check." For a smoke, read the
`reasoning_content` of a handful of rows: it should work the answer out, not restate it.

## Data format

- `reasoning_content` on the assistant message; for tool calls, on the assistant message that
  carries `tool_calls`.
  ```json
  {"messages": [{"role": "user", "content": "..."}, {"role": "assistant", "content": "5", "reasoning_content": "..."}]}
  ```
- In train and test rows, a `reasoning` key, or any other key at the row's top level, is
  dropped without an error. Check the key name when the user supplies their own reasoning.
- Reasoning is optional per row; the backfill fills the gaps. `test.jsonl` needs none.
- Conversations are split into one row per assistant turn, and only the final turn keeps its
  reasoning.
- With `enable_thinking: false`, reasoning in the data is removed on read, so the same data
  serves both kinds of student.

## Length budget

- `tuning.max_completion_length` (default 2048 tokens) caps what the student generates in
  evaluation, reasoning included. A cut-off completion scores near zero. Training logs
  `tuning.max_completion_length (X) is below the longest training completion (Y)` when it is
  too low; raise it then.
- `synthgen.validation_max_total_length` is also checked after the backfill, and rows that grew
  past it are dropped. Backfill and length drops are not replaced, so compare the train row
  count with the target after synthgen.

## Evaluation

Metrics read only `content` and `tool_calls`; there is no reasoning metric. The student is
evaluated with thinking on, at the vendor's thinking sampling settings (Qwen3: temperature 0.6,
top_p 0.95, top_k 20; Qwen3.5/Qwen3.6/Qwen3.8: temperature 1.0, top_p 0.95, top_k 20). To judge
the reasoning, read it in the prediction rows.

## Deployment

- `model_client.py` sends `chat_template_kwargs: {"enable_thinking": true}` with
  `temperature=0`. Use it, as for any model (`deployment.md` § Why model_client.py instead of
  raw requests).
- A hosted deployment needs no extra setup.
- Local vLLM needs the reasoning parser for the family, or the thinking block stays at the start
  of `content`. The flags per family: `deployment.md` § Serving locally. With the parser, the
  reasoning comes back in `reasoning_content` and `content` holds only the answer.
- Reasoning text in the answer is expected from a reasoning student. It is a smoke-test failure
  only for a model trained with `enable_thinking: false`.
