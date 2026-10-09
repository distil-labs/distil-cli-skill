# Data Preparation: Overview

The Dataset directory contract. Read it before any task-specific page.

## The Dataset directory

```
input-dir/
├── config.yaml                 # training configuration (../configuration.md)
├── job_description.json        # task definition (../job-description.md)
├── train.jsonl                 # labelled training rows (optional)
├── test.jsonl                  # held-out test rows (optional)
└── traces.jsonl                # production traces (optional; ../data-preparation/traces.md)
```

The same five files are what every Dataset on the platform holds and what `dataset download`
writes, so a downloaded directory is accepted unchanged by `dataset create --data`. The config
and the job description are required. Each data file is optional: a missing one is an empty
split, and the expand operations fill the empty ones (`../platform.md` § The expand
operations). All data files are JSONL.

## Row format

Train and test rows are chat-format conversations.

| Task | Row shape |
|---|---|
| `classification`, `question-answering` | `{"messages": [<user>, <assistant>]}` |
| `chat-completion` | `{"messages": [<user>, <assistant>, ...]}`, assistant turns carry content, `tool_calls`, or both |
| `chat-completion-agentic` | `{"messages": [<user>, <assistant with tool_calls>, <tool>, ..., <assistant>]}` |

- `<user>` and `<assistant>` are `{"role": ..., "content": "<non-empty>"}`. The final assistant
  message carries the answer or label as `content`.
- Chat completion rows, tool calls and role rules: `chat-completion.md`.
- An assistant message may carry `reasoning_content` for a reasoning student
  (`../reasoning-models.md`); it is dropped on load unless `base.enable_thinking` is true.
- The flat `{"question": ..., "answer": ...}` row format of the previous platform is refused
  (`../migrating-old-entities.md`).

Trace rows follow `trace_processing.observation_format` (`traces.md`).

Full examples: the task-specific pages.

## Validation rules

Enforced at upload and before every expand, so check before submitting.

- [ ] every message `content` is non-empty, except a chat completion assistant turn that
      carries only `tool_calls`
- [ ] per row, total length <= `synthgen.validation_max_total_length` (`../configuration.md`)
- [ ] train and test share NO identical rows (exact duplicates fail)
- [ ] classification: the label sets in train, test, and `classes_description` are
      identical; a populated split must hold every class
- [ ] chat completion: the conversation rules in `chat-completion.md`; rows with parallel tool
      calls need `synthgen.max_tool_calls_per_turn` of 2 or more
- [ ] `visual_task` matches whether the rows carry images
- [ ] `traces.jsonl` parses in the configured `observation_format`

There is no hard minimum row count. Aim for 20+ diverse train examples and a test set covering
the production distribution.

## Empty splits

A missing or empty data file says "no data of this kind". A populated split is held to every
rule above. What each stage does with an empty split:

| Stage | Empty split | Behavior |
|---|---|---|
| relabel traces | train or test | Fills it. The job needs traces, not rows |
| synthetic data generation | the split it fills | Runs zero-shot, and the output becomes the split |
| synthetic data generation | traces | Runs without context (`../platform.md` § The expand operations) |
| teacher evaluation | test | REFUSES. There is nothing to score the teacher against |
| teacher evaluation | train | Runs zero-shot: `evaluation.num_few_shot_examples` is capped to the train rows |
| model training | train | REFUSES. There is nothing to train on |
| model training | test | Runs and produces a model, but no evaluation results at all |

Tell the user before they build on an empty split:

- **No test set means no scores**: no teacher evaluation, no base-vs-tuned comparison, no
  verdict. `slm metrics` returns null scores, which is not a failed job. The test set is built
  first for that reason (`../../stages/build-a-test-set.md`).
- **No train set means synthetic data generation must run before training**: it fills the
  split from the task description, the tool or class definitions and the traces, with no rows
  to imitate (`../../stages/build-a-train-set.md`).

The two refusals:

```
Teacher evaluation scores the teacher against a test set, and this job has none:
test.jsonl is empty. Provide test examples, or skip this stage.

Finetuning needs a training set, and this job has none: train.jsonl is empty.
Generate synthetic data first, or provide training examples.
```

## Validate before submitting

Creating the Dataset is the validator (`../execution/cli.md` § The Dataset): it parses the
directory exactly as the jobs will and rejects an invalid bundle with the reason, before any
job is created. Every expand validates the Dataset it writes with the same rules, so what you
write by hand is the only thing to check.
