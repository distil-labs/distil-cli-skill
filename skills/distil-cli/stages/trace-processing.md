# Stage: Trace Processing

Turns a traces object into a SeedDataset: traces are filtered for relevance, relabelled by a
teacher (optionally a committee), and become the train split, with the leftover traces as
unstructured context. The test split is not built here. It is copied from the PreparedTraces the
job runs over, so run `test-set-from-traces.md` first, or upload a curated `test.jsonl` with the
traces.

## Working Directory

```
trace-processing/
├── smoke-1/
│   ├── input/           # the subsample upload directory, and any override sent
│   ├── run.md           # the PreparedTraces id, the command and the SeedDataset id
│   └── output/          # fetched processed data
├── smoke-2/             # next smoke: one change against smoke-1
└── full-1/              # the last passing smoke's settings over the full trace set
```

## Step 1: Prepare the Input

The input is a PreparedTraces id. For the full run it is the updated PreparedTraces from
`test-set-from-traces.md`, which holds the test set and the traces it did not use. The trace
file, the task type and the job description are prepared before that
(`../workflows/build-a-model.md` Step 3, `../references/data-preparation/traces.md`). Without a
test set the SeedDataset has an empty test split, and teacher evaluation cannot run on it.

The `trace_processing` config section controls this stage; its parameters and defaults are in
`../references/configuration.md` § trace_processing. Test set from traces reads the same section
for its relabelling, so a change here changes both jobs. `observation_format` must match the
shape of `traces.jsonl`. Never set `relabel: false`: the production answers would pass through
unreviewed, and the student would only learn to imitate the model it is meant to beat. When
relabelled answers look worse than the originals, change the relabelling teacher or committee
instead.

Two behaviours to expect:

- **No floor on the train split.** Filtering, relabelling and schema checks each drop traces,
  and whatever survives is written, an empty split included. A train split far below
  `num_traces_as_training_base` usually means the traces do not match the task type, for example
  multi-turn traces for a single-turn task. The job log counts each drop.
- **The exemplar counts can come out lower.** No in-context exemplar count may exceed the train
  rows (`../references/configuration.md` § Cross-field validation), so this stage caps all four
  to the train split it produced and logs a warning. Read the counts from the processed config.

For a reasoning student, see `../references/reasoning-models.md`.

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- what the traces look like (count, observation format, typical length) and the task type
- where the test set comes from: the test set from traces run, or an uploaded `test.jsonl`
- the processing config (relabelling teacher or committee, relevance filtering,
  `num_traces_as_training_base`)
- the remaining `prepared_traces_post` and `seed_datasets_from_prepared_traces_post` credits:
  the smoke spends one of each, the full run one `seed_datasets_from_prepared_traces_post`
  (`../references/platform.md` § Credits)
- the path: normal (a smoke on a subsample, its analysis, then the full run) or fast (skip
  Steps 3-5)

## Step 3: Smoke Run

A smoke runs on a trimmed copy of the traces, uploaded as its own PreparedTraces:

1. Get a local `traces.jsonl`: the trace file itself, or, for traces created directly from an
   inference endpoint, the endpoint's records downloaded and converted
   (`../references/data-preparation/traces.md` § From an inference endpoint).
2. Trim it to 50-100 lines, and set `trace_processing.num_traces_as_training_base: 10` in a copy
   of `config.yaml`.
3. Upload the trimmed directory without `test.jsonl` (the smoke checks the train split only),
   and run trace processing on the upload
   (`../references/execution/cli.md` § Test set from traces and trace processing).

Record the ids in `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded, then download the SeedDataset into `output/`
(`../references/execution/cli.md` § Output layout). Check first how many traces reached
`train.jsonl`. Near-total loss usually means the relevance filter and the job description
disagree about what the task is.

Then analyze the processed rows on two axes:

1. **Correctness against the job description**: valid task rows in the right format, and
   relabelled answers that are correct and better than the originals where they differ.
2. **Distribution against the traces**: length (characters, or turns), topic coverage, style.
   Watch for filtering that dropped whole categories.

## Step 5: Iterate Until the Smoke Passes

If an axis fails, change one setting in a new smoke: relevance filtering when irrelevant or
incoherent traces reach the output, the relabelling teacher or committee when relabelled answers
are worse than the originals, `trace_processing_instructions` when the rewrites mishandle a
task-specific quirk. Each is an override on the smoke's PreparedTraces, not a new upload. Move
on once a smoke passes both axes.

## Step 6: Full Run

Submit the last passing smoke's config, with the intended `num_traces_as_training_base`
(default 200), as an override on the PreparedTraces from `test-set-from-traces.md`. Every later
run is an override on the same PreparedTraces.

## Step 7: Analyze the Results

Repeat the Step 4 analysis on the full output. Check the train row count and the four exemplar
counts in the processed config, and that `test.jsonl` is the test set the user approved. The
SeedDataset is the input of `teacher-evaluation.md` and `synthetic-data-generation.md`.

## What Can Be Changed to Improve the Next Iteration

- **The relabelling teacher and committee** (`trace_processing.teacher_model_name`,
  `trace_processing.relabelling_committee_models`): the models that rewrite the production
  answers into the seed rows. Synthetic data generation imitates the seed rows, so better
  relabelling raises the quality of the whole training set.
- **`trace_processing_instructions`** in the job description: guidance for the rewrite only. It
  tells the relabelling about quirks of the traces without changing what a correct answer is.
- **Relevance filtering** (`relevance_filtering`, `min_relevance_score`,
  `min_coherence_score`): which traces may become seed rows. It trades seed volume for seed
  cleanliness.
- **`num_traces_as_training_base`**: how many traces become seed rows. More seed rows give
  generation more real examples to imitate.
- **`compress_job_description`**: shortens a long task description for the filtering model.
- **`synthgen.validation_max_total_length`**: how long a processed row may be. Raise it when the
  traces embed documents or schemas.

All are overrides on the same PreparedTraces. Cost: one `seed_datasets_from_prepared_traces_post`
credit, and synthetic data and training again on the new SeedDataset.
