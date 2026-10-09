# Migrating Old Entities

The platform used to hold data in three entities: PreparedTraces with a config, a job
description and an optional test set; SeedDataset (`train.jsonl`, `test.jsonl`, optional
`unstructured.jsonl`); and TrainingDataset (`train.jsonl`, `test.jsonl`). The Dataset replaced
the last two (`platform.md` § Entities and jobs). Old entities stay readable and nothing else:
`distil seed-dataset {list,show,download,download-metadata}` and the same for
`distil training-dataset` work; every other command of theirs, and `distil traces
expand-test-set`, prints the replacement and exits 1.

To keep working from an old entity, move its files into a Dataset:

1. Download it: `distil seed-dataset download -d old <id>` or
   `distil training-dataset download -d old <id>` (free). For an old PreparedTraces,
   `distil traces download -d old <id>` gives its `traces.jsonl` only; the config, job
   description and test set it was uploaded with are not downloadable, so take them from your
   own files.
2. Fix what the new format rejects:
   - **Flat rows** (`{"question": ..., "answer": ...}`): rewrite every row as a `messages`
     conversation (`data-preparation/overview.md` § Row format).
   - **`unstructured.jsonl`**: if trace processing wrote it (each row is one serialised trace),
     rename it to `traces.jsonl` and set `trace_processing.observation_format:
     unstructured_with_openai_messages` in the config. If it holds uploaded documents, leave it
     out: a Dataset has no place for context documents.
   - **`final-synthetic-dataset/`**: move its `train.jsonl` and `test.jsonl` to the top level.
   - **Removed task types** (`question-answering-open-book`, `question-answering-closed-book`,
     `information-extraction`, `question-answering-open-book-synthetic-context`,
     `question-answering-legacy`): set `base.task: question-answering` and drop `context` from
     the rows (`task-types.md`).
   - **A QA job description** with text in `input_description` or `context_description`:
     remove the two keys.
   - The config's old keys load as they are and move to their new names
     (`configuration.md`); nothing to edit.
3. Upload the directory: `distil dataset create --data old` (`execution/cli.md` § The Dataset).
   The create validates it and names the first row that fails.

The new Dataset has no parent: it is the root of a new chain. A test set from an old entity is
a locked test set like any other (`../stages/build-a-test-set.md` Step 5), and a train set from
an old TrainingDataset is a train set to top up (`../stages/build-a-train-set.md` § Topping Up).
