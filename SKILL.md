---
name: distil-cli
description: Train task-specific small language models (SLMs) using the Distil Labs CLI. Helps with data preparation, model training, and deployment.
---

# Distil Labs CLI Skill

Use this skill to help users train specialized small language models (SLMs) using the Distil Labs platform. The platform uses knowledge distillation to create models up to 70x smaller than large models while maintaining comparable accuracy.

---

## Usage Examples

### Claude Code (End-to-End Workflow)

In Claude Code, you can run CLI commands directly. Here are example conversations:

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

---

## Installation

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
```

## Authentication

```bash
# Create account
distil signup

# Login (opens browser)
distil login

# Check current user
distil whoami

# Logout
distil logout
```

## Core Workflow

The typical workflow has 6 steps:

### 1. Create a Model

```bash
distil model create my-model-name
# Returns: Model ID (use this for all subsequent commands)
```

### 2. Prepare Data Files

Every training job requires these files in a directory:

| File | Required | Description |
|------|----------|-------------|
| `job_description.json` | Yes | Task definition and objectives |
| `train.csv` or `train.jsonl` | Yes | Training examples (20+ minimum) |
| `test.csv` or `test.jsonl` | Yes | Held-out evaluation set |
| `config.yaml` | Yes | Training configuration |
| `unstructured.csv` | Optional | Domain text for synthetic data generation |

**Important:** Data format varies by task type. Read the appropriate data preparation guide:

- **Classification** (intent detection, sentiment, categorization): Read `data-classification.md`
- **Question Answering with RAG** (context-based, document QA): Read `data-qa-rag.md`
- **Question Answering Closed-Book** (model learns facts internally): Read `data-qa-closed.md`
- **Tool Calling** (function calling, API routing): Read `data-tool-calling.md`

### 3. Upload Data

```bash
# Upload entire directory (recommended)
distil model upload-data <model-id> --data ./my-data-folder

# Or upload files individually
distil model upload-data <model-id> \
  --job-description job_description.json \
  --train train.csv \
  --test test.csv \
  --config config.yaml \
  --unstructured unstructured.csv
```

### 4. Run Teacher Evaluation

Validates that a large model can solve your task before training:

```bash
# Start evaluation
distil model run-teacher-evaluation <model-id>

# Check status and results
distil model teacher-evaluation <model-id>
```

**Interpreting results:**
- High teacher accuracy → Task is well-defined, proceed to training
- Low teacher accuracy → Refine task description, improve data quality, or check for inconsistencies

### 5. Train the Model

```bash
# Start training
distil model run-training <model-id>

# Check status
distil model training <model-id>
```

**Training statuses:**
- `JOB_PENDING` - Waiting to start
- `JOB_RUNNING` - Currently training
- `JOB_SUCCEEDED` - Complete
- `JOB_FAILED` - Error occurred

Training typically takes several hours.

### 6. Download Trained Model

```bash
distil model download <model-id>
```

Downloads an Ollama-ready package with:
- `model/` - Model weights
- `model-adapters/` - LoRA adapters
- `Modelfile` - For Ollama integration
- `model_client.py` - Inference script
- `README.md` - Usage instructions

## CLI Reference

### Model Management

```bash
# List all models
distil model show

# Show specific model details (includes Upload ID, Training ID, etc.)
distil model show <model-id>

# Download uploaded data files
distil model download-data <model-id>
```

### JSON Output

Add `--output json` to any command for machine-readable output:

```bash
distil model show --output json
distil model training <model-id> --output json
```

### Command Aliases

- `distil model` = `distil models` = `distil m`

## Local Deployment

### Option 1: Ollama

```bash
# Install Ollama from https://ollama.com/

# Create and run model
ollama create my-model -f model/Modelfile
ollama run my-model
```

Query via API:
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
response = client.chat.completions.create(
    model="my-model",
    messages=[{"role": "user", "content": "Your question"}]
)
```

### Option 2: vLLM

```bash
pip install vllm openai
vllm serve model --api-key EMPTY
```

Query via API:
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
response = client.chat.completions.create(
    model="model",
    messages=[{"role": "user", "content": "Your question"}]
)
```

### Using the Provided Client

```bash
python model_client.py --question "Your question here"

# For QA tasks with context
python model_client.py --question "Your question" --context "Your context"
```

## Supported Models

### Student Models (what you train)
- Llama 3.2 (1B, 3B), Llama 3.1 8B
- SmolLM2 (135M, 1.7B)
- Gemma 3 (270M, 1B, 4B)
- Qwen3 (0.6B, 1.7B, 4B, 8B)
- IBM Granite 3.1/3.3 8B

### Teacher Models (used for distillation)
- DeepSeek R1, V3.1
- Qwen3 (235B, 480B variants)
- Llama 3.1 405B, 3.3 70B
- GPT OSS (20B, 120B)

## Troubleshooting

### Check model status
```bash
distil model show <model-id>
```

### Training failed
1. Check teacher evaluation results first
2. Verify data format matches task type requirements
3. Ensure sufficient training examples (20+ minimum)

### Authentication issues
```bash
distil logout
distil login
```

## Platform Support

- Linux (x86_64): Yes
- macOS (Intel): Yes
- macOS (Apple Silicon): Yes
- Windows: Use WSL or REST API
