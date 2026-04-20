---
name: distil-cli
version: 4.0.0
description: >
  Train task-specific small language models (SLMs) using the Distil Labs CLI and platform.
  Activate this skill when the user asks about: distil labs, distil CLI, the distil command,
  training small language models, knowledge distillation, creating SLMs from production traces,
  data preparation for model training, fine-tuning student models from teacher models,
  deploying trained SLMs, model evaluation metrics, task-specific model training,
  classification/QA/tool-calling model training, synthetic data generation for training,
  or any question about the distil platform, its configuration, or its workflows.
  Also activate when the user mentions: distil model, distil login, distil register,
  uploading training data, teacher evaluation, model retuning, model deployment with llama-cpp or vLLM,
  or when working with config.yaml / job_description.json / train.csv files for model training.
---

# Distil CLI

## Environment

| Environment | What you can do |
|---|---|
| **Claude Code** | Full end-to-end workflow: run CLI commands, prepare data, train, deploy |
| **Claude.ai (browser)** | Data preparation guidance only: help choose task types, create config files, format datasets. User runs CLI commands themselves. |

**Windows users:** The Distil CLI does not support Windows natively. Without WSL, users won't get the full experience — CLI commands won't run. They can fall back to the REST API via `references/api-reference.md`, or run the CLI inside WSL. Call this out early if the user is on Windows.

## Default to Workflows for Training Tasks

If the user wants to train a model end-to-end (any phrasing — "help me train a model", "build me an SLM", "I want to fine-tune for X"), do NOT answer ad hoc from the reference files. Instead:

1. Ask one disambiguating question: **"Do you have a labeled dataset to start from, or production traces from an existing LLM application?"**
2. Load the matching workflow:
   - **Dataset** → `workflows/dataset-to-model.md`
   - **Traces** → `workflows/traces-to-model.md`
3. Follow the workflow step-by-step. The workflow tells you which references to read at each step.

The workflows encode the right sequence, decision points, and checkpoints (e.g., confirming config before kicking off a 6+ hour training job). Skipping the workflow and answering Q&A-style usually leads to skipped steps and missing analysis reports.

For specific lookup questions ("what's the API endpoint for X", "what does parameter Y mean", "how do I download predictions"), continue to use the routing table below — those don't need a workflow.

## Intent Detection

Route the user to the right reference file based on their intent. Read the referenced file BEFORE answering.

**If multiple intents apply, read multiple files.** For data preparation, always read both the overview and the task-specific file.

### End-to-End Workflows (Use These First)

| User intent | Read this file |
|---|---|
| "Help me train a model" / "I want to build an SLM" / "Train a model for X" / "Fine-tune for Y" | First ask: dataset or traces? Then load the matching workflow below. |
| "Train from a dataset" / "dataset to model" / end-to-end with CSV data | `workflows/dataset-to-model.md` |
| "Train from production traces" / "traces to model" / end-to-end with traces | `workflows/traces-to-model.md` |
| "Model isn't good enough" / "How do I improve?" / "Retune" / iteration / "Teacher eval scored too low" | `workflows/improving-a-model.md` |

### Getting Started & Platform

| User intent | Read this file |
|---|---|
| "I'm new" / "How do I get started?" / "Install the CLI" | `references/getting-started.md` |
| "What is distil labs?" / "What can it do?" / "How does it work?" | `references/platform-overview.md` |
| "How do I use command X?" / "What CLI commands are available?" | `references/cli-reference.md` |
| "How do I use the API?" / "REST API" / "API authentication" / "programmatic access" | `references/api-reference.md` |

### Task & Model Selection

| User intent | Read this file |
|---|---|
| "What task type should I pick?" / "Classification or QA?" / describing their use case | `references/task-selection-guide.md` |
| "Which model should I use?" / "What student/teacher models are available?" / "Can I use model X?" | `references/model-catalog.md` |
| "How do I write a good task/input description?" / "What goes in job_description.json?" | `references/job-description-guide.md` |

### Data Preparation

Always read `references/tasks/prepare-data/overview.md` first, then the task-specific file.

| User intent | Read this file (after overview.md) |
|---|---|
| General data prep / "What files do I need?" | `references/tasks/prepare-data/overview.md` (just this one) |
| Question answering data | `references/tasks/prepare-data/question-answering.md` |
| Classification data | `references/tasks/prepare-data/classification.md` |
| Tool calling data | `references/tasks/prepare-data/tool-calling.md` |
| Multi-turn tool calling data | `references/tasks/prepare-data/multi-turn-tool-calling.md` |
| Open book QA / RAG data | `references/tasks/prepare-data/open-book-qa.md` |
| Closed book QA data | `references/tasks/prepare-data/closed-book-qa.md` |

### Pipeline Steps

| User intent | Read this file |
|---|---|
| "How do I upload my dataset?" / "upload-data command" | `references/tasks/upload-dataset.md` |
| "How do I use production traces?" / "upload-traces" / "reprocess traces" | `references/tasks/upload-and-process-traces.md` |
| "How do I run teacher evaluation?" / "Is my task feasible?" | `references/tasks/teacher-evaluation.md` |
| "How do I train?" / "Start training" / "Training status" | `references/tasks/training.md` |
| "How do I deploy?" / "Download model" / "Run inference" | `references/tasks/deployment-integration.md` |
| "distil login" / "Credit balance is too low" / 401 errors / "/login doesn't work for distil" | `references/tasks/verify-auth.md` |
| "Download predictions" / "per-example results" / "inspect model outputs" | `references/tasks/retrieve-predictions.md` |
| "Analyze predictions" / "write analysis report" / "compare teacher and student" | `references/tasks/analyze-predictions.md` |
| "How do I poll a long-running job?" / "Wait for training" / "grep status doesn't work" | `references/tasks/polling-jobs.md` |

