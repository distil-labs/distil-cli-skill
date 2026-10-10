# Stage: Build a Train Set

Fills the Dataset's `train.jsonl` with the rows the student learns from, using the teacher that
passed `teacher-evaluation.md`. The train set is real rows first and synthetic rows second:

1. **Relabelled traces** (`relabel-traces.md` on the train split): production inputs with
   answers rewritten by the teacher. Generation imitates these, so they set the format, the
   style and the hard cases.
2. **Synthetic rows** (`synthetic-data-generation.md` on the train split): the bulk of the
   set, generated from the rows in the split, the job description, the mutators and the traces
   left as context. The default target is 10,000.

Both runs leave `test.jsonl` unchanged, so the model trained on the result is scored on the
locked test set (`build-a-test-set.md`).

## Working Directory

The mechanics stages own their directories (`relabel-traces-train/`,
`synthetic-data-generation-train/`). This stage keeps `train-set.md` in the project root: the
plan, the Dataset ids of each run, and the analysis of the final set.

## Step 1: Take Stock

Download the test-set Dataset (`../references/execution/cli.md` § Reading a Dataset) and
record:

- the rows already in `train.jsonl` (an uploaded train set stays in and is added to);
- the distinct traces left in `traces.jsonl` after the test set took its share;
- the teacher that passed teacher evaluation. It goes into both `base.teacher_model_name` and
  `trace_processing.teacher_model_name` of the override, because the test-set Dataset's config
  is expanded and names the relabelling teacher on its own
  (`../references/model-catalog.md` § Defaults).

## Step 2: Plan and Confirm with the User

Decide the two counts:

- `trace_processing.num_train_relabelled`: how many of the remaining traces become real train
  rows. Default 200; more when the pool is large. Leave traces for context: generation wants
  `T + min(T, 1000)` of them for a target of T, takes what is left otherwise, and logs a
  warning and generates without context below `min(T / 4, 10)` (`../references/platform.md` § The expand operations).
  With a few hundred traces left, relabelling most of them and generating with little context
  is the better trade: a relabelled row is worth more than a context trace.
- `synthgen.train_generation_target`: the synthetic rows to add. 10,000 unless the user asks
  for a different size.

Present both, the teacher, the mutator plan (`synthetic-data-generation.md` Step 1) and the
credits: one `datasets_from_datasets_relabel_traces_post`, one
`datasets_from_datasets_generate_synthetic_data_post`, the smoke credits, and the
`slms_from_datasets_post` credit training will need after this
(`../references/platform.md` § Credits). Generation is wasted if no training credit is left.

## Step 3: Relabel Traces into the Train Split

Run `relabel-traces.md` on the train split of the test-set Dataset with `num_train_relabelled`
from the plan. Skip it when no traces are left, or when the user uploaded a train set and has no
traces. Record the new Dataset id in `train-set.md`.

## Step 4: Generate Synthetic Rows

Run `synthetic-data-generation.md` on the train split of the Dataset Step 3 produced (or the
test-set Dataset when Step 3 was skipped), smoke first, then the full run with
`train_generation_target` from the plan. Record the new Dataset id in `train-set.md`.

## Step 5: Analyze the Train Set

Download the final Dataset and repeat the analysis of `synthetic-data-generation.md` Step 7 on
the whole train split: form against the job description, distribution against the relabelled
rows and the traces, and the requested slices. Check the size against the plan.

- **It passes**: the Dataset goes to `model-training.md`.
- **It fails**: do not train on it. Fix the mechanics stage that produced the bad rows and run
  it again on its parent.

## Topping Up

A later iteration that needs more rows (`../workflows/model-iterations.md` Step 4) does not
rebuild: run `synthetic-data-generation.md` on the train split of the best kept iteration's
Dataset, with `train_generation_target` set to the rows to add. The job generates from every
row already in the split, relabelled and synthetic alike, and dedups against them
(`../references/platform.md` § The expand operations), so the output is old and new merged. Any
traces left serve as context; after a full run there are usually none, and the top-up
generates without context. A top-up may take the fast path when its settings passed a smoke
before.

One exception: a train split that generation expanded into per-turn rows
(`clean_training_targets` or `enable_thinking` on a multi-turn task) cannot be the examples of
a later run. Top up on the Dataset before that run instead, with the full target.

## What Can Be Changed to Improve the Next Iteration

- **The teacher**: a different teacher is checked by `teacher-evaluation.md` first, then Step 3
  and Step 4 run again on the test-set Dataset.
- **The relabelled rows** (`relabel-traces.md` § What Can Be Changed to Improve the Next
  Iteration): who rewrites them, with
  what guidance, how many. Generation imitates them, so better relabelling raises the whole
  set.
- **The synthetic rows** (`synthetic-data-generation.md` § What Can Be Changed to Improve the
  Next Iteration): instructions,
  mutators, the `synthgen` config, or a top-up.
- **Hand-corrected rows**: download the Dataset, fix `train.jsonl`, upload the directory as a
  new Dataset (`../references/execution/cli.md` § Supplying files).

Cheapest first: a top-up or a hand correction (one credit), then generation again, then
relabelling again (and generation after it), then a new teacher (teacher evaluation, then
both).
