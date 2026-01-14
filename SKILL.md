---
name: distil-cli
description: Train task-specific small language models (SLMs) using the Distil Labs CLI. Helps with data preparation, model training, and deployment.
---

# Distil CLI

Train specialized small language models (SLMs) using the Distil Labs platform. The platform uses knowledge distillation to create models up to 70x smaller than large models while maintaining comparable accuracy.

**What you can help with depends on the environment:**

| Environment | Capabilities |
|-------------|--------------|
| **Claude Code** | Full end-to-end workflow: task selection, data preparation, running CLI commands, training, and deployment |
| **Claude Browser** | Task selection and data preparation only: help users choose the right task type and create job_description.json, config.yaml, train.csv, test.csv files. User runs CLI commands themselves. |

## Instructions

### Prerequisites

Install the CLI and authenticate:
```bash
# Install
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh

# Authenticate (if not already logged in)
distil login
```

Other auth commands: `distil signup` (create account), `distil whoami` (check user), `distil logout`

### Core Workflow

**Step 1: Create a Model**

Register a new model to track your experiment:
```bash
distil model create my-model-name
# Returns: Model ID (use this for all subsequent commands)
```

List all models with `distil model show`.

**Step 2: Task Selection**

Choosing the right task type is crucial. Help the user by asking what they need the model to do:

| If the user needs to... | Choose | Data Guide |
|-------------------------|--------|------------|
| Extract specific facts from documents (invoices, contracts, tickets) | **Question Answering** | `data-question-answering.md` |
| Assign text to categories from a fixed set | **Classification** | `data-classification.md` |
| Generate structured tool/API calls from natural language | **Tool Calling** | `data-tool-calling.md` |
| Answer questions using context passages they provide at inference | **Open Book QA (RAG)** | `data-qa-rag.md` |
| Answer questions from knowledge learned during training (no context at inference) | **Closed Book QA** | `data-qa-closed.md` |

**Question Answering** — Extracts or generates precise answers from text. The input contains both the document and the question. Use when retrieving specific facts from documents.
- *Examples:* "What is the termination clause?" from contracts, "What's the total due?" from invoices, "What was the root cause?" from incident reports

**Classification** — Assigns text to one category from a fixed set. Use when you need deterministic categorization, not open-ended generation.
- *Examples:* Intent detection, content moderation (toxic/safe), sentiment analysis, ticket triage by department

**Tool Calling** — Maps natural language to structured function calls with correct parameters. Use when routing user requests to backend APIs/services.
- *Examples:* Voice assistant commands → smart home APIs, chatbot intents → CRM operations, natural language → database queries
- *Note:* Only supports Llama3 family student models

**Open Book QA (RAG)** — Answers questions using a provided context passage. The model grounds answers in the given text, not general knowledge. Use when you have (or can retrieve) relevant passages at inference time.
- *Examples:* Customer support from product docs, legal document analysis, technical documentation assistants
- *When to pick:* You've chunked documents for a RAG pipeline and want the model to answer strictly from retrieved chunks

**Closed Book QA** — Learns facts from your unstructured data during training and answers without external context at inference. Use when you want knowledge "baked into" the model.
- *Examples:* FAQ bots, domain-specific knowledge assistants
- *When to pick:* You have lots of unstructured data, users shouldn't need to provide context, or RAG retrieval is difficult for your use case

**Step 3: Data Preparation**

Once the task type is selected, read the appropriate data guide and help the user prepare these files:

| File | Required | Description |
|------|----------|-------------|
| `job_description.json` | Yes | Task objectives and configuration |
| `train.csv` or `train.jsonl` | Yes | 20+ labeled (question, answer) pairs |
| `test.csv` or `test.jsonl` | Yes | Held-out evaluation set |
| `config.yaml` | Yes | Task type (defaults work well; see `config.md` for advanced options) |
| `unstructured.csv` | No | Domain text for synthetic data generation |

**Step 4: Upload Data**
```bash
distil model upload-data <model-id> --data ./my-data-folder
```

**Step 5: Teacher Evaluation**

Before training, validate whether a large language model can solve your task. This serves as:
- **Feasibility check**: If the teacher can solve the task, the student model will learn it effectively
- **Performance benchmark**: Teacher accuracy predicts expected SLM performance

```bash
distil model run-teacher-evaluation <model-id>
distil model teacher-evaluation <model-id>  # Check status/results
```

**Interpreting results:**
- High accuracy → Task is well-defined, proceed to training
- Low accuracy → Revise task description, improve data quality, or check for inconsistencies

For details on evaluation metrics (LLM-as-a-Judge, Exact-Match, ROUGE-L, tool_call_equivalence, etc.), see `metrics.md`.

**Step 6: Model Training**

Train your SLM using knowledge distillation:
1. Teacher model generates synthetic training data from your examples
2. Synthetic data is validated for diversity and quality
3. Student model learns from synthetic data with task-specific optimization

```bash
distil model run-training <model-id>
distil model training <model-id>  # Check status
```

Training takes several hours. Statuses: `JOB_PENDING`, `JOB_RUNNING`, `JOB_SUCCEEDED`, `JOB_FAILED`

**If SLM performance is below expectations:**
1. Increase the number of training examples
2. Make task description more specific
3. Modify config parameters (e.g., increase epochs)
4. Try a larger student model

