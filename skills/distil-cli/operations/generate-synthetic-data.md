# Operation: Generate Synthetic Data

The teacher writes new rows for one split, using the rows already in that split as examples, the
job description, the mutators, and the traces left in the Dataset as context. Validation and
deduplication keep the rows that pass, and the job writes a new Dataset with them added to the
split and the context traces removed (`../references/platform.md` § The expand operations). The
parent is unchanged.

| | |
|---|---|
| **Called by** | the make-rows protocol, for the test split (test-set phase) or the train split (model-iteration phase), as a smoke or a full run |
| **Price** | the generation target times the average tokens of a row in the parent, at the teacher's tier (`../references/platform.md` § Billing). A Dataset made by an expand carries the task description in every row, so a long task description raises the price |

## Settings that control the output

| Setting | File | What it changes |
|---|---|---|
| `synthetic_data_generation_instructions` | `job_description.json` | A constant added to every generation call: what the inputs look like (formats, domains, register, noise). It moves every row the same way, so it fits when the whole set is off in one direction |
| `synthgen.mutators` | `config.yaml` | The dimensions the rows are spread over, one value sampled per call, weighted when the mix matters (`../references/mutators.md`). They shape the distribution, so they fit coverage: slices missing, rare or over-represented. For chat completion with tools, a mutator over the tools spreads rows across them (`../references/data-preparation/chat-completion.md` § Synthetic data with tools) |
| `synthgen.validation_instructions` | `config.yaml` | An LLM validator that rejects rows against criteria you write (§ LLM validation below) |
| `synthgen.train_generation_target`, `synthgen.test_generation_target` | `config.yaml` | How many rows the run adds to the split |
| the rest of `synthgen` | `config.yaml` | Volume and cleanliness, not what rows are about: `use_traces_as_context`, `validation_max_total_length`, `validation_similarity_threshold` (lower it for near-duplicates), `generation_in_single_call` and the exemplar counts, `teacher_temperature`, `clean_training_targets`, `output_is_json` (`../references/configuration.md` § synthgen) |
| `base.teacher_model_name` | `config.yaml` | Who writes the rows. A stronger teacher fixes errors that clearer instructions do not |
| the split's rows | the parent Dataset | The examples. A run on the latest Dataset generates from every row in it, relabelled and generated alike; a run on an earlier Dataset generates from fewer |

`task_description` is not one of them: it defines what a correct answer is for evaluation and the
judge too, and stays fixed (`../references/job-description.md` § What each field feeds). For a
reasoning student see `../references/reasoning-models.md`; with `base.variable_system_prompt` on,
add the system-prompt mutator (`../references/variable-system-prompts.md` § Generation). When the
answer is a label, check whether a mutator causes label errors (`../references/mutators.md`
§ Mutators for labels).

### LLM validation

`synthgen.validation_instructions` checks each generated row against criteria you write; they
need not appear in the job description:

```yaml
synthgen:
  validation_instructions: |
    Reject examples where the assistant hangs up without saying goodbye.
```

The validator is `evaluation.llm_as_a_judge_model_name` (falling back to the teacher). It sees
the instructions, the job description and the whole row, runs after the structural and diversity
checks with one call per surviving row, and drops rejected rows without rewriting them. Omit the
field or set it to `null` to turn it off; an empty string is invalid. It runs during generation
only, not during relabelling (`../references/configuration.md` § LLM validation).

## Working directory

```
synthetic-data-generation-<split>/
├── smoke-1/
│   ├── input/        # the config and job description sent
│   ├── output/       # the new Dataset, downloaded, and the parent's split for the diff
│   ├── review/       # filled by the blind review
│   └── run.md        # parent id, change tested, price, new Dataset id, result
├── smoke-2/
└── full-1/
```

## Step 1: Prepare the input

The input is a Dataset id. Read its config and job description into `input/`, edit the settings
for this run, and keep the whole files: an override replaces each file whole, and an omitted
field or section takes the library default, not the parent's value. A base-only config runs with
`train_generation_target: 10000` and no mutators (`../references/execution/cli.md` § Overrides).

```bash
distil dataset download-metadata -d synthetic-data-generation-<split>/smoke-1/input <dataset-id>
```

A **smoke** is the same override with the split's target set to the smoke size from the run log
header (`../references/platform.md` § Smoke runs). A **full run** sends the settings of the last
passing smoke with the planned target.

