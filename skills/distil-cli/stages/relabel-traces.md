# Stage: Relabel Traces

Turns traces into rows of one split. The job takes `trace_processing.num_{split}_relabelled`
traces from the Dataset, filters them for relevance, has a teacher (optionally a committee)
rewrite the production answers, validates the rows and writes a new Dataset with the rows added
to the split and the used traces removed (`../references/platform.md` § The expand operations).
The same mechanics serve two outcomes: `build-a-test-set.md` runs it on the test split,
`build-a-train-set.md` on the train split. This file is the procedure for one run; the outcome
stages say how many rows to aim for and what to do with them.

The `trace_processing` config section controls this stage; its parameters and defaults are in
`../references/configuration.md` § trace_processing. Never set `relabel: false`: the production
answers would pass through unreviewed, and the student would only learn to imitate the model it
is meant to beat. When relabelled answers look worse than the originals, change the relabelling
teacher or committee instead. The job refuses `base.enable_thinking: true`: relabel with it off,
and turn it on for synthetic data generation (`../references/reasoning-models.md`).

## Working Directory

```
relabel-traces-<split>/
├── smoke-1/
│   ├── input/           # the override config and job description sent, if any
│   ├── run.md           # the parent Dataset id, the command and the new Dataset id
│   └── output/          # the new Dataset, downloaded, and the parent's files for the diff
├── smoke-2/             # next smoke: one change against smoke-1
└── full-1/              # the last passing smoke's settings with the intended count
```

## Step 1: Prepare the Input

The input is a Dataset id with traces in it. Download it (`../references/execution/cli.md`
§ Reading a Dataset) and count the distinct lines of `traces.jsonl`: the job refuses to run when
the count is below `num_{split}_relabelled`, and every trace it uses is gone for the expands
after it. Decide the count with the outcome stage's budget, and never set it above what the
other expands can spare.

Set the relabelling deliberately:

- `trace_processing.teacher_model_name` (inherits `base.teacher_model_name`) and
  `relabelling_committee_models`: who rewrites the answers. The reference answers of a test set
  define what every score measures; the default large teacher writes them unless the user asks
  for a stronger one (`../references/model-catalog.md` § Defaults). In an expanded config the
  field is set on its own, so a new teacher goes into both fields.
- `trace_processing_instructions` in the job description: guidance for the rewrite only,
  without changing what a correct answer is (`../references/job-description.md`).
- `relevance_filtering`, `min_relevance_score`, `min_coherence_score`: which traces may become
  rows. Off by default.
- `observation_format`: must match the shape of `traces.jsonl`
  (`../references/data-preparation/traces.md`).

Two behaviours to expect:

- **No floor on the rows.** Filtering, relabelling, validation and the overlap check against
  the other split each drop traces, and whatever survives is written, zero rows included. A
  count far below the traces used usually means the traces do not match the task type, for
  example multi-turn traces for a single-turn task. The job log counts each drop.
- **The traces are shuffled and deduplicated first**, so two runs on the same parent pick
  different traces, and duplicates in `traces.jsonl` do not count.

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- what the traces look like (count, observation format, typical length) and the task type
- the split, the count, and what the trace pool holds after this run
- the relabelling teacher or committee, the instructions, and relevance filtering
- the remaining credits: `datasets_from_datasets_smoke_post` for the smoke,
  `datasets_from_datasets_relabel_traces_post` for the full run
  (`../references/platform.md` § Credits)
- the path: normal (a smoke of 128 traces, its analysis, then the full run) or fast (skip
  Steps 3-5; the usual choice when the count is near 128 anyway, and the only one below 128
  traces, where the smoke is refused like any run short of traces)

## Step 3: Smoke Run

Submit with `--smoke` (`../references/execution/cli.md` § Relabel traces), which relabels 128
traces of the parent whatever the config says. The smoke's Dataset is a side branch: the full
run goes on the same parent (`../references/platform.md` § Smoke runs). Record the parent id,
the command and the new Dataset id in `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded, then download the new Dataset into `output/`
(`../references/execution/cli.md` § Reading a Dataset). The new rows are the lines of the split
after the parent's rows; diff against the parent's file to isolate them. Check first how many
of the 128 traces reached the split. Near-total loss usually means the relevance filter and the
job description disagree about what the task is.

Then analyze the new rows on two axes:

1. **Correctness against the job description**: valid task rows in the right format, and
   relabelled answers that are correct and better than the originals where they differ. Read
   the trace and its row side by side for a handful.
2. **Distribution against the traces**: length (characters, or turns), topic coverage, style.
   Watch for filtering that dropped whole categories.

## Step 5: Iterate Until the Smoke Passes

If an axis fails, change one setting in a new smoke: relevance filtering when irrelevant or
incoherent traces reach the output, the relabelling teacher or committee when relabelled answers
are worse than the originals, `trace_processing_instructions` when the rewrites mishandle a
task-specific quirk. Each is an override on the same parent Dataset
(`../references/execution/cli.md` § Overrides), not a new upload. Move on once a smoke passes
both axes.

## Step 6: Full Run

Submit the last passing smoke's config, with the intended `num_{split}_relabelled`, as an
override on the parent Dataset, without `--smoke`. Record the ids in `run.md`.

## Step 7: Analyze the Results

Repeat the Step 4 analysis on the full output, and read the row count and the remaining trace
count from the job log. The new Dataset is the parent of the next expand; the outcome stage
decides what that is.

## What Can Be Changed to Improve the Next Iteration

- **The relabelling teacher and committee** (`trace_processing.teacher_model_name`,
  `trace_processing.relabelling_committee_models`): the models that rewrite the production
  answers. On the test split they write the reference answers, so they decide what every score
  measures; on the train split, synthetic data generation imitates their rows, so better
  relabelling raises the quality of the whole training set.
- **`trace_processing_instructions`** in the job description: guidance for the rewrite only. It
  tells the relabelling about quirks of the traces without changing what a correct answer is.
- **Relevance filtering** (`relevance_filtering`, `min_relevance_score`,
  `min_coherence_score`): which traces may become rows. It trades volume for cleanliness.
- **`num_{split}_relabelled`**: how many traces become rows. More test rows narrow the noise
  band and cover more of production; more train rows give generation more real examples to
  imitate. Both spend traces the other split cannot use.
- **`compress_job_description`**: shortens a long task description for the filtering model.
- **`synthgen.validation_max_total_length`**: how long a row may be. Raise it when the traces
  embed documents or schemas.
- **New traces**: relabelling can only produce what the traces contain. Traces that show a
  failure the user cares about, run through this stage, add it to the split.

All except new traces are overrides on the same parent Dataset. Cost: one
`datasets_from_datasets_relabel_traces_post` credit, the traces it uses, and every expand after
it again.
