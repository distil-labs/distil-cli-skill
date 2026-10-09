# Stage: Synthetic Data Generation

The teacher generates rows for one split from the rows already in it, the job description, the
mutators and the traces left in the Dataset as context. Validation and dedup keep the rows that
pass, and the job writes a new Dataset with them added to the split and the context traces
removed (`../references/platform.md` § The expand operations). The same mechanics serve two
outcomes: `build-a-test-set.md` runs it on the test split, `build-a-train-set.md` on the train
split. This file is the procedure for one run; the outcome stages say how many rows to aim for.

## Working Directory

```
synthetic-data-generation-<split>/
├── smoke-1/
│   ├── input/           # the override config and job description sent, if any
│   ├── run.md           # the parent Dataset id, the command and the new Dataset id
│   └── output/          # the new Dataset, downloaded, and the parent's files for the diff
├── smoke-2/             # next smoke: one change against smoke-1
├── full-1/              # the last passing smoke's settings, without --smoke
└── top-up-1/            # a later run on another parent (build-a-train-set.md § Topping Up)
```

## Step 1: Prepare the Input

The input is a Dataset id. Changes to it are a config or job-description override
(`../references/execution/cli.md` § Overrides), kept in `input/`. For a reasoning student, see
`../references/reasoning-models.md`.

What the job reads from the Dataset:

- **The split's rows are the examples.** With an empty split the job runs zero-shot, and the
  in-context exemplar counts are capped to the rows there are. Tell the user when the split is
  empty: no rows fix the format or style, and Step 4's distribution check has nothing to compare
  against. A train split that an earlier run expanded into per-turn rows (`clean_training_targets`
  or `enable_thinking` on a multi-turn task) is refused as examples: generate from the Dataset
  before that run.
- **The other split is held out**: new rows that duplicate it are dropped, so generating into
  test never leaks train rows and the reverse.
- **The traces are the context** when `synthgen.use_traces_as_context` is true: `T +
  min(T, 1000)` of them for a target of T, or all that are left, and they leave the Dataset.
  Below `min(T / 4, 10)` traces the job generates without context and leaves the traces alone.
  Count the traces before deciding the target, because this run can empty the pool for every
  expand after it.

What shapes the generated rows:

- `task_description` defines what a correct answer is and is fixed
  (`../references/job-description.md` § What each field feeds).
- `synthetic_data_generation_instructions` and mutators are the two settings that control the
  rows; § What Can Be Changed to Improve the Next Iteration describes both. If the task already
  names patterns, domains or proportions to cover, write them as mutators now. If a dimension
  clearly matters but its values are not obvious, give the mutator a `description` and no
  `values`. Otherwise start without mutators and revisit after the smoke.
- The `synthgen` config (`../references/configuration.md` § synthgen). The target is
  `train_generation_target` or `test_generation_target` for the split. Set deliberately:
  `validation_max_total_length` to the longest combined row length expected; lower
  `generation_in_single_call` and the exemplar counts when rows are long; `output_is_json: true`
  when answers must be JSON (question answering only).

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- the split, the key `synthgen` config and the intended target
- the examples (row count of the split) and the context (trace count, and what is left after)
- the mutator plan: dimensions and values, values left for the teacher to detect, or none yet
- the remaining credits: `datasets_from_datasets_smoke_post` per smoke,
  `datasets_from_datasets_generate_synthetic_data_post` per full run
  (`../references/platform.md` § Credits)
- the path: normal (a smoke, its analysis, then the full run) or fast (skip Steps 3-5)

## Step 3: Smoke Run

