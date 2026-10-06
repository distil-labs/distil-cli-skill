---
name: distil-cli
metadata:
  version: "9.1.0"
description: >
  Use when building or training a model on the distil labs platform end to end: preparing
  model-building inputs (config.yaml, job_description.json, train/test data), running or
  analyzing any pipeline stage (trace processing, teacher evaluation, synthetic data
  generation, model training, deployment), iterating when a teacher score or student score is
  too low, or deciding what to run next after a stage completes. Activate for phrasings like
  "build a model for X", "train a student model", "run a teacher eval", "generate synthetic
  data", "the score is low, what now", or "retrain on existing synthetic data". Stages run
  through the distil CLI or the distil labs API. This skill owns the model-building logic on top of it.
  Also activate for distil labs lookups that are not a full build: distil CLI commands
  (distil auth, distil seed-dataset, distil traces, distil teacher-evaluation,
  distil training-dataset, distil slm, distil deployment, distil inference-endpoint,
  distil api-keys, distil access-token), the distil labs REST API, config.yaml parameters, supported student
  and teacher models, or what an evaluation metric means.
---

# Building Models

Building task-specific small language models on the distil labs platform: from production
traffic, a trace file or a labelled dataset, through teacher evaluation and synthetic data
generation, to a finetuned, evaluated student served behind an inference endpoint.

Before the first stage, install the `distil` CLI and sign in per
`references/execution/README.md` § Set up the CLI, then settle the backend per § Choose the
backend.

## How the Skill Is Organised

The skill separates what to do from how to run it. Each fact has exactly one owning page, and
every other page links to it.

- **Stages** (`stages/`): one unit of pipeline work each, directly invocable. A stage file is a
  step-by-step procedure: what it needs, what to confirm with the user, how to run it, how to
  analyze the results and, for the stages an iteration re-enters, § What Can Be Changed to
  Improve the Next Iteration.
- **Workflows** (`workflows/`): sequence the stages and own the decision points, verdicts and
  gates. Each opens with a Workflow Map; print it to the user when the workflow starts.
- **References** (`references/`): shared knowledge, one page per topic.
  `references/platform.md` owns how the platform behaves.
- **Execution backends** (`references/execution/`): the only files that know how stages actually
  run. `cli.md` holds every `distil` command and `backend-api.md` every REST API request; stages
  and workflows link their sections, and both files use the same § names, so resolve a citation
  in the backend in use. `references/execution/README.md` owns the setup and the choice: install
  the `distil` CLI and sign in, then use `cli.md` by default or `backend-api.md` when the user
  prefers the API. Make that choice once, before the first stage, and record it in `run.md`.

## Stage Protocol

Each stage works in a `<stage-name>/<iteration-id>/` directory under one project root named
after the model, for example `my-model/synthetic-data-generation/smoke-1/`. Stages with a smoke
run follow one template:

1. Prepare the Input
2. Confirm the Setup with the User
3. Smoke Run
4. Pull and Analyze the Smoke Outputs
5. Iterate Until the Smoke Passes
6. Full Run
7. Analyze the Results

A smoke run (`smoke-1`, `smoke-2`, then `full-1`) catches config, data and prompt problems
on a subsample before the full run (`references/platform.md` § Smoke runs; durations in
§ Job status). The user picks the normal path (smoke first) or the fast path (full run
directly). Stages without a smoke run number their steps consecutively.

Every stage opens with a user gate: present the input, the key config choices, the plan and
the expected cost against the remaining credits (`references/platform.md` § Credits), and get
the user's go-ahead. Nothing launches on defaults the user never saw. After the analysis,
present the findings and agree on the next step together.

Runs are not reproducible: the teacher and the judge run at non-zero temperature, so treat
every score as a sample, and never decide on a difference smaller than the run-to-run variation
measured on the same model and test set (`references/evaluation-metrics.md` § Verdicts).

## File Map

| Stage | Purpose |
|---|---|
| `stages/inference-endpoint.md` | Create an inference endpoint: collect production traces, or serve a trained model to production traffic |
| `stages/test-set-from-traces.md` | Build a test set from production traffic (traces) |
| `stages/trace-processing.md` | Turn traces into a seed dataset |
| `stages/teacher-evaluation.md` | Feasibility check: can the teacher solve the task? |
| `stages/synthetic-data-generation.md` | The teacher generates the training dataset |
| `stages/model-training.md` | Finetune the student and evaluate it |
| `stages/local-deployment.md` | Serve the trained model with vLLM on your own GPU |

| Workflow | Purpose |
|---|---|
| `workflows/build-a-model.md` | The end-to-end loop: endpoint → traces → test set → seed dataset → synthetic data → training → serving endpoint → back to traces. Entry points: an LLM in production, a trace file, a labelled dataset |
| `workflows/model-iterations.md` | Improving a trained model: read the scores, inspect the predictions, find the stage the errors come from, change it, run again. Runs autonomously to an agreed target |

| Reference | Purpose |
|---|---|
| `references/platform.md` | How the platform behaves: entities, jobs, smoke runs, overrides, credits, outputs |
| `references/task-types.md` | The four task types and how to choose between them |
| `references/data-preparation/overview.md` | Input directory contract and validation checklist (read first) |
| `references/data-preparation/<task>.md` | Per-task data format: `question-answering.md`, `classification.md`, `chat-completion.md` (both chat completion tasks) |
| `references/data-preparation/traces.md` | Trace formats, and converting inference endpoint records |
| `references/job-description.md` | Writing job descriptions per task type |
| `references/configuration.md` | config.yaml parameters, defaults, cross-field validation |
| `references/model-catalog.md` | Teacher and student models, defaults, task compatibility |
| `references/reasoning-models.md` | Everything about reasoning students (`base.enable_thinking`) |
| `references/mutators.md` | Synthetic data diversity controls (`synthgen.mutators`) |
| `references/evaluation-metrics.md` | Metrics per task type, primary metrics, verdicts |
| `references/deployment.md` | Model artifacts, the inference client, local serving |
| `references/inference-endpoints.md` | Inference endpoints: collecting and serving, lifetime, keys, records, the two routes to traces |
| `references/execution/README.md` | Install the CLI, sign in and choose the backend |
| `references/execution/cli.md` | Execution backend: every `distil` command (default) |
| `references/execution/backend-api.md` | Execution backend: every distil labs REST API request |

## Routing

- End-to-end request ("build/train a model for X") → `workflows/build-a-model.md`. Ask what the
  user starts with to pick the entry point: an LLM in production (Step 1), a trace file
  (Step 3) or a labelled dataset (Step 5).
- Endpoint questions ("put an endpoint in front of GPT-4", "download the traces", "put the
  trained model behind the endpoint") → `stages/inference-endpoint.md`.
- Single-stage request ("run a teacher eval", "regenerate the synthetic data") → that stage file
  plus the execution backend in use. If none is set up yet, run
  `references/execution/README.md` § Set up the CLI and § Choose the backend first.
- Results are disappointing ("teacher score is low", "student is far below teacher"), or "keep
  iterating until the model is good" → `workflows/model-iterations.md`.
- Building evals/test sets "help me build evals", "I want to build a test set for my case" 
  or anything about building evals and test sets -> `stages/test-set-from-traces.md`
- Lookup question (a config parameter, a metric, a data format, a command) → the matching
  reference file.
