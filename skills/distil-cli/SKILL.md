---
name: distil-cli
version: 6.6.2
description: >
  Use when building or training a model on the distil labs platform end to end: preparing
  model-building inputs (config.yaml, job_description.json, train/test data), running or
  analyzing any pipeline stage (trace processing, teacher evaluation, synthetic data
  generation, model training, deployment), iterating when a teacher score or student score is
  too low, or deciding what to run next after a stage completes. Activate for phrasings like
  "build a model for X", "train a student model", "run a teacher eval", "generate synthetic
  data", "the score is low, what now", or "retrain on existing synthetic data". Stages run
  through the distil labs API. This skill owns the model-building logic on top of it.
  Also activate for distil labs lookups that are not a full build: distil CLI commands
  (distil auth, distil seed-dataset, distil traces, distil teacher-evaluation,
  distil training-dataset, distil slm, distil deployment), the distil labs REST API,
  config.yaml parameters, supported student and teacher models, or what an evaluation
  metric means.
---

# Building Models

Building task-specific small language models on the distil labs platform: from raw data or
production traces, through teacher evaluation and synthetic data generation, to a finetuned,
evaluated, deployable student model.

## Architecture

The skill separates what to do from how to run it, so the model-building logic stays
independent of the mechanics of running a job.

- **Stage**: one unit of pipeline work. Each stage file defines purpose, inputs, run, outputs,
  analysis (with its report template), and iteration levers. Stages are directly invocable.
- **Workflow**: sequences stages toward a goal. It owns the decision points, verdicts and
  gates.
- **References**: knowledge shared by both (data formats, configuration, models, metrics).
- **Execution backend** (`references/execution/`): the only files that know how stages actually
  run. They hold the commands and the request shapes. How the platform *behaves* lives in
  `references/platform.md`, so a second backend adds commands rather than restating the model.
  Stage files cite operations by § name alone, so resolve those in the backend in use.
  `references/execution/README.md` owns the choice: install the `distil` CLI and use
  `cli.md`, and fall back to `backend-api.md` only when the install is impossible. Make that
  choice once, before the first stage, and record it in `run.md`.

## Stage Protocol

Each stage file defines a step-by-step protocol over a `<stage-name>/<iteration-id>/` working
directory. All stage directories live under one project root named after the model, for example
`my-model/teacher-evaluation/smoke-1/`. Stages follow one shared template:

1. Prepare the Input Directory
2. Confirm the Setup with the User
3. Smoke Run
4. Pull and Analyze the Smoke Outputs
5. Iterate Until the Smoke Passes
6. Full Run
7. Analyze the Results

Smoke runs (`smoke-1`, `smoke-2`, ..., then `full-1`) exist because full runs take hours and
burn GPU and LLM budget. A minimal run catches config, data and prompt problems in minutes.
The saving is real for synthgen and trace processing, where cost scales with the data. It is
much smaller for training, where the fixed overhead dominates: model load, base eval, tuned
eval. A measured training smoke on 100 rows took 93% of the wall clock of the full run on 654.
Its value there is the memory-fit answer, not the time saved.

Teacher evaluation and test-set expansion are small enough to run directly, so they have no
smoke steps (3-5). Deployment has no smoke/full split either, and keeps its own short
choose-route/serve/smoke-test shape. Follow the stage file's steps in order.

**Runs are not reproducible, by design.** The teacher generates and the judge scores at
non-zero temperature, so the same config submitted twice gives different data and different
numbers. Two runs of one synthgen config produced 512 and 634 examples. One untrained model
scored 0.64, 0.60 and 0.58 on one 50-row test set. `base.random_seed` does not pin this. Treat
every score as a sample. Quote the run it came from, and never resolve a decision on a
difference smaller than the noise band in `references/evaluation-metrics.md` § Verdicts.

Every stage opens with a user gate. Before submitting anything, present the setup: the prepared
input, the key config choices, the plan for smokes and sweeps, and the expected cost against
the remaining credits for the routes it spends (`references/platform.md` § Credits). Then
confirm which path the user wants, normal (smoke first) or fast (skip the smoke steps and
submit the full run directly), and get their go-ahead. After a stage's analysis, present the
findings and agree on the next step together. Nothing launches on defaults the user never saw.

## File Map

### Stages

One unit of pipeline work each, directly invocable. A stage file is a step-by-step procedure
following the shared template above: what inputs it needs, what to confirm with the user, how
to run it, and how to analyze the results.

| File | Purpose |
|---|---|
| `stages/trace-processing.md` | Convert production traces into training-ready seed data |
| `stages/teacher-evaluation.md` | Feasibility check: can the teacher solve the task? |
| `stages/synthetic-data-generation.md` | Teacher generates the synthetic training dataset |
| `stages/model-training.md` | Finetune the student on synthetic data and evaluate it |
| `stages/model-deployment.md` | Fetch and run the trained model |
| `stages/test-set-expansion.md` | Grow the test set into uncovered areas (inverted synthgen) |

### Workflows

Sequencers over stages toward a goal. A workflow file contains almost no operational detail: it
orders the stages, owns the decision points and gates between them, and says when to deviate
(skip a stage, take a different path). Each workflow opens with a Workflow Map, an ASCII
diagram of its steps, gates and loops. Print that map to the user when the workflow starts.

| File | Purpose |
|---|---|
| `workflows/dataset-to-model.md` | End to end from a labeled dataset; owns the decide step |
| `workflows/traces-to-model.md` | End to end from production traces |
| `workflows/improving-a-model.md` | Iteration 2: gap diagnosis, test-set expansion, seed blending, targeted mutators |

### References

Shared knowledge, one focused page per topic: formats, parameters, catalogs, and the
execution backends. Stages and workflows link here instead of repeating facts. Each fact has
exactly one owning page.

| File | Purpose |
|---|---|
| `references/platform.md` | How the platform behaves: entities, jobs, overrides, credits, outputs |
| `references/task-types.md` | The task types, which needs context/unstructured data, how to choose |
| `references/data-preparation/overview.md` | Input directory contract and validation checklist (read first) |
| `references/data-preparation/<task>.md` | Per-task data format (one page per task type) |
| `references/data-preparation/traces.md` | Trace input formats for trace processing |
| `references/job-description.md` | Writing good job descriptions per task type |
| `references/configuration.md` | config.yaml parameters, defaults, cross-field validation |
| `references/model-catalog.md` | Teacher and student models, task compatibility, llm providers |
| `references/mutators.md` | Synthetic data diversity controls (`synthgen.mutators`, value recipes) |
| `references/evaluation-metrics.md` | Metrics per task type, primary metrics, relative verdict gates |
| `references/deployment.md` | Model artifacts and serving options |
| `references/execution/README.md` | Which backend to use: install the CLI, fall back to the API |
| `references/execution/cli.md` | Execution backend: run stages through the `distil` CLI (default) |
| `references/execution/backend-api.md` | Execution backend: run stages through the distil labs API |


## Routing

- End-to-end request ("build/train a model for X") → ask whether the starting point is a
  labeled dataset or production traces, then load the matching workflow.
- Single-stage request ("run a teacher eval", "regenerate the synthetic data") → load that
  stage file plus the execution backend in use. If none is chosen yet, run
  `references/execution/README.md` § Choose the backend first.
- Results are disappointing ("teacher score is low", "student is far below teacher") → the
  relevant stage's levers, or `workflows/improving-a-model.md`.
- Lookup question (a config parameter, a metric, a data format) → the matching reference file.
