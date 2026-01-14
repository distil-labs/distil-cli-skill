# Distil CLI Skill for Claude

A Claude skill for training task-specific small language models (SLMs) using the [Distil Labs](https://distillabs.ai) CLI.

## What This Skill Does

This skill teaches Claude how to help you:

- **Train specialized SLMs** - Create models up to 70x smaller than large models while maintaining accuracy
- **Prepare training data** - Generate proper data files for classification, QA, and tool calling tasks
- **Run the Distil CLI** - Execute commands for the full training workflow
- **Deploy models locally** - Set up your trained models with Ollama or vLLM

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
| Classification | Categorize text into fixed classes | Intent detection, sentiment analysis |
| QA (RAG/Open-Book) | Answer questions using provided context | Document QA, support from docs |
| QA (Closed-Book) | Answer questions from learned knowledge | FAQ bots, domain assistants |
| Tool Calling | Select and invoke functions | API routing, workflow automation |

## Quick Start

Once the skill is installed, just ask Claude to help you train a model:

> "Help me train a classification model for customer support intent detection"

Claude will guide you through:
1. Creating a model with `distil model create`
2. Preparing your data files
3. Uploading data and running teacher evaluation
4. Training the model
5. Downloading and deploying locally

## Documentation

- [Distil Labs Documentation](https://docs.distillabs.ai)
- [CLI Reference](https://docs.distillabs.ai/getting-started/cli)

## License

Apache 2.0 - see [LICENSE](LICENSE) for details.
