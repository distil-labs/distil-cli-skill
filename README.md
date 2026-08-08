# Distil CLI Skill for Claude

An agent skill for building task-specific small language models on the [distil labs](https://distillabs.ai)
platform: from raw data or production traces, through teacher evaluation and synthetic data
generation, to a finetuned, evaluated, deployable student model.

Everything runs on the distil labs platform through the `distil` CLI. The only prerequisite is an
account. Where the CLI cannot be installed, the skill falls back to the REST API and needs
`pip install requests pyyaml` instead.

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

Install the CLI and authenticate:

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
distil auth
```

The skill does this itself at the start of a project if you skip it. On a platform the installer
does not support — Windows without WSL, or any environment that permits no new binary — set
credentials for the API fallback instead:

```bash
export DL_USERNAME="you@example.com"
export DL_PASSWORD="…"
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

## Layout

| Path | What it is |
|---|---|
| `SKILL.md` | Entry point: architecture, stage protocol, routing |
| `stages/` | One unit of pipeline work each, directly invocable |
| `workflows/` | Sequencers over stages, owning the gates and decision points |
| `references/` | Shared knowledge: data formats, config, models, metrics |
| `references/execution/` | The only files that know how stages actually run; its `README.md` picks the backend |

The skill separates *what to do* from *how to run it*. Stages and workflows hold the
model-building logic; the execution backends under `references/execution/` hold the commands and
request shapes. Swapping the CLI for the REST API changes the commands and nothing else.

## Quick Start

Once the skill is installed, just ask Claude to build you a model:

> "Help me build a classification model for customer support intent detection"

Claude asks whether you are starting from a labeled dataset or from production traces, routes you
to the matching workflow, and walks the pipeline with you: preparing the input directory, checking
feasibility with a teacher evaluation, generating synthetic training data, training the student,
and deploying it. Every stage confirms the setup and the expected credit cost with you before it
submits anything, and runs a cheap smoke first where a smoke is worth running.

## Documentation

- [distil labs documentation](https://www.distillabs.ai/docs)
- [CLI reference](https://www.distillabs.ai/docs/getting-started/cli)

## License

Apache 2.0 — see [LICENSE](LICENSE) for details.
