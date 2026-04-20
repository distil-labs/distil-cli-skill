# Getting Started

## Prerequisites

The Distil CLI supports the following platforms:

| Platform             | Supported |
|----------------------|-----------|
| Linux (x86_64)       | Yes       |
| macOS (Intel)        | Yes       |
| macOS (Apple Silicon) | Yes      |
| Windows              | No (use WSL or the REST API) |

## Install the CLI

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
```

Verify the installation by running `distil` with no arguments. The CLI should print the list of available commands.

## Create an Account

Register a new account using the CLI:

```bash
distil register
```

Alternatively, sign up through the web app at [app.distillabs.ai/sign-up](https://app.distillabs.ai/sign-up).

## Log In

Authenticate with the platform:

```bash
distil login
```

This opens a browser for login. Enter the username and password created during registration.

> **`/login` (Claude Code) is NOT `distil login`.** The `/login` slash command in Claude Code authenticates your Claude session. It does NOT authenticate the Distil CLI. If you see "Credit balance is too low" or 401-style errors from `distil` commands, your Distil session has expired. Re-authenticate from a Claude Code prompt with the bang prefix so the command runs in your shell:
>
> ```
> ! distil login
> ```

Verify the currently authenticated user:

```bash
distil whoami
```

Log out when needed:

```bash
distil logout
```

## Update the CLI

Keep the CLI up to date. The platform evolves quickly — run this before starting any new project so new commands and features are available:

```bash
distil update
```

## Quickstart Walkthrough

This walkthrough trains a question-answering model end-to-end. It uses a minimal dataset to show the full flow: create a model, prepare data, upload, evaluate, train, and deploy.

### 1. Create a Model

Register a new model to track the experiment:

```bash
distil model create my-first-model
# Output includes the Model ID (use this for all subsequent commands)
```

List all models at any time:

```bash
distil model list
```

### 2. Choose a Task Type

Select the task type that matches the problem. For this walkthrough, use `question-answering`. The platform supports six task types:

| Task Type                 | Use When                                                        |
|---------------------------|-----------------------------------------------------------------|
| Question Answering        | Solve problems by returning text answers (QA, text transforms)  |
| Classification            | Assign text to categories from a fixed set                      |
| Tool Calling              | Generate structured tool/API calls from natural language         |
| Multi-Turn Tool Calling   | Generate tool calls in multi-turn conversations                 |
| Open Book QA (RAG)        | Answer questions given provided context passages                |
| Closed Book QA            | Answer questions from knowledge learned during training         |

### 3. Prepare Minimal Data

Create a directory (e.g., `./my-data`) containing the following files:

| File                   | Required | Description                                   |
|------------------------|----------|-----------------------------------------------|
| `job_description.json` | Yes      | Task objectives and configuration              |
| `train.csv`            | Yes      | 20+ labeled (question, answer) pairs           |
| `test.csv`             | Yes      | Held-out evaluation set                        |
| `config.yaml`          | Yes      | Task type, student model, and teacher model    |
| `unstructured.csv`     | No       | Domain text for synthetic data generation      |

**job_description.json** -- describe the task clearly:

```json
{
  "task_description": "Extract the key dates mentioned in the input text and return them as a comma-separated list."
}
```

**config.yaml** -- specify the task, student model, and teacher model:

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-1B-Instruct
  teacher_model_name: openai.gpt-oss-120b
```

**train.csv** and **test.csv** -- provide labeled examples in CSV format:

```
question,answer
"The contract was signed on Jan 3 2025 and expires Dec 31 2025.","Jan 3 2025, Dec 31 2025"
```

Include at least 20 examples in train.csv and a separate set in test.csv.

### 4. Upload Data

```bash
distil model upload-data <model-id> --data ./my-data
```

Check upload status:

```bash
distil model upload-status <model-id>
```

### 5. Run Teacher Evaluation

Validate that a large teacher model can solve the task before training the student:

```bash
distil model run-teacher-evaluation <model-id>
```

Check status and results:

```bash
distil model teacher-evaluation <model-id>
```

High teacher accuracy means the task is well-defined. Low accuracy means the job description, data, or config needs revision. Iterate on the data and re-upload until the teacher performs well.

### 6. Train the Student Model

Start the knowledge distillation training pipeline:

```bash
distil model run-training <model-id>
```

Training takes several hours. Monitor progress:

```bash
distil model training <model-id>
```

For the full status list and the canonical polling loop, see `references/tasks/polling-jobs.md`.

### 7. Download and Deploy

Download the trained model:

```bash
distil model download <model-id>
```

Deploy locally (uses llama-cpp as the backend):

```bash
distil model deploy local <model-id>
```

Get the invocation command for the deployed model:

```bash
distil model invoke <model-id>
```

This outputs a ready-to-run command. Copy and execute it to query the model.
