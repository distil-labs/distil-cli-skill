# Stage: Test Set from Traces

Takes a traces object (a PreparedTraces, uploaded or created from an inference endpoint) and
builds an updated traces object (a new PreparedTraces) that carries a test set made from the
traces. The job relabels some traces into test rows, optionally generates synthetic test rows
from them, and optionally scores the original model on the relabelled traces; that score is the
baseline the trained student must beat. The traces it uses are removed from the updated trace
set, so trace processing over it cannot put the same trace in train and test.

## Working Directory

```
test-set-from-traces/
├── iteration-1/
│   ├── input/           # the override config and job description sent, if any
│   ├── run.md           # the parent PreparedTraces id, the command, the updated PreparedTraces id
│   └── output/          # fetched traces directory, original-model metrics and predictions
└── iteration-2/         # next attempt: one change against iteration-1
```

## Step 1: Prepare the Input

The input is a PreparedTraces id, from an upload or from an inference endpoint
(`../references/execution/cli.md` § Test set from traces and trace processing, § Traces from an
endpoint). Its optional `test.jsonl` of curated rows stays in the test set, and the job adds to
it.

The `traces_to_test_set` config section controls the job; its parameters and defaults are in
`../references/configuration.md` § traces_to_test_set. How a trace is relabelled comes from the
`trace_processing` section, shared with trace processing. Set `evaluate_original_model: true` on
the run whose test set will be kept: the original model's score is the baseline every later
verdict is read against. Keep `min_relabelled_examples` at its default 0; `num_synthetic_examples`
is a floor, since generation runs in batches.

Count the traces for both jobs: this job removes `num_traces_to_relabel +
2 * num_synthetic_examples` traces, and both jobs refuse a trace set smaller than
`trace_processing.num_traces_as_training_base`: they parse the trace file with the same
validation, so this job is refused below it too. Size the trace set for it, or lower the setting. For a reasoning student, see
`../references/reasoning-models.md`.

## Step 2: Confirm the Setup with the User

Present and confirm: the trace count and how the two jobs split it, `num_traces_to_relabel`,
`num_synthetic_examples`, `evaluate_original_model`, the relabelling teacher, and the remaining
`prepared_traces_with_expanded_test_set_post` credits, one per run
(`../references/platform.md` § Credits). This stage has no smoke run.

## Step 3: Run

Submit the job (`../references/execution/cli.md` § Test set from traces and trace processing),
with any change in `input/` sent as an override. Record the parent id, the command and the
updated PreparedTraces id in `run.md`, and poll that id to `JOB_SUCCESS`.

## Step 4: Review with the User

This review is a hard gate: nothing goes to trace processing without the user's approval.
Download the updated PreparedTraces into `output/` and read its `test.jsonl`: the uploaded test
rows, then the relabelled traces, then the synthetic rows. Fetch the original-model metrics and
predictions (`../references/execution/cli.md` § Fetch metrics). Review with the user:

- **Size**: the row count against `num_traces_to_relabel` and `num_synthetic_examples`. A large
  loss means filtering or schema checks dropped traces; the job log has the cause.
- **Correctness**: relabelled answers that are correct against the job description, and better
  than the original answers where they differ.
- **Coverage**: the test rows against the traces on the dimensions that matter for the task
  (topic, length, class balance). Synthetic rows measure agreement with the teacher, a weaker
  signal than relabelled traces, so read them with more care.
- **Baseline**: the original model's score on the primary metric (`../references/evaluation-metrics.md`
  § Primary metric per task), on the rows this run relabelled only (not rows carried over from a
  parent traces object), and a handful of its predictions marked wrong.

Drop or fix what fails. To change the test set, run this stage again with different settings,
or edit the downloaded `test.jsonl` and upload the directory as a new PreparedTraces, which goes
to trace processing directly. An edited test set has no original-model baseline: the job scores
the original model only on traces it relabels in the same run, and the earlier score was
measured on rows that have changed.

## Step 5: Pass to Trace Processing

The updated PreparedTraces id is the input of `trace-processing.md`, and its test set is the
project's test set from here on. Scores measured on different test sets are not comparable, so
when this stage runs again in a later iteration, re-run teacher evaluation on the new test set
before reading any verdict.

## What Can Be Changed to Improve the Next Iteration

- **`traces_to_test_set.num_traces_to_relabel`**: how many traces become test rows. A larger test
  set narrows the noise band, so smaller differences between iterations become readable, and
  covers more of the production distribution.
- **`traces_to_test_set.num_synthetic_examples`**: test rows the teacher generates on top of the
  relabelled ones. It grows the test set beyond what the traces hold, with rows that measure
  agreement with the teacher rather than with production.
- **The relabelling settings** (`trace_processing.teacher_model_name`,
  `trace_processing.relabelling_committee_models`, `trace_processing_instructions`): who writes
  the reference answers and with what guidance. The reference answers define what every score
  measures.
- **New traces**: a test set can only measure what the traces contain. Traces that show a
  failure the user cares about, run through this stage, add it to the test set.

All except new traces are overrides on the same PreparedTraces. A new test set makes every
earlier score stale, so it needs the user's approval. Cost: one
`prepared_traces_with_expanded_test_set_post` credit per run.
