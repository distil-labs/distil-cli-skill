# Stage: Model Training

Finetunes the student on the synthetic dataset and evaluates base and tuned student on the
test set. That yields the tuned-vs-teacher-vs-base comparison and the deployment verdict.
Training is the multi-hour GPU stage, so how much you experiment here is a budget decision
the user makes, not you.

Both splits of the dataset gate this stage, in opposite ways
(`../references/data-preparation/overview.md` § Empty splits):

- **An empty train split stops the run.** Synthetic data generation is what fills it, so run
  that first. The refusal is deliberate, and it names itself:

  ```
  Finetuning needs a training set, and this job has none: train.jsonl is empty.
  Generate synthetic data first, or provide training examples.
  ```

- **An empty test split does not stop the run.** Training completes and produces a model, but
  there is no evaluation at all: no base-vs-tuned comparison, no metric values, and no
  evaluation output directories. Both quantization sweeps are skipped, so neither `eval/` nor
  `eval-4bit/` is written, and there is no `metrics-eval-aggregated.json` to read. The metrics
  are recorded as "not available" rather than as a score, so the run is not mistaken for one
  whose scoring failed. Do not read a missing metrics file as a job failure. Check the test
  split first.

  Step 7's analysis rests entirely on those numbers, so a run with no test set cannot produce
  a deployment verdict. Tell the user before submitting: they get a model they cannot yet
  measure. A test set can be added later, and the model evaluated then.

## Working Directory

```
model-training/
├── base-input/          # the dataset id and its config; every run derives from it
├── smoke-<student>-1/   # memory calibration; one series per selected student
│   ├── input/           # the config sent for this run (and truncated data for smokes)
│   ├── run.md           # the dataset id, the override sent, and the SLM id returned
│   └── output/          # fetched metrics and training logs
├── smoke-<student>-2/   # next calibration attempt for that student, if needed
└── full-<student>-1/    # one full training per student
```

## Step 1: Prepare the Base Input Directory

The base input is a TrainingDataset, referenced by id. Two entry points:

- **Starting from synthetic data generation**: the TrainingDataset synthgen produced is
  ready to train as-is (the execution backend § Reuse synthetic data for training-only runs).
- **Retraining**: an existing TrainingDataset serves as the base input. The data stays
  untouched, and only the config changes.

**The dataset must have finished first.** Its config only becomes readable at `JOB_SUCCESS`.
Poll synthgen to completion before reading it, or the read raises. Do not chain
submit-synthgen → read-config in one go. That ordering fails every time.

Read its config once (the execution backend § Overrides). Every run derives from that config:
edit the fields this run varies and send the whole thing back as an override.
Smokes additionally need truncated data, which is the one variation an override cannot
express. See Step 3.

Key config is the `tuning` section (`../references/configuration.md`):
`base.student_model_name` (set per run in the next steps), `per_device_train_batch_size`
(default 1, where higher is faster but risks OOM), and `num_train_epochs` (default 4).

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- **The path.** Read the balances first (`../references/platform.md` § Credits).
  Credits decide what is *possible*. Where more than one thing is possible, the user chooses.

  | Balance | What it gates |
  |---|---|
  | `training_datasets_download_get` | the memory-calibration smokes, Steps 3-5. Calibration needs the dataset's longest rows and `/download` is the only route to them, so at zero the smokes are not possible |
  | `training_datasets_post` | staging each truncated smoke dataset; rarely the blocker |
  | `slms_from_training_datasets_post` | one per training run, so a sweep of N students needs N |

  So:

  - **No download credits** → Steps 3-5 are unavailable. Go straight to Step 6 with
    `per_device_train_batch_size: 1` and use the OOM levers below if a run fails. Nothing to
    decide.
  - **Fewer than N training credits** → a sweep of N is unavailable. Train the students the
    balance covers.
  - **Credits for both** → present the choice. The normal path calibrates memory settings per
    student before the full run. The fast path skips Steps 3-5 and submits the full run
    directly. Normal costs more credits and finds each student's fastest working
    configuration. Fast conserves both, and is the right call for a re-run of a proven setup.

  State the rough cost either way: each full run is multi-hour GPU time. An exhausted
  `slms_from_training_datasets_post` means the generation spend is already stranded.
- **The student list**: fixed now, before any smoke run, because memory settings are
  calibrated per student (a 4B model fits different settings than a 9B one). Fast path: one
  good default student, 4B-class unless the deployment target demands smaller
  (`../references/model-catalog.md`). Normal path: students across the sizes that fit the
  deployment target (`../references/model-catalog.md` § Size tiers, and tool-calling tasks
  restrict the family).

## Step 3: Smoke Run

