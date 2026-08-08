# Stage: Trace Processing

Turns raw production traces into a training-ready input directory: traces are filtered for
relevance, relabeled by a teacher (optionally a committee), and split into train/test plus
unstructured context. The original model is also evaluated on the generated test set, which
gives the baseline the trained student must beat. Setting `evaluate_original_model` to false
skips that.

## Working Directory

```
trace-processing/
├── smoke-1/
│   ├── input/           # the exact inputs this iteration ran
│   ├── run.md           # submission command and job identifiers
│   └── output/          # fetched processed data and original-model eval
├── smoke-2/             # next iteration: copy the previous input, change one thing
└── full-1/              # last passing smoke input with the full trace set
```

## Step 1: Prepare the Input Directory

The input is a trace directory: traces.jsonl, job_description.json, config.yaml with a
`trace_processing` section, and optionally a curated test.jsonl (which then replaces the
generated test split). Convert raw logs following
`../references/data-preparation/traces.md`, pick the task type first
(`../references/task-types.md`), and write the job description per
`../references/job-description.md`. The job description's optional
`trace_processing_instructions` field carries task-specific guidance for the rewrite and fix
edits, for example "preserve the caller's interruptions verbatim" for phone-call transcripts.

Trace processing is controlled by the `trace_processing` config section. The full table is in
`../references/configuration.md`. The ones to set deliberately:

- `observation_format` must match the shape of traces.jsonl (`openai_messages` default, image
  and unstructured variants).
- `relabel` (default true) has the teacher rewrite assistant answers, and
  `relabelling_committee_models` upgrades this to a committee. Leave it on. Relabeling is the
  point of this stage, and `relabel: false` reduces it to filtering and splitting: the
  original production answers pass through unreviewed, so the student only learns to imitate
  the model it is meant to beat. Do not turn it off. When relabeled answers look worse than
  the originals, fix the relabeling teacher or committee instead.
- `relevance_filtering` is off by default, so every seed trace flows straight through. Set
  it `true` to have an LLM score traces and drop the low relevance/coherence ones. That
  costs an LLM pass over every trace, and `min_relevance_score` / `min_coherence_score` only
  apply once it is on.
- `num_traces_as_training_base` / `num_traces_as_testing_base` (defaults 200/200) seed the
  splits, testing base ≥ 1. Equal counts are a reasonable default. Weight the testing base
  higher when you want a larger test set than training set. Leftover traces become
  unstructured context.
- `min_generated_examples` (default 1) is a floor checked per split, train and test each
  separately, after filtering and relabeling have dropped a fraction of the traces. Its
  ceiling is therefore the SMALLER of the two base counts, and well below that in practice.
  A supplied `test.jsonl` is rejected up front when it has fewer rows than this.
- `evaluate_original_model` (default true) produces the baseline the student must beat, and is
  this stage's LLM-judge cost. Set it false on smokes (Step 3).
- `synthgen.validation_max_total_length` also applies to processed examples. When traces embed
  documents or schemas, raise it.
- `compress_job_description: true` if the task description is very long and would overwhelm
  the filtering model.

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- what the raw traces look like (count, observation format, typical length) and the chosen
  task type
- the processing config (relabel/committee, relevance filtering on or off, trace-base
  counts) and the trace-processing teacher
- the remaining `prepared_traces_post` and `seed_datasets_from_prepared_traces_post`
  credits, since a run spends one of each (`../references/platform.md` § Credits)
- the path: normal (a small-slice smoke, its analysis, then the full run) or fast (skip the
  smoke, Steps 3-5, and submit the full run directly). Either way the test-set review follows
  the full run.

## Step 3: Smoke Run

Process a small slice first to check the conversion and the processing settings. Subsample
`traces.jsonl` to ~50-100 lines, and in the iteration copy of config.yaml set:

```yaml
trace_processing:
  num_traces_as_training_base: 10
  num_traces_as_testing_base: 10
  evaluate_original_model: false
  min_generated_examples: 2
```

**`min_generated_examples` must come down with the base counts.** It is a floor checked per
split *after* filtering and relabeling have dropped a fraction of the traces, so a value tuned
for a full run fails outright at smoke scale.

The subsample is a data change, so it stages a PreparedTraces of its own. Submit against it
via the execution backend (§ Trace processing) and record the identifiers in `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded, then pull the outputs into `output/`. The processed train/test data
and the config and job description copies come from the SeedDataset's download route, and the
original-model evaluation from its metrics route as `base_model_performance` plus the
per-example predictions behind `base_model_predictions_download_url` (the execution backend
§ Output layout). Check the survival rate first: how many traces made it through filtering and
relabeling. Near-total loss usually means the relevance filter and the job description
disagree about what the task is.

With `evaluate_original_model: false` the metrics response returns nulls for both fields.
Expected for a smoke, not a failure. The baseline comes from the full run.

Then analyze the processed examples on two axes:

1. **Correctness against the job description**: are the rows valid task examples in the right
   format, and are relabeled answers actually correct, and better than the originals where
   they differ?
2. **Distribution match against the raw traces**: compare processed examples to the incoming
   traces along task-relevant dimensions, for example length (characters, or turns), topic
   coverage, style. Watch for filtering that silently dropped whole categories.

## Step 5: Iterate Until the Smoke Passes

If the analysis fails, adjust the inputs in a new smoke iteration: `relevance_filtering:
true` when irrelevant or incoherent traces reach the output, the relabeling teacher or
committee settings when relabeled answers are worse than originals (not `relabel: false`, see
Step 1), `trace_processing_instructions` in the job description when the rewrites mishandle
task-specific quirks. These are all settings, so each iteration is a config override on the
smoke's PreparedTraces rather than a fresh staging (the execution backend § Trace
processing). Only move on once a smoke run passes both checks.

## Step 6: Full Run

Copy the last passing smoke `input/` to `full-1/input/`, then:

- restore the full `traces.jsonl` and the intended base counts (defaults 200/200)
- raise `min_generated_examples` to a fraction of the smaller base count, so a thinned split
  fails loudly
- drop the smoke's `evaluate_original_model: false`, so the run produces the baseline

Then submit.

The full trace set is a data change, so it stages one PreparedTraces. Every run after this one
is an override on it: a retry, a settings change, a later iteration.

## Step 7: Analyze the Results

Repeat the Step 4 analysis on the full output, and review the generated test set closely
with the user (size, label and length distribution, edge-case coverage) together with the
original-model baseline. This test set gates every downstream verdict. If it is not
trustworthy, fix it now, supply a curated test.jsonl, or grow it with
`test-set-expansion.md`. The processed output doubles as the input directory for teacher
evaluation (`teacher-evaluation.md` and `synthetic-data-generation.md`).
