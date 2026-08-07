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

Sign up using the CLI:

```bash
distil signup
```

This opens your browser to sign up. Once you finish, you are logged in — there is no separate login step. Alternatively, sign up through the web app at [app.distillabs.ai/sign-up](https://app.distillabs.ai/sign-up).

## Log In

If you already have an account, authenticate with:

```bash
distil auth
```

This opens your browser to sign in, then hands the session back to the CLI (`distil login` is an alias). For headless environments, pass credentials directly with `distil auth --email <email> --password <password>` to skip the browser.

> **`/login` (Claude Code) is NOT `distil auth`.** The `/login` slash command in Claude Code authenticates your Claude session. It does NOT authenticate the Distil CLI. If you see "Credit balance is too low" or 401-style errors from `distil` commands, your Distil session has expired. Re-authenticate from a Claude Code prompt with the bang prefix so the command runs in your shell:
>
> ```
> ! distil auth
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

This walkthrough trains a question-answering model end-to-end. It uses a minimal dataset to show the full flow: prepare data, upload, evaluate, generate a training dataset, train, and serve.

Each step prints an ID that the next step needs. Note them as you go for convenience, though nothing is lost if you don't — every entity records the one it came from, so `distil <entity> show <id>` recovers the lineage later:

```
<upload-id> -> <teacher-evaluation-id>                              (feasibility check)
<upload-id> -> <training-dataset-id> -> <slm-id> -> <deployment-id>  (the model itself)
```

### 1. Choose a Task Type

Select the task type that matches the problem. For this walkthrough, use `question-answering`. The platform supports six task types:

| Task Type                 | Use When                                                        |
|---------------------------|-----------------------------------------------------------------|
| Question Answering        | Solve problems by returning text answers (QA, text transforms)  |
| Classification            | Assign text to categories from a fixed set                      |
| Tool Calling              | Generate structured tool/API calls from natural language         |
| Multi-Turn Tool Calling   | Generate tool calls in multi-turn conversations                 |
| Open Book QA (RAG)        | Answer questions given provided context passages                |
| Closed Book QA            | Answer questions from knowledge learned during training         |

### 2. Prepare Minimal Data

Create a directory (e.g., `./my-data`) containing the following files:

| File                   | Required | Description                                   |
|------------------------|----------|-----------------------------------------------|
| `job_description.json` | Yes      | Task objectives and configuration              |
| `train.jsonl`          | Yes      | 20+ labeled examples, each a `messages` conversation |
| `test.jsonl`           | Yes      | Held-out evaluation set                        |
| `config.yaml`          | Yes      | Task type, student model, and teacher model    |
| `unstructured.jsonl`   | No       | Domain text for synthetic data generation      |

**job_description.json** -- describe the task clearly:

```json
{
  "task_description": "Extract the key dates mentioned in the input text and return them as a comma-separated list.",
  "input_description": "Free-form text (e.g., contract clauses, emails, meeting notes) that may mention one or more dates in formats like 'Jan 3 2025', '2025-01-03', or '3rd of January, 2025'."
}
```

Note: `input_description` is required for `question-answering` only — the platform rejects QA uploads without it, and synthgen reads it (instead of `task_description`) to generate new inputs. Other task types use different fields (`classes_description`, `tools`, unstructured `context`); see `references/job-description-guide.md`.

**config.yaml** -- specify the task, student model, and teacher model:

```yaml
base:
  task: question-answering
  student_model_name: Llama-3.2-1B-Instruct
  teacher_model_name: openai.gpt-oss-120b
```

**train.jsonl** and **test.jsonl** -- provide labeled examples, each a `messages` conversation with a `user` turn (input) and an `assistant` turn (expected output):

```json
{"messages": [{"role": "user", "content": "The contract was signed on Jan 3 2025 and expires Dec 31 2025."}, {"role": "assistant", "content": "Jan 3 2025, Dec 31 2025"}]}
```

Include at least 20 examples in the train file and a separate set in the test file.

### 3. Upload the Data

```bash
distil upload create --data ./my-data
# Output: Upload successful. Upload ID: <upload-id>
```

Check upload status:

```bash
distil upload status <upload-id>
```

Uploads from local files are complete the moment they are created. Trace-derived uploads process asynchronously — see `references/tasks/upload-and-process-traces.md`.

### 4. Run Teacher Evaluation

Validate that a large teacher model can solve the task before spending credits on the student:

```bash
distil teacher-evaluation create-from-upload <upload-id>
# Output: Teacher evaluation started. Teacher Evaluation ID: <teacher-evaluation-id>
```

Check status, then read the scores:

```bash
distil teacher-evaluation status <teacher-evaluation-id>
distil teacher-evaluation metrics <teacher-evaluation-id> --output json | jq '.teacher_performance'
```

High teacher accuracy means the task is well-defined. Low accuracy means the job description, data, or config needs revision. Iterate on the data and create a new upload until the teacher performs well.

### 5. Generate the Training Dataset

Synthetic data generation expands your handful of examples into a full training set. Costs 2 credits.

```bash
distil training-dataset create-from-upload <upload-id>
# Output: Synthetic data generation started. Training Dataset ID: <training-dataset-id>
distil training-dataset status <training-dataset-id>
```

Look at what was generated before you train on it — this is free and takes seconds:

```bash
distil training-dataset sample <training-dataset-id>
```

### 6. Train the Student Model

```bash
distil slm create-from-training-dataset <training-dataset-id>
# Output: Training started. SLM ID: <slm-id>
```

Training takes several hours. Monitor progress:

```bash
distil slm status <slm-id>
```

When it finishes, compare the tuned student against the untuned baseline:

```bash
distil slm metrics <slm-id> --output json | jq '.tuned_model_performance, .base_model_performance'
```

For the full status list and the canonical polling loop, see `references/tasks/polling-jobs.md`.

### 7. Serve the Model

Host it on Distil Labs infrastructure:

```bash
distil deployment create-from-slm <slm-id>
# Output: Deployment started. Deployment ID: <deployment-id>

distil deployment status <deployment-id>      # wait for endpoint_status == "running"
distil deployment endpoint <deployment-id>    # prints the URL and API key
```

The endpoint is OpenAI-compatible, so any OpenAI client library works against it. Shut it down when you are done, so it stops consuming credits:

```bash
distil deployment delete <deployment-id>
```

Or download the model and run it yourself:

```bash
distil slm download <slm-id> --destination ./my-slm    # model.tar + config.yaml
```

See `references/tasks/deployment-integration.md` for serving with vLLM, llama-cpp, or Ollama.
