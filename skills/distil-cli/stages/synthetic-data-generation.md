# Stage: Synthetic Data Generation

The teacher generates the training dataset from the seed data, job description, and mutators.
Validation and dedup then produce the merged training dataset (root `train.jsonl` = seed +
surviving synthetic) that model training consumes. When `synthgen.clean_training_targets` is
set, a final teacher pass minimally repairs corrupted or truncated training targets.
Multi-turn data is first expanded to per-turn examples, so every turn is covered.

## Working Directory

```
synthetic-data-generation/
├── smoke-1/
│   ├── input/           # the exact inputs this iteration ran
│   ├── run.md           # submission command and job identifiers
│   └── output/          # fetched datasets and logs
├── smoke-2/             # next iteration: copy the previous input, change one thing
└── full-1/              # last passing smoke input with full generation parameters
```

## Step 1: Prepare the Input Directory

The input is the standard directory (config.yaml, job_description.json, train/test.jsonl,
unstructured.jsonl where required). Prepare it following
`../references/data-preparation/overview.md` plus the task-specific page. In most projects
you start from the input that already passed teacher evaluation.

### The three levers on what gets generated

Every generation call is shaped by three inputs:

| Lever | Where | Applied |
|---|---|---|
| `task_description` | job_description.json | **constant**: every call, and every eval and judge prompt |
| `synthetic_data_generation_instructions` | job_description.json | **constant**: every generation call, generation only |
| `mutation_topics` / `basic_mutators_to_use` | config.yaml `synthgen` | **sampled**: one value per active mutator, per call |

- `task_description` says how to solve the task. It defines what a correct answer is, so it
  also feeds evaluation and the judge. Changing it changes what "correct" means everywhere.
  Keep it matched to the user's production prompt and constant across iterations
  (`../references/job-description.md`).
- `synthetic_data_generation_instructions` says how to generate the data: what the inputs look
  like, their formats, domains, register and noise. It touches generation only, so it is the
  safe place to steer the inputs without redefining the task.
- Mutators shape the distribution of the generated data. Each call samples one value per
  active mutator, so the composition of the list sets the proportions: three values asking for
  English and one asking for French gives roughly a 75/25 split (`../references/mutators.md`).

The two constants shift every example the same way. The mutators decide how the examples are
distributed. So when the whole dataset is wrong in the same way (a format, a misread rule, the
wrong register throughout), fix a constant. When the mix is wrong (a slice missing, or
over-represented against what production sees), change the mutator values and their
proportions.

Synthgen behavior is controlled by the `synthgen` section of config.yaml. The full parameter
table is in `../references/configuration.md`. The ones to set deliberately:

- `validation_max_total_length` sets the character cap on question + answer (+ context) per
  example. Set it to the maximum combined length you expect in real data: the default 30,000
  rejects longer uploaded examples and silently filters longer generated ones.
- When examples are long, lower `generation_in_single_call`,
  `num_positive_exemplars_per_generation`, and `num_unlabelled_exemplars_per_generation`. Each
  multiplies the prompt and output size per teacher call.
- `output_is_json: true` whenever answers must be valid JSON (QA tasks only).
- `base.llm_num_parallel_requests` above the default 4 can help, but do not expect linear
  gains: per-call latency and the between-batch validation usually dominate.

If the task already names patterns, domains or proportions to cover, translate them into
mutator values now. Otherwise keep the defaults and revisit after the smoke analysis.

## Step 2: Confirm the Setup with the User

Before submitting anything, present and confirm:

- the key synthgen config (`validation_max_total_length`, per-call and exemplar counts,
  `output_is_json`, the intended `generation_target`)
- the mutator plan (topics and built-ins now, or defaults first and revisit after the smoke)
- the remaining `training_datasets_from_seed_datasets_post` credits, one per smoke and one
  for the full run (`../references/platform.md` § Credits)
- the path: normal (a 64-example smoke, its analysis, then the full run) or fast (skip the
  smoke, Steps 3-5, and submit the full run directly)

## Step 3: Smoke Run

Run a minimal generation to check the setup before spending the full budget. In the iteration
copy of config.yaml set `generation_target: 64` and `generation_iteration_size: 16`, then
submit a synthetic-data-generation job via the execution backend (§ Submitting jobs).
Record the command and job identifiers in `run.md`.

## Step 4: Pull and Analyze the Smoke Outputs

Confirm the job succeeded, then pull the generated data into `output/`: the root
`train.jsonl`/`test.jsonl` (locations: the execution backend's output sections). The root
`train.jsonl` is the seed and the surviving synthetic examples MERGED, so the synthetic count
is root minus seed. Compare that count against the smoke target first: falling well short
means validation filtered heavily, usually length caps or format problems.

Then analyze the synthetic examples on three axes:

1. **Form against the job description**: does each sampled example parse, carry the required
   output format, and respect the stated constraints? A malformed row is a defect whatever
   the rate, so this axis is pass/fail on the shape, not a score on the answers. Read a
   handful of answers to check the teacher understood the task, but do not gate on a label
   error rate. The student absorbs a low rate of teacher noise, and there is no threshold
   that separates acceptable from not.
2. **Distribution match against the seed examples**: compare generated against seed data along
   the dimensions relevant to this task, for example length (characters, or turns for
   multi-turn), topic coverage, style and register, class balance. Pick the dimensions that
   matter for the task and quantify where possible.
3. **Targeted slices actually materialized**: if mutators or the job description asked for a
   particular slice (a topic, a class, a value range), count it in the output. Mutation
   topics are suggestions and can silently yield nothing.

   **Only read this axis when the smoke is large enough to answer it.** A run makes
   `generation_target / generation_in_single_call` mutator draws. At the default target of 64
   and `generation_in_single_call: 4`, that is 16 draws. A grid of more than 16 cells
   therefore leaves cells empty by pigeonhole, whatever the config says, and empty cells at
   this scale tell you nothing. Size the smoke to at least ~2x the grid before treating axis 3
   as a gate, or skip it here and check it on the full run.

Problems visible in 64 examples will be everywhere in 10,000. This is the cheap moment to fix
them.

## Step 5: Iterate Until the Smoke Passes

If the analysis fails on any axis, adjust one of the three levers (Step 1) in a new smoke
iteration: mutator values and their proportions for the mix of examples, the generation
instructions for how the inputs themselves look, the relevant config field for scale and
validation. Present each smoke analysis to the user. Only move on once a smoke run passes
every axis and the user agrees to the full run.

## Step 6: Full Run

Copy the last passing smoke `input/` to `full-1/input/`, restore the intended
`generation_target` (default 10000) and `generation_iteration_size` (default 128), and
submit.

## Step 7: Analyze the Results

When the run finishes, pull a sample of the root `train.jsonl` and repeat the Step 4
analysis, plus check the final size (expect seed count + surviving synthetic, merged) against
the target and the overall balance. `generation_target` is a floor rounded up to the next
`generation_iteration_size` batch, so landing above it is normal. Landing well short means
validation filtered heavily.

Present the findings to the user, then take one of two branches:

- **It passes.** The dataset is ready for model training (`model-training.md`).
- **It fails.** Do not carry the dataset into training. Bad data does not announce itself in
  the training metrics. It shows up as a plateau you will spend a full training run
  discovering. Go back to Step 5's levers and submit another full run: mutators for coverage
  gaps, the relevant config or job-description field for correctness or format problems. A
  regenerated dataset costs one synthgen credit. Training on a bad one costs a training credit
  and hours, and then you regenerate anyway.