Submit with `--smoke` (`../references/execution/cli.md` § Synthetic data generation), which
generates 128 rows whatever the target says. The smoke's Dataset is a side branch: the full run
goes on the same parent (`../references/platform.md` § Smoke runs). Record the command and the
new Dataset id in `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded, then download the new Dataset into `output/`
(`../references/execution/cli.md` § Reading a Dataset). The new rows are the lines of the split
after the parent's rows; diff against the parent's file to isolate them, since nothing in a row
says it was generated. Falling well short of 128 means validation filtered heavily, usually on
length or format; the job log counts the drops.

Analyze the new rows on three axes:

1. **Form against the job description**: each row parses, carries the required output format
   and respects the stated constraints. A malformed row is a defect at any rate. Read a handful
   of answers to check the teacher understood the task. Scattered label errors are noise the
   student tolerates; one rule broken again and again is a defect to fix
   (§ What Can Be Changed to Improve the Next Iteration).
2. **Distribution against the examples and the traces**: length (characters, or turns), topic
   coverage, style and register, class balance. Pick the dimensions that matter for the task and
   count. With no examples, compare against the traces.
3. **Requested slices present**: count each slice the mutators or the job description asked
   for. Mutator values are requests and can produce nothing. Gate on this axis only when the
   mutator grid has at most 16 cells; with more, check it on the full run.

For a reasoning student, also check the reasoning (`../references/reasoning-models.md`).

## Step 5: Iterate Until the Smoke Passes

If an axis fails, change one setting in a new smoke: mutators for the mix of rows, the
generation instructions for how the inputs look, the `synthgen` config for volume and
validation. Present each smoke analysis to the user, and move on once a smoke passes every axis
and the user agrees to the full run.

## Step 6: Full Run

Submit the last passing smoke's settings without `--smoke`, with the target the outcome stage
set, as an override on the parent Dataset. Record the ids in `run.md`.

## Step 7: Analyze the Results

Repeat the Step 4 analysis on a sample of the new rows, and check the final size against the
target. The target rounds up to the next `generation_iteration_size` batch, so landing above it
is normal. Present the findings to the user:

- **It passes**: the new Dataset goes to the outcome stage's next step.
- **It fails**: do not build on it. Bad data shows up in training only as a plateau found after
  a full training run. Change the settings per Step 5 and submit another full run on the same
  parent, which costs one generation credit instead of a training credit and a full training
  run.

## What Can Be Changed to Improve the Next Iteration

- **The examples**: the rows in the split when the job runs. A top-up on the best kept iteration's Dataset
  generates from everything in it, relabelled and synthetic rows alike
  (`build-a-train-set.md` § Topping Up); a run on an earlier Dataset in the chain generates from
  less.
- **Wrong or malformed generated rows** (the teacher mislabels one slice, or breaks a rule or
  the format): either describe the slice and its correct handling in more detail in
  `synthetic_data_generation_instructions`, or use a stronger teacher
  (`../references/model-catalog.md`), and generate again; or download the Dataset, correct the
  rows, and upload the directory as a new Dataset with `distil dataset create --data <dir>`
  (`../references/execution/cli.md` § Supplying files; one `datasets_post` credit). In
  classification, also check whether a mutator causes it (`../references/mutators.md`
  § Mutators for classification).
- **`synthetic_data_generation_instructions`** in the job description: a constant added to every
  generation call that describes the inputs to write (formats, domains, register, noise). It
  moves every generated row the same way, so it fits when the whole set is off in the same
  direction, without changing what a correct answer is.
- **Mutators** (`synthgen.mutators`, `../references/mutators.md`): the dimensions the rows are
  spread over, one value sampled per generation call, weighted when the mix matters. They shape
  the distribution rather than every row, so they fit coverage: slices that are missing, rare,
  or over-represented against production. For chat completion with tools, a mutator over the
  tools spreads the rows across them (`../references/data-preparation/chat-completion.md`
  § Synthetic data with tools).
- **The `synthgen` config**: how much is generated (`train_generation_target`,
  `test_generation_target`), whether the traces serve as context (`use_traces_as_context`), how
  much survives validation (`validation_max_total_length`, `validation_similarity_threshold`),
  how much each call writes (`generation_in_single_call`, the exemplar counts), how much the
  teacher varies (`teacher_temperature`), and whether a final pass repairs the training targets
  (`clean_training_targets`). These change the volume and cleanliness of the rows, not what
  they are about.

All are overrides on the same parent Dataset. Cost: one
`datasets_from_datasets_generate_synthetic_data_post` credit per full run, the context traces
it uses, and training again on the new Dataset.
