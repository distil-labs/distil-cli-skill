# Distil CLI Skill for Claude

A Claude skill for training task-specific small language models (SLMs) using the [Distil Labs](https://distillabs.ai) CLI and platform.

## What This Skill Does

This skill teaches Claude how to help you:

- **Train specialized SLMs** — Create models up to 70x smaller than large models while maintaining accuracy
- **Prepare training data** — Generate proper data files for classification, QA, tool calling, and multi-turn tool calling tasks
- **Run the Distil CLI** — Execute commands for the full training workflow
- **Deploy models locally or remotely** — Set up your trained models with llama-cpp, vLLM, or Distil-managed infrastructure
- **Train from production traces** — Convert existing LLM logs into fine-tuned small models

## Installation

### Claude Code

```bash
/plugin marketplace add https://github.com/distil-labs/distil-cli-skill
/plugin install distil-cli@distil-cli-skill
```

### Claude.ai / Claude Desktop

1. [Download this repo as ZIP](https://github.com/distil-labs/distil-cli-skill/archive/refs/heads/main.zip) (or click "Code" → "Download ZIP" on GitHub)
2. Go to [claude.ai](https://claude.ai) → Settings → Capabilities → Skills
3. Click "Upload skill" and select the downloaded ZIP file
4. Toggle the skill ON

## Prerequisites

Install the Distil Labs CLI:

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
```

## Supported Task Types

| Task Type | Use Case | Example |
|-----------|----------|---------|
| Question Answering | Extract answers from documents | Invoice parsing, contract analysis, ticket extraction |
| Classification | Categorize text into fixed classes | Intent detection, sentiment analysis, ticket triage |
| Tool Calling | Select and invoke functions/APIs | API routing, workflow automation, chatbot actions |
| Multi-Turn Tool Calling | Multi-step conversations with tool use | DevOps chatbots, file system assistants, database interfaces |
| Open Book QA (RAG) | Answer questions using provided context | Document QA, support from docs |
| Closed Book QA | Answer from knowledge learned during training | FAQ bots, domain assistants |

## Skill Structure

```
distil-cli-skill/
├── SKILL.md                          # Router + core instructions
├── references/
│   ├── getting-started.md            # Install CLI, auth, quickstart
│   ├── platform-overview.md          # What Distil Labs is, concepts, value prop
│   ├── cli-reference.md              # All CLI commands with args and flags
│   ├── task-selection-guide.md       # Choosing the right task type
│   ├── model-catalog.md              # Student + teacher models, compatibility, defaults
│   ├── job-description-guide.md      # Writing job_description.json
│   ├── configuration.md              # Full config.yaml reference
│   ├── mutations-guide.md            # Controlling synthetic data diversity
│   ├── evaluation-metrics.md         # Metrics reference + interpretation
│   ├── api-reference.md              # REST API setup + endpoints
│   └── tasks/
│       ├── prepare-data/
│       │   ├── overview.md
│       │   ├── question-answering.md
│       │   ├── classification.md
│       │   ├── tool-calling.md
│       │   ├── multi-turn-tool-calling.md
│       │   ├── open-book-qa.md
│       │   └── closed-book-qa.md
│       ├── upload-dataset.md
│       ├── upload-and-process-traces.md
│       ├── teacher-evaluation.md      # Incl. canonical verdict thresholds
│       ├── training.md
│       ├── deployment-integration.md
│       ├── retrieve-predictions.md
│       ├── analyze-predictions.md
│       ├── polling-jobs.md            # Canonical polling loop
│       └── verify-auth.md
└── workflows/
    ├── dataset-to-model.md            # E2E: dataset → eval → train → deploy
    ├── traces-to-model.md             # E2E: traces → process → eval → train → deploy
    └── improving-a-model.md           # Iteration: ITERATE/RETHINK/RETUNE/ESCALATE
```

## Quick Start

Once the skill is installed, just ask Claude to help you train a model:

> "Help me train a classification model for customer support intent detection"

Claude will guide you through:
1. Creating a model with `distil model create`
2. Preparing your data files
3. Uploading data and running teacher evaluation
4. Training the model
5. Downloading and deploying

## Documentation

- [Distil Labs Documentation](https://www.distillabs.ai/docs)
- [CLI Reference](https://www.distillabs.ai/docs/getting-started/cli)

## License

Apache 2.0 — see [LICENSE](LICENSE) for details.
