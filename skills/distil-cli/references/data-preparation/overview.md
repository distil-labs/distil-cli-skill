# Data Preparation: Overview

Input directory contract for all stages. Read it before any task-specific page.

## Input directory

```
input-dir/
├── config.yaml                 # training configuration (../configuration.md)
├── job_description.json        # task definition (../job-description.md)
├── train.jsonl                 # seed training data (present; may be empty)
├── test.jsonl                  # held-out test data (present; may be empty)
├── unstructured.jsonl          # required for open-book and closed-book QA
└── metadata.json               # optional: {"major_version": "1", "minor_version": "0"}
```

This directory IS the job input, on both runtimes. There is no separate compiled artifact, and
edits to the files take effect directly. The schema version comes from `metadata.json` when
present, and is otherwise inferred from the first train row (chat-format rows mean V1), so
directories in this skill's format need no metadata file.

All data files are JSONL.

## Row format (chat format, JobInput V1)

Every runtime parses the same directory (backends only move files around). Train and test rows
are chat-format conversations.

| Task | Row shape |
|---|---|
| `classification`, `question-answering`, `question-answering-closed-book` | `{"messages": [<user>, <assistant>]}` |
| `question-answering-open-book` | `{"messages": [<user>, <assistant>], "context": "..."}` |
| `tool-calling-closed-book` | `{"messages": [<user>, <assistant with tool_calls>]}` |
| `multi-turn-tool-calling-closed-book` | `{"messages": [<user>, ..., <assistant with tool_calls>]}` |
| `chat-completion` | `{"messages": [<user>, <assistant>, ...]}`, assistant turns carry content, `tool_calls`, or both |
| `chat-completion-agentic` | `{"messages": [<user>, <assistant with tool_calls>, <tool>, ..., <assistant>]}` |

- `<user>` and `<assistant>` are `{"role": ..., "content": "<non-empty>"}`. The final assistant
  message carries the answer or label as `content`.
- Tool-call assistant messages instead have EMPTY `content` and
  `"tool_calls": [{"type": "function", "function": {"name": ..., "arguments": {...}}}]`. The
  key is `arguments`, and it is a JSON object, not a string.
- Chat completion tasks relax this: their assistant turns carry content, `tool_calls`, or both
  (at least one), and parallel calls are allowed. See `chat-completion.md`.
- `unstructured.jsonl` rows are `{"context": "..."}` (string).

Full examples: the task-specific pages.

## Validation rules (enforced at parse/job start, so check before submitting)

- [ ] `train.jsonl` and `test.jsonl` are both PRESENT. Either can be empty (§ Empty
      splits); a missing file is an error. Every message `content` is non-empty, except
      the tool-call assistant messages noted above, whose `content` is empty by design
- [ ] context present when the task requires it
- [ ] per row, total length <= `synthgen.validation_max_total_length`
- [ ] train and test share NO identical rows (exact duplicates fail)
- [ ] unstructured (when provided): non-empty string `context` rows, at least
      `synthgen.num_unlabelled_exemplars_per_generation` of them
- [ ] classification: the label sets in train, test, and `classes_description` are
      identical. Each class has at least the configured exemplar counts (defaults 1)
- [ ] no in-context exemplar count exceeds the number of train rows. The four counts are
      `synthgen.num_positive_exemplars_per_generation`,
      `synthgen.num_negative_exemplars_per_generation`,
      `evaluation.num_few_shot_examples` and `tuning.num_few_shot_examples_student`
- [ ] tool calling: every tool call validates against the `tools` schemas. Multi-turn
      conversations start with a user message, end with an assistant tool call, and have a
      valid role sequence
- [ ] chat completion: conversations start with a user message, end with an assistant message,
      and have a valid role sequence. Tool results only in `chat-completion-agentic`, and only
      after an assistant turn that made calls. Calls validate against `tools`; tool calls in
      the data with no declared tools fail

There is no hard minimum row count. Aim for 20+ diverse train examples and a test set covering
the production distribution.

## Empty splits

`train.jsonl` and `test.jsonl` must both be there. Either can be empty. The file itself is the
statement: an empty file says "no labelled data of this kind", and a missing file is almost
always a wrong path.

This is a narrow allowance, not a starting point to recommend. The normal shape is a populated
train split and a populated test split, and a populated split is held to every rule above:
the classification label sets must still match, and a class too thin for
the exemplar counts still fails. Use an empty split only when the user genuinely has no data of
that kind, for example a job that would otherwise carry one dummy row to satisfy the parser.

What each stage does with an empty split:

| Stage | Empty split | Behavior |
|---|---|---|
| trace processing | either | Writes it out. `min_generated_examples` is the only floor |
| synthgen | train | Runs. Prompt blocks drop to zero-shot, and the output becomes the train split |
| teacher evaluation | test | REFUSES. There is nothing to score the teacher against |
| model training | train | REFUSES. There is nothing to train on |
| model training | test | Runs and produces a model, but no evaluation results at all |

Consequences to state to the user before they choose this:

- **No test set means no scores.** Teacher evaluation cannot run. Training still produces a
  model, but no base-vs-tuned comparison, no metric values, and no evaluation output
  directories. The metrics are recorded as "not available" rather than as a score, so a run
  with no test set is not mistaken for one whose scoring failed. Add a test set later and
  evaluate then.
- **No train set is workable at the start and fatal at the end.** Synthgen fills the split from
  the task description, the tool or class definitions, and `unstructured.jsonl`. Training
  refuses if the split is still empty when it starts. So synthgen must run first.

Order matters, and the two refusals name themselves:

```
Teacher evaluation scores the teacher against a test set, and this job has none:
test.jsonl is empty. Provide test examples, or skip this stage.

Finetuning needs a training set, and this job has none: train.jsonl is empty.
Generate synthetic data first, or provide training examples.
```

A test set with no train set is the more useful of the two shapes: teacher evaluation runs
zero-shot, synthgen and training both run, and every score is still available.

Empty splits apply to the chat format (JobInput V1) only. The older format still requires both
splits.

## Validate before submitting

Creating the SeedDataset is the validator (`../execution/` § The SeedDataset). It parses the
directory exactly as the jobs will (the full schema validation, no model calls) and rejects
an invalid bundle with the reason, before any job is created or any credit spent.

Read the reason rather than the failure alone: rows in the wrong shape fail with
`messages: Field required` on every line, a missing required file is named directly, and a
task the platform does not support comes back as a list of the ones it does. Stage outputs
(trace processing, synthgen) are already valid job-input directories, so validate what you
author, and re-upload after edits.