When training completes, compare SLM metrics against teacher metrics. For help interpreting results, see `metrics.md`.

**Step 7: Download and Deploy**
```bash
distil model download <model-id>
```

For local deployment with Ollama or vLLM, read `deployment.md`.

### CLI Reference

```bash
# List all models
distil model show

# Show specific model details
distil model show <model-id>

# Download uploaded data files
distil model download-data <model-id>

# JSON output for scripting
distil model show --output json
```

Command aliases: `distil model` = `distil models` = `distil m`

### Supported Models

**Student Models (what you train):**
Llama 3.2 (1B, 3B), Llama 3.1 8B, SmolLM2 (135M, 1.7B), Gemma 3 (270M, 1B, 4B), Qwen3 (0.6B, 1.7B, 4B, 8B), IBM Granite 3.1/3.3 8B

**Teacher Models (used for distillation):**
DeepSeek R1, V3.1, Qwen3 (235B, 480B), Llama 3.1 405B, 3.3 70B, GPT OSS (20B, 120B)

### Troubleshooting

**Check model status:**
```bash
distil model show <model-id>
```

**Training failed:**
1. Check teacher evaluation results first
2. Verify data format matches task type requirements
3. Ensure sufficient training examples (20+ minimum)

**Authentication issues:**
```bash
distil logout
distil login
```

### Platform Support

- Linux (x86_64): Yes
- macOS (Intel): Yes
- macOS (Apple Silicon): Yes
- Windows: Use WSL or REST API

---

## Examples

### Claude Code (End-to-End Workflow)

In Claude Code, you can run CLI commands directly.

**Example 1: Train a classification model**
```
User: Help me train a model to classify customer support tickets into categories

Claude: I'll help you train a classification model. Let me start by creating a model:

> distil model create support-ticket-classifier

Now let's prepare your data. I'll read `data-classification.md` for the format...

[Claude creates job_description.json, config.yaml, and helps format train.csv/test.csv]

Data files ready. Uploading:

> distil model upload-data abc123 --data ./support-classifier-data

Let's validate with teacher evaluation first:

> distil model run-teacher-evaluation abc123

[Checks status, then proceeds to training when ready]

> distil model run-training abc123
```

**Example 2: Check training status and download**
```
User: What's the status of my model abc123?

Claude: Let me check:

> distil model show abc123
> distil model training abc123

Your training completed successfully! Downloading the model:

> distil model download abc123

Model downloaded. To run it locally with Ollama:

> ollama create support-classifier -f model/Modelfile
> ollama run support-classifier
```

**Example 3: Debug a failed training**
```
User: My training failed, can you help?

Claude: Let me investigate:

> distil model show abc123
> distil model training abc123

I see the issue - teacher evaluation accuracy was low. Let me check your data:

> distil model download-data abc123

[Reviews data files and suggests improvements]
```

---

### Claude Browser (Data Preparation Only)

In the browser, Claude cannot run CLI commands but can help prepare all data files. The user then runs commands themselves.

**Example 1: Prepare classification data**
```
User: I want to build a sentiment classifier for product reviews

Claude: I'll help you prepare the data files. Based on `data-classification.md`, you need:

1. **job_description.json:**
{
  "task_description": "Classify product reviews by sentiment",
  "classes_description": {
    "positive": "Reviews expressing satisfaction, praise, or recommendation",
    "negative": "Reviews expressing dissatisfaction, complaints, or warnings",
    "neutral": "Reviews that are balanced or purely factual"
  }
}

2. **config.yaml:**
task: classification
student_model_name: Llama-3.2-1B-Instruct
...

3. **train.csv format:**
question,answer
"This product exceeded my expectations!",positive
"Broke after two days, waste of money",negative
...

Once you have 20+ examples in train.csv and test.csv, run:
$ distil model create sentiment-classifier
$ distil model upload-data <model-id> --data ./my-data
$ distil model run-teacher-evaluation <model-id>
```

**Example 2: Prepare RAG/QA data**
```
User: I need a model that answers questions about our product documentation

Claude: For RAG-style QA, I'll prepare files per `data-qa-rag.md`:

1. **job_description.json:**
{
  "task_description": "Answer customer questions using provided documentation excerpts"
}

2. **config.yaml:**
task: question-answering-open-book
...

3. **train.csv format** (note the context column):
question,context,answer
"How do I reset my password?","Password Reset: Click forgot password...","Click 'Forgot Password' on the login page..."
...

[Provides complete file templates]

Save these files and run the CLI commands to train.
```

**Example 3: Prepare tool calling data**
```
User: I want a model that can call our internal APIs based on user requests

Claude: For tool calling, I'll prepare files per `data-tool-calling.md`:

1. **job_description.json** with your tool schemas:
{
  "task_description": "Select the correct API endpoint based on user request",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_order_status",
        "description": "Look up order status by order ID",
        "parameters": {...}
      }
    },
    ...
  ]
}

Note: Tool calling requires Llama3 family models.

2. **train.csv:**
question,answer
"Where is my order #12345?","{""name"": ""get_order_status"", ""parameters"": {""order_id"": ""12345""}}"
...

[Provides complete templates]
```
