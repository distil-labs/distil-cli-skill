# Stage: Synthetic Data Generation

The teacher generates the training dataset from the seed data, the job description and the
mutators. Validation and dedup then produce the TrainingDataset that model training consumes:
its root `train.jsonl` is the seed rows and the surviving synthetic rows, merged. Conversations
are expanded per assistant turn according to `base.should_expand_dataset`
(`../references/configuration.md` § Conversation expansion).

## Working Directory

```
synthetic-data-generation/
├── smoke-1/
│   ├── input/           # the override config and job description sent, if any
│   ├── run.md           # the SeedDataset id, the command and the TrainingDataset id
│   └── output/          # fetched sample, metrics and logs
├── smoke-2/             # next smoke: one change against smoke-1
└── full-1/              # the last passing smoke's settings, without --smoke
```

## Step 1: Prepare the Input

The input is the SeedDataset that passed teacher evaluation, by id. Changes to it are a config
or job-description override (`../references/execution/cli.md` § Overrides), kept in `input/`.
For a reasoning student, see `../references/reasoning-models.md`.

With an empty train split the stage still runs, zero-shot, and its output becomes the train
split (`../references/data-preparation/overview.md` § Empty splits). Tell the user: no seed rows
fix the format or style, and Step 4's distribution check has nothing to compare against.

What shapes the generated data:

- `task_description` defines what a correct answer is and is fixed
  (`../references/job-description.md` § What each field feeds).
- `synthetic_data_generation_instructions` and mutators are the two settings that control the
  data; § What Can Be Changed to Improve the Next Iteration describes both. If the task already
  names patterns, domains or proportions to cover, write them as mutators now. If a dimension
  clearly matters but its values are not obvious, give the mutator a `description` and no
  `values`. Otherwise start without mutators and revisit after the smoke.
- The `synthgen` config (`../references/configuration.md`). Set deliberately:
  `validation_max_total_length` to the longest combined row length expected; lower
  `generation_in_single_call` and the exemplar counts when rows are long; `output_is_json: true`
  when answers must be JSON (question answering only).

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- the key `synthgen` config and the intended `generation_target`
- the mutator plan: dimensions and values, values left for the teacher to detect, or none yet
- the remaining `training_datasets_from_seed_datasets_smoke_post` credits, one per smoke, and
  `training_datasets_from_seed_datasets_post` credits, one per full run
  (`../references/platform.md` § Credits)
- the path: normal (a smoke, its analysis, then the full run) or fast (skip Steps 3-5)

## Step 3: Smoke Run

Submit with `--smoke` (`../references/execution/cli.md` § Synthetic data generation; what a
smoke generates: `../references/platform.md` § Smoke runs). Record the command and the
TrainingDataset id in `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded, then read the data into `output/` (`../references/execution/cli.md`
§ Output layout and § Reading a TrainingDataset). The synthetic count is the root row count
minus the seed count. Falling well short of the target means validation filtered heavily,
usually on length or format.

The free sample mixes seed and generated rows and does not mark which is which
(`../references/platform.md` § What each stage produces). Compare it against the seed rows to
tell them apart, and read the axes below on the rows that are not in the seed. The sample can
hold few or none of the generated rows, notably when the seed is large (reused training data).
When it doesn't cover them, use the full `training-dataset download` instead.

Analyze the synthetic rows on three axes:

1. **Form against the job description**: each row parses, carries the required output format
   and respects the stated constraints. A malformed row is a defect at any rate. Read a handful
   of answers to check the teacher understood the task. Scattered label errors are noise the
   student tolerates; one rule broken again and again is a defect to fix
   (§ What Can Be Changed to Improve the Next Iteration).
2. **Distribution against the seed rows**: length (characters, or turns), topic coverage, style
   and register, class balance. Pick the dimensions that matter for the task and count.
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

Submit the last passing smoke's settings without `--smoke`, with `generation_target: 10000`
unless the user asks for a different size.

## Step 7: Analyze the Results

Repeat the Step 4 analysis on a sample of the root `train.jsonl`, and check the final size
against the target. `generation_target` rounds up to the next `generation_iteration_size`
batch, so landing above it is normal.

Present the findings to the user:

- **It passes**: the dataset goes to `model-training.md`.
- **It fails**: do not train on it. Bad data shows up in training only as a plateau found after
  a full training run. Change the settings per Step 5 and submit another full run, which costs
  one generation credit instead of a training credit and a full training run.

## What Can Be Changed to Improve the Next Iteration

- **The seed**: the last iteration's training data, unless it was bad, with `generation_target`
  set to the rows to add (`../workflows/model-iterations.md` § Step 4: Set Up the Next Iteration).
- **Wrong or malformed generated rows** (the teacher mislabels one slice, or breaks a rule or
  the format): either describe the slice and its correct handling in more detail in
  `synthetic_data_generation_instructions`, or use a stronger teacher
  (`../references/model-catalog.md`), and generate again; or download the training data,
  correct the rows, and upload the directory as a new TrainingDataset with
  `distil training-dataset create --data <dir>` (`../references/execution/cli.md` § Supplying
  files; one `training_datasets_download_get` and one `training_datasets_post` credit). In
  classification, also check whether a mutator causes it (`../references/mutators.md`
  § Mutators for classification).
- **`synthetic_data_generation_instructions`** in the job description: a constant added to every
  generation call that describes the inputs to write (formats, domains, register, noise). It
  moves every generated row the same way, so it fits when the whole training set is off in the
  same direction, without changing what a correct answer is.
- **Mutators** (`synthgen.mutators`, `../references/mutators.md`): the dimensions the data is
  spread over, one value sampled per generation call, weighted when the mix matters. They shape
  the distribution rather than every row, so they fit coverage: slices that are missing, rare,
  or over-represented against production. For chat completion with tools, a mutator over the
  tools spreads the data across them (`../references/data-preparation/chat-completion.md`
  § Synthetic data with tools).
- **The `synthgen` config**: how much is generated (`generation_target`, or a top-up on the last
  dataset, `../workflows/model-iterations.md` § Step 4: Set Up the Next Iteration), how much
  survives validation (`validation_max_total_length`, `validation_similarity_threshold`), how
  much each call writes (`generation_in_single_call`, the exemplar counts), how much the teacher
  varies (`teacher_temperature`), and whether a final pass repairs the training targets
  (`clean_training_targets`). These change the volume and cleanliness of the data, not what it
  is about.

All are overrides on the same SeedDataset. Cost: one `training_datasets_from_seed_datasets_post`
credit per full run, and training again on the new dataset.
