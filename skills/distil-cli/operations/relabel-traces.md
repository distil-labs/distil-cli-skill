# Operation: Relabel Traces

Turns traces into rows of one split. The job takes `trace_processing.num_{split}_relabelled`
traces from the parent Dataset, optionally filters them for relevance, has a teacher (or a
committee) rewrite the production answers, validates the rows, and writes a new Dataset with the
rows added to the split and the used traces removed (`../references/platform.md` § The expand
operations). The parent is unchanged.

| | |
|---|---|
| **Called by** | the make-rows protocol, for the test split (test-set phase) or the train split (model-iteration phase), as a smoke or a full run |
| **Price** | the relabelled traces times their average tokens, once per teacher in the committee, at each relabelling teacher's tier (`../references/platform.md` § Billing) |

## Settings that control the output

The `trace_processing` section of the config controls this job; parameters and defaults:
`../references/configuration.md` § trace_processing.

| Setting | File | What it changes |
|---|---|---|
| `trace_processing_instructions` | `job_description.json` | Guidance added to every relabelling call: how to rewrite the answers of these traces (their quirks), without changing what a correct answer is |
| `trace_processing.teacher_model_name`, `trace_processing.relabelling_committee_models` | `config.yaml` | Who rewrites the answers. On the test split they write the reference answers, so they decide what every score measures; on the train split, generation later imitates their rows |
| `trace_processing.relevance_filtering`, `min_relevance_score`, `min_coherence_score` | `config.yaml` | Drops traces that do not belong to this task. For traces where several applications or agents are mixed together; it does not filter out inputs the user defined as out of scope, which still belong to the task. Off by default |
| `trace_processing.num_{split}_relabelled` | `config.yaml` | How many traces become rows. More test rows narrow the noise; more train rows give generation more real examples. Both spend traces the other split cannot use |
| `trace_processing.compress_job_description` | `config.yaml` | Shortens a long task description for the filtering model |
| `synthgen.validation_max_total_length` | `config.yaml` | The longest row allowed, in characters. Raise it when the traces embed documents or schemas |
| `trace_processing.observation_format` | `config.yaml` | Must match the shape of `traces.jsonl` (`../references/data-preparation/traces.md`) |

`trace_processing.teacher_model_name` inherits `base.teacher_model_name` only until the config is
first expanded; in a config read back from an expanded Dataset it is set on its own, so a new
relabelling teacher goes into both fields (`../references/model-catalog.md` § Defaults). Use an
open-weight teacher: proprietary models are for teacher evaluation only.

Never set `trace_processing.relabel: false`: the production answers would pass through
unreviewed, and the student would learn to imitate the model it is meant to beat. Answers the
user vouches for are uploaded as rows, not relabelled. With `base.enable_thinking: true`, every
relabelled assistant turn carries reasoning (`../references/reasoning-models.md` § Where the
reasoning comes from). Keep the thinking setting the same for every split and run.

The new traces come only from the user's traffic: relabelling can only produce what the traces
contain.

## Working directory

```
relabel-traces-<split>/
├── smoke-1/
│   ├── input/        # the config and job description sent
│   ├── output/       # the new Dataset, downloaded, and the parent's split for the diff
│   ├── review/       # filled by the blind review
│   └── run.md        # parent id, change tested, price, new Dataset id, result
├── smoke-2/
└── full-1/
```

## Step 1: Prepare the input

The input is a Dataset id with traces in it. Download it (`../references/execution/cli.md`
§ Reading a Dataset) and count the distinct traces. The job refuses to run when the Dataset holds
fewer distinct traces than `num_{split}_relabelled`, a smoke included, and every trace it uses
is gone for the expands after it.

Read the parent's config and job description into `input/`, change the settings for this run,
and keep the whole files: an override replaces each file whole, and an omitted field takes the
library default, not the parent's value (`../references/platform.md` § Overrides).

```bash
distil dataset download-metadata -d relabel-traces-<split>/smoke-1/input <dataset-id>   # free
```

A **smoke** is the same override with `num_{split}_relabelled` set to the smoke size from the run
log header (`../references/platform.md` § Smoke runs). A **full run** sends the settings of the
last passing smoke with the planned count.

Behaviour to expect:

- **No floor on the rows.** Filtering, relabelling, validation and the overlap check against the
  other split each drop traces, and whatever survives is written, zero rows included.
- **The traces are shuffled and deduplicated first**, so two runs on the same parent pick
  different traces, and duplicates in `traces.jsonl` do not count.
- **The remaining traces are written back in `openai_messages` form**, without their system
  prompts unless `base.variable_system_prompt` is on, whatever the parent's observation format.
- **A trace without an answer gets one.** A trace that ends on the user's turn is completed with
  one assistant turn the teacher writes; an observed answer is corrected while staying as close
  to it as possible, so it keeps that answer's habits.

## Step 2: Price it

```bash
distil dataset relabel-traces-<split> --dry-run --config <input>/config.yaml \
  --job-description <input>/job_description.json <dataset-id>
```

Add the price to the run log's spending and to the run's `run.md`. If the dry run reports that
the balance does not cover it, the submission would be refused with 402: report the price and the
balance to the user with https://app.distillabs.ai/billing.

## Step 3: Submit

Submit on the parent, never on a smoke's Dataset (`../references/execution/cli.md` § Relabel
traces):

```bash
distil dataset relabel-traces-<split> --output json --config <input>/config.yaml \
  --job-description <input>/job_description.json <dataset-id> | jq -r .id
```

Check the exit code: an error prints `{"error": …}` and `jq` prints `null`. Record the new
Dataset id in the run log's lineage, with the parent, and give the user
https://app.distillabs.ai/datasets/<new-id>.
To stop it before it finishes: `../references/execution/cli.md` § Cancel a job.

## Step 4: Wait and pull

Wait with `bash poll.sh dataset <new-id>` (`../references/execution/cli.md` § Wait for a job).
On `JOB_FAILURE`, read the log and search it for the first error
(`../references/platform.md` § Job status); a failed job is refunded. On success, download the
new Dataset and the parent's split into `output/`:

```bash
distil dataset download -d relabel-traces-<split>/smoke-1/output/new <new-id>
distil dataset download -d relabel-traces-<split>/smoke-1/output/parent <dataset-id>
distil dataset logs --output json <new-id> | jq -r .logs > relabel-traces-<split>/smoke-1/output/job.log
```

## Step 5: Read the output

The new rows are the lines of the split after the parent's rows; nothing in a row says it was
relabelled, so diff against the parent's split to isolate them.

- **Count.** How many traces reached the split, against the count asked for, and how many traces
  remain (the job log says how many were dropped and why). Near-total loss usually means the
  relevance filter and the job description disagree about what the task is. A count far below
  the traces used, without filtering, usually means the traces do not match the task type, for
  example multi-turn traces for a single-turn task.
- **Correctness.** Valid rows in the right format, and relabelled answers that are correct under
  the job description and better than the originals where they differ. Read a handful of traces
  and their rows side by side. Label correctness is verified by the blind review the calling
  protocol runs; this check catches format and gross errors.
- **Distribution against the traces.** Length (characters, or turns), topic coverage and style.
  Watch for filtering that dropped whole categories.

Record the row counts and findings in `run.md` and the run log.