What the job reads from the parent:

- **The split's rows are the examples.** An empty split generates zero-shot, and the exemplar
  counts are capped to the rows there are; no rows then fix the format or style. A train split
  that an earlier run expanded into per-turn rows (`clean_training_targets` or `enable_thinking`
  on a multi-turn task) is refused as examples: generate from the Dataset before that run.
- **The other split is held out.** New rows that duplicate any row of it are dropped, so
  generating into test never leaks train rows, and the reverse.
- **The traces are the context** when `use_traces_as_context` is true: `T + min(T, 1000)` of them
  for a target of T, or all that are left, and they leave the Dataset. Below `min(T / 4, 10)`
  traces the job logs a warning, generates without context and leaves the traces alone. Count
  the traces first: one run can empty the pool for every expand after it.

Set deliberately: `validation_max_total_length` to the longest row expected; lower
`generation_in_single_call` and the exemplar counts when rows are long; `output_is_json: true`
when answers must be JSON (question answering only).

## Step 2: Price it

```bash
distil dataset generate-synthetic-data-<split> --dry-run --config <input>/config.yaml \
  --job-description <input>/job_description.json <dataset-id>
```

Add the price to the run log's spending and to the run's `run.md`. When the parent's rows are
long, a smoke on an earlier Dataset with the same example rows costs a fraction; whether that
fits is the calling protocol's choice. If the balance does not cover the price, the submission
would be refused with 402: report both numbers to the user with
https://app.distillabs.ai/billing.

## Step 3: Submit

Submit on the parent, never on a smoke's Dataset
(`../references/execution/cli.md` § Synthetic data generation):

```bash
distil dataset generate-synthetic-data-<split> --output json --config <input>/config.yaml \
  --job-description <input>/job_description.json <dataset-id> | jq -r .id
```

Check the exit code (an error prints `{"error": …}`). Record the new Dataset id in the run log's
lineage, with the parent, and give the user https://app.distillabs.ai/datasets/<new-id>.
To stop it before it finishes: `../references/execution/cli.md` § Cancel a job.

## Step 4: Wait and pull

Wait with `bash poll.sh dataset <new-id>` (`../references/execution/cli.md` § Wait for a job).
On `JOB_FAILURE`, search the log for the first error (`../references/platform.md` § Job status).
On success, download the new Dataset, the parent's split and the job log into `output/`:

```bash
distil dataset download -d synthetic-data-generation-<split>/smoke-1/output/new <new-id>
distil dataset download -d synthetic-data-generation-<split>/smoke-1/output/parent <dataset-id>
distil dataset logs --output json <new-id> | jq -r .logs > synthetic-data-generation-<split>/smoke-1/output/job.log
```

To check what the run used, compare `input/config.yaml` with the new Dataset's config
(`distil dataset download-metadata`).

## Step 5: Read the output

The new rows are the lines of the split after the parent's rows; diff against the parent's split
to isolate them, since nothing in a row says it was generated.

- **Count.** The final size against the target. The target rounds up to the next
  `generation_iteration_size` batch, so landing above it is normal. Falling short means
  validation filtered heavily, usually on length or format. A run that adds no valid rows for 8
  batches in a row stops short and succeeds with what it has; its log lists the most common
  validation errors of the last batch, which say what to change.
- **Form against the job description.** Each row parses, carries the required output format and
  respects the stated constraints. A malformed row is a defect at any rate. Read a handful of
  answers to check the teacher understood the task. Label correctness is verified by the blind
  review the calling protocol runs.
- **Distribution against the examples and the traces.** Length (characters, or turns), topic
  coverage, style and register, class balance; with no examples, compare against the traces.
- **Requested slices present.** Count the rows of each mutator value or slice asked for: a
  mutator value is a request and can produce nothing. With more than 16 cells in the mutator
  grid, a smoke is too small to count every cell; count them on the full run.
- **LLM validation.** When it is on, read a sample of the rows it kept and check them against the
  validation instructions: the validator makes mistakes too.
- For a reasoning student, check the reasoning (`../references/reasoning-models.md`). With
  `base.variable_system_prompt` on, check the system messages
  (`../references/variable-system-prompts.md` § Generation).

Record the row counts and findings in `run.md` and the run log.
