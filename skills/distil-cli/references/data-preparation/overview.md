# Data Preparation: Overview

Input directory contract for all stages; read before any task-specific page.

## Input directory

```
input-dir/
├── config.yaml                 # training configuration (../configuration.md)
├── job_description.json        # task definition (../job-description.md)
├── train.jsonl                 # seed training data
├── test.jsonl                  # held-out test data
├── unstructured.jsonl          # required for open-book and closed-book QA
└── metadata.json               # optional: {"major_version": "1", "minor_version": "0"}
```

This directory IS the job input, on both runtimes; there is no separate compiled artifact,
and edits to the files take effect directly. The schema version comes from `metadata.json`
when present and is otherwise inferred from the first train row (chat-format rows mean V1),
so directories in this skill's format need no metadata file.

All data files are JSONL.

## Row format (chat format, JobInput V1)

Every runtime parses the same directory (backends just move files around). Train/test rows are
chat-format conversations.

| Task | Row shape |
|---|---|
| `classification`, `question-answering`, `question-answering-closed-book` | `{"messages": [<user>, <assistant>]}` |
| `question-answering-open-book` | `{"messages": [<user>, <assistant>], "context": "..."}` |
| `tool-calling-closed-book` | `{"messages": [<user>, <assistant with tool_calls>]}` |
| `multi-turn-tool-calling-closed-book` | `{"messages": [<user>, ..., <assistant with tool_calls>]}` |

- `<user>` / `<assistant>` are `{"role": ..., "content": "<non-empty>"}`; the final assistant
  message carries the answer or label as `content`.
- Tool-call assistant messages instead have EMPTY `content` and
  `"tool_calls": [{"type": "function", "function": {"name": ..., "arguments": {...}}}]`;
  note the key is `arguments` and it is a JSON object, not a string.
- `unstructured.jsonl` rows are `{"context": "..."}` (string).

Full examples: the task-specific pages.

## Validation rules (enforced at parse/job start; check before submitting)

- [ ] train and test are non-empty; every message `content` is non-empty, except the
      tool-call assistant messages noted above, whose `content` is empty by design
- [ ] context present when the task requires it
- [ ] per row, total length <= `synthgen.validation_max_total_length`
- [ ] train and test share NO identical rows (exact duplicates fail)
- [ ] unstructured (when provided): non-empty string `context` rows, at least
      `synthgen.num_unlabelled_exemplars_per_generation` of them
- [ ] classification: the label sets in train, test, and `classes_description` are
      identical; each class has at least the configured exemplar counts (defaults 2)
- [ ] tool calling: every tool call validates against the `tools` schemas; multi-turn
      conversations start with a user message, end with an assistant tool call, and have a
      valid role sequence

No hard minimum row count; aim for 20+ diverse train examples and a test set covering the
production distribution.

## Validate before submitting

Creating the SeedDataset is the validator (`../execution/` § The SeedDataset). It parses the
directory exactly as the jobs will — the full schema validation, no model calls — and rejects
an invalid bundle with the reason, before any job is created or any credit spent.

Read the reason rather than just the failure: rows in the wrong shape fail with
`messages: Field required` on every line, a missing required file is named directly, and a
task the platform does not support comes back as a list of the ones it does. Stage outputs
(trace processing, synthgen) are already valid job-input directories, so validate what you
author, and re-upload after edits.
