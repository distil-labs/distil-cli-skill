# Data Preparation: Overview

Input directory contract for all stages. Read it before any task-specific page.

## Input directory

```
input-dir/
├── config.yaml                 # training configuration (../configuration.md)
├── job_description.json        # task definition (../job-description.md)
├── train.jsonl                 # seed training data (present; may be empty)
├── test.jsonl                  # held-out test data (present; may be empty)
├── unstructured.jsonl          # optional: unlabelled in-domain texts
└── metadata.json               # optional: {"major_version": "1", "minor_version": "0"}
```

This directory is the job input; edits to the files take effect directly. The schema version
comes from `metadata.json` when present and is otherwise inferred from the first train row, so
a directory in this format needs no metadata file. All data files are JSONL.

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
  (`../reasoning-models.md`).
- `unstructured.jsonl` rows are `{"context": "..."}` (string).

Full examples: the task-specific pages.

## Validation rules (enforced at parse/job start, so check before submitting)

- [ ] `train.jsonl` and `test.jsonl` are both PRESENT. Either can be empty (§ Empty
      splits); a missing file is an error. Every message `content` is non-empty, except
      a chat completion assistant turn that carries only `tool_calls`
- [ ] per row, total length <= `synthgen.validation_max_total_length` (`../configuration.md`)
- [ ] train and test share NO identical rows (exact duplicates fail)
- [ ] unstructured (when provided): non-empty string `context` rows, at least
      `synthgen.num_unlabelled_exemplars_per_generation` of them
- [ ] classification: the label sets in train, test, and `classes_description` are
      identical. Each class has at least the configured exemplar counts (defaults 1)
- [ ] no in-context exemplar count exceeds the number of train rows
      (`../configuration.md` § Cross-field validation)
- [ ] chat completion: the conversation rules in `chat-completion.md`

There is no hard minimum row count. Aim for 20+ diverse train examples and a test set covering
the production distribution.

## Empty splits

`train.jsonl` and `test.jsonl` must both be there, and either can be empty: an empty file says
"no labelled data of this kind", a missing file is an error. Use an empty split only when the
user has no data of that kind. A populated split is held to every rule above.

What each stage does with an empty split:

| Stage | Empty split | Behavior |
|---|---|---|
| trace processing | either | Writes it out. No floor applies. The test split is the PreparedTraces' `test.jsonl`, empty when there is none |
| synthgen | train | Runs. Prompt blocks drop to zero-shot, and the output becomes the train split |
| teacher evaluation | test | REFUSES. There is nothing to score the teacher against |
| teacher evaluation | train | Runs only with `evaluation.num_few_shot_examples: 0`, since few-shot examples come from the train split. Normally teacher evaluation runs on the seed data's train split |
| model training | train | REFUSES. There is nothing to train on |
| model training | test | Runs and produces a model, but no evaluation results at all |

Tell the user before they choose an empty split:

- **No test set means no scores**: no teacher evaluation, no base-vs-tuned comparison, no
  verdict. `slm metrics` returns null scores, which is not a failed job. A test set can be added
  later.
- **No train set means synthgen must run before training**: it fills the split from the task
  description, the tool or class definitions and `unstructured.jsonl`, with no seed examples to
  imitate.

The two refusals:

```
Teacher evaluation scores the teacher against a test set, and this job has none:
test.jsonl is empty. Provide test examples, or skip this stage.

Finetuning needs a training set, and this job has none: train.jsonl is empty.
Generate synthetic data first, or provide training examples.
```

A test set with no train set is the more useful of the two shapes: teacher evaluation runs
with `evaluation.num_few_shot_examples: 0`, synthgen and training both run, and every score is
still available.

## Validate before submitting

Creating the SeedDataset is the validator (`../execution/cli.md` § The SeedDataset): it parses
the directory exactly as the jobs will and rejects an invalid bundle with the reason, before any
job is created. Stage outputs are already valid job-input directories, so validate what
you write by hand.