### Configuration & Metrics

| User intent | Read this file |
|---|---|
| "How do I configure training?" / "config.yaml" / "tuning parameters" | `references/configuration.md` |
| "What are mutations?" / "Seed data doesn't cover all scenarios" / "Improve on specific domains" | `references/mutations-guide.md` |
| "What do these metrics mean?" / "How do I interpret results?" | `references/evaluation-metrics.md` |

## Quick Platform Summary

The core flow is: **create model** → **prepare data** → **upload** → **teacher evaluation** → **train** → **deploy**. Teacher evaluation is a feasibility check — if the teacher can solve the task, the student will learn it. Training takes several hours and produces a downloadable model you can deploy locally (llama-cpp, vLLM) or via the platform.

For the longer write-up of what Distil Labs is and why, read `references/platform-overview.md`. For the list of supported task types, student models, and teacher models — always read `references/model-catalog.md` before recommending a specific model; availability changes over time.

## Quickstart Checklist

Minimum steps to go from zero to a trained model. Read the relevant reference files for details on each step.

```bash
# 1. Install / update and authenticate
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh   # first-time install
distil update                                                  # if already installed — the platform evolves quickly
distil login

# 2. Create a model
distil model create my-model-name
# Note the model ID from the output

# 3. Prepare data files in a directory:
#    - job_description.json  (task objectives)
#    - config.yaml           (task type, student model, teacher model)
#    - train.csv             (20+ labeled examples)
#    - test.csv              (held-out evaluation set)

# 4. Upload data
distil model upload-data <model-id> --data ./my-data-dir

# 5. Run teacher evaluation (feasibility check)
distil model run-teacher-evaluation <model-id>
distil model teacher-evaluation <model-id>  # Check results
# Status values: JOB_NOT_STARTED, JOB_PENDING, JOB_RUNNING, JOB_SUCCESS, JOB_FAILURE, JOB_STOPPED

# 6. Train (long-running — confirm config with user before starting)
distil model run-training <model-id>
distil model training <model-id>  # Check status

# 7. Download and deploy
distil model download <model-id>
distil model deploy local <model-id>
distil model invoke <model-id>  # Get the curl command to query your model
```

**Alternative: Train from traces** — Instead of steps 3-4, prepare a `traces.jsonl` file with your production logs, a `job_description.json`, and a `config.yaml`, then run `distil model upload-traces <model-id> --data ./my-traces-dir`. See `references/tasks/upload-and-process-traces.md` for trace formats and task compatibility (note: `question-answering-open-book` is not supported via traces).

## Instructions

### Before Answering Any Question

1. Identify the user's intent using the routing table above.
2. Read the referenced file(s). Do not answer from memory alone — the reference files contain exact formats, constraints, and edge cases.
3. If the user's intent spans multiple topics (e.g., "help me prepare classification data and train"), read all relevant files.

### Deterministic-First Principle

When helping users, exhaust all mechanical/lookup steps before engaging judgment. Check model compatibility constraints, file format requirements, and parameter defaults from the reference files *first*. Only then apply judgment for task selection, data quality assessment, or configuration tuning.

### Data Preparation Rules

- Always read `references/tasks/prepare-data/overview.md` before any task-specific data file. The overview contains shared requirements (directory structure, min examples, file formats) that the task files assume you know.
- Ask the user what task type they need before preparing data. If unclear, read `references/task-selection-guide.md` and help them decide.
- Ask what student and teacher models they want. If unsure, read `references/model-catalog.md`. Default recommendation: `Llama-3.2-1B-Instruct` as student, `openai.gpt-oss-120b` as teacher.
- Before writing `job_description.json`, read `references/job-description-guide.md` for what makes each field good — `input_description` self-containment is the most common footgun.

### In Claude Code

- Run CLI commands directly. Do not just tell the user what to run.
- After running `distil model create`, capture the model ID and use it in subsequent commands.
- Check status commands (`upload-status`, `teacher-evaluation`, `training`) to monitor progress.
- When training or evaluation is running, tell the user approximately how long it takes and suggest checking back.
- **Polling long-running jobs:** Copy the canonical polling loop from `references/tasks/polling-jobs.md` verbatim. Do not write your own grep loop or `sleep N && command` chain — the former picks the wrong status pattern half the time and the latter is blocked by Claude Code. Use `while ...; do ...; sleep 60; done` with the sleep inside the loop body.
- **Status checks always use `--output json | jq`** — never grep human-readable output. The default text output also omits some metrics (notably LLM-as-a-Judge), so for analysis always use `--output json` too.

### In Claude.ai (Browser)

- Provide complete, copy-pasteable file contents (job_description.json, config.yaml, train.csv, test.csv).
- List the CLI commands the user should run in order, with their model ID placeholder.
- Explain what each command does and what to look for in the output.

### When the User Wants to Improve a Model

Read `workflows/improving-a-model.md`. It covers both iteration cases — teacher eval below thresholds, and training results that don't pass the DEPLOY bar — and consolidates the levers (job description, data, synthgen/mutations, student/teacher choice, tuning parameters, retune).

### Command Aliases

`distil model` = `distil models` = `distil m` — all three work identically.
