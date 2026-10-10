# Migrating Old Entities

A seed dataset (`train.jsonl`, `test.jsonl`, optional `unstructured.jsonl`) and a training
dataset (`train.jsonl`, `test.jsonl`) are readable with `distil seed-dataset` and
`distil training-dataset` `list`, `show`, `download` and `download-metadata`; their other
commands, and `distil traces expand-test-set`, exit 1 naming the command to use. To build on one,
move its files into a Dataset:

1. Download it: `distil seed-dataset download -d old <id>` or
   `distil training-dataset download -d old <id>` (free). For an old PreparedTraces,
   `distil traces download -d old <id>` gives its `traces.jsonl` only; the config, job
   description and test set it was uploaded with are not downloadable, so take them from your
   own files.
2. Fix what a Dataset rejects:
   - **Flat rows** (`{"question": ..., "answer": ...}`): rewrite every row as a `messages`
     conversation (`data-preparation/overview.md` § Row format).
   - **`unstructured.jsonl`**: if trace processing wrote it (each row is one serialised trace),
     rename it to `traces.jsonl` and set `trace_processing.observation_format:
     unstructured_with_openai_messages` in the config. If it holds uploaded documents, leave it
     out: a Dataset has no place for context documents.
   - **`final-synthetic-dataset/`**: move its `train.jsonl` and `test.jsonl` to the top level.
   - **Task types a Dataset does not accept** (`question-answering-open-book`, `question-answering-closed-book`,
     `information-extraction`, `question-answering-open-book-synthetic-context`,
     `question-answering-legacy`): set `base.task: question-answering` and drop `context` from
     the rows (`task-types.md`). A multi-turn tool-calling task maps to `chat-completion`
     or `chat-completion-agentic`.
   - **A QA job description** with text in `input_description` or `context_description`:
     remove the two keys.
   - **Deprecated config keys**: replace or delete them (`configuration.md` § Deprecated keys).
3. Upload the directory: `distil dataset create --data old` (`execution/cli.md` § The Dataset).
   The create validates it and names the first row that fails.

The new Dataset has no parent: it is the root of a new chain. A test set from an old entity is
a locked test set like any other (`../stages/build-a-test-set.md` Step 5), and a train set from
an old TrainingDataset is a train set to top up (`../stages/build-a-train-set.md` § Topping Up).