**Requires `training_datasets_download_get` credits.** The Step 2 gate already resolved this:
without them the calibration cannot be done at all, so skip to Step 6 and train with
`per_device_train_batch_size: 1`. Do not substitute the free `/sample` here. It returns at
most 128 rows drawn from the first 384 and no test split, so the rows it yields are not the
dataset's longest, and the calibration it produces would bound nothing.

Calibration runs on the ~100 longest training examples. Selecting those means holding the
dataset, which is a data change that no config override can express. So it downloads the
dataset and stages a new one, on top of the training credit each attempt costs.

Training settings sit on a scale from fastest to most memory-tolerant: high batch size → low
batch size → the other OOM prevention methods (see Dealing with OOM below). The smoke runs
find each student's point on that scale (each `smoke-<student>-N` series is independent, and
larger students land lower), using worst-case examples:

- Download the dataset, truncate to the ~100 LONGEST train examples and ~10 LONGEST test
  examples (longest rows are what OOMs first), and stage them as a new TrainingDataset
  (the execution backend § The TrainingDataset). Because the data changed, this is a new
  dataset rather than an override, which is what makes the step cost the extra credit.
- **Classification: truncate per class, not by length alone.** Train and test must each carry
  every label in `classes_description`, so take the longest examples *within each class* until
  you reach the size you want. A split missing a label fails validation, and the check runs
  against train and test separately, so a truncated train split fails the same way a truncated
  test split does. With more classes than the target row count, raise the count to fit them.
- On that truncated dataset, train with the student set, `num_train_epochs: 1` and
  `per_device_train_batch_size: 4` in the config override. Record `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded and check: the metrics came back for base and tuned, training loss
decreased in the log, and the run fit in memory. Do not download the model to check the
artifacts. The smoke answers a memory question, and the tarball is gigabytes. Its metrics say
nothing about final quality either: tiny data, one epoch.

## Step 5: Iterate Until the Smoke Passes

If the run OOMed, move down the scale in the next smoke iteration (halve the batch size, and
from batch size 1 continue with the OOM levers below). If it ran, optionally try a higher
batch size. Keep each student's fastest surviving setting. Move on once every selected
student has one.

**Carry the setting forward as a ceiling, not as the value to train at.** The smoke answers
only "what fits in memory". Batch size is not a free speed knob: at a fixed
`num_train_epochs`, it divides the number of optimizer steps, so a larger batch trains the
model *less*. The drop on the primary metric can exceed what any other lever in this stage
changes.

So when the full run uses a calibrated batch size above 1, raise `num_train_epochs` with it
and say so when presenting the plan in Step 6. Holding the step count constant, by multiplying
the epochs by the same factor as the batch, is the upper bound. Less is often enough, and both
shipped examples run 4 epochs at batch 8. If credits do not allow more epochs, prefer the
smaller batch and the longer wall clock.

## Step 6: Full Run

Present the final plan to the user first: the students, each one's calibrated settings, the
epoch count that goes with them (Step 5), and the expected cost (one multi-hour training per
student). On their go-ahead, submit one training per student against the same TrainingDataset:
for each, take a copy of the dataset's config, set the student and its calibrated memory
settings (fast path: the default student with batch size 1), and send it as the override.

Take the copy per submission and send the config whole. A config trimmed to the fields you
changed silently reverts everything you left out (the execution backend § Overrides).

The data is untouched, so the sweep costs training credits only. Submissions run concurrently,
and the whole sweep is N ordinary calls (the execution backend § Submitting jobs).

## Step 7: Analyze the Results

Confirm each job succeeded, then pull the base and tuned student metrics into `output/`
(the execution backend § Fetch metrics). With no test set there are no metrics to pull and no
verdict to render, so stop here and report that. Otherwise analyze with a three-way comparison on
the primary metric (`../references/evaluation-metrics.md`): base student (floor), teacher
(ceiling), tuned student. For a sweep, one row per student. What to do with the result
(deploy, retune, or start a new workflow iteration) is decided in the workflow's Decide step
(`../workflows/dataset-to-model.md`).

## Dealing with OOM

When a training run crashes out of memory, apply these levers in order (each next one costs
more speed or quality). The first three are config, so they go through the training override
on any run. Only the fourth touches the data, so it needs the dataset downloaded:

1. Lower `per_device_train_batch_size` (halve it, down to a floor of 1).
2. `memory_optimized_training: true` (activation offloading and gradient checkpointing, and
   significantly slower).
3. `use_qlora: true` together with `memory_optimized_training: true` (4-bit base model,
   roughly 3x less VRAM).
4. Filter out the longest ~1%+ of training examples (the tail drives peak memory). To create
   a new trimmed smoke test: first trim the full dataset by removing the longest X%, then
   sample the longest 100 from the trimmed dataset to create the smoke inputs.
