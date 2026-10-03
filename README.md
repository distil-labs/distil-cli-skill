# distil labs Agent Skill

An [Agent Skill](https://agentskills.io) for building task-specific small language models on the [distil labs](https://distillabs.ai)
platform: from raw data or production traces, through teacher evaluation and synthetic data
generation, to a finetuned, evaluated, deployable student model.

Everything runs on the distil labs platform through the `distil` CLI. The only prerequisite is an
account.

The skill works with every coding agent that reads the Agent Skills format, including Claude Code,
Codex, Cursor, Gemini CLI, GitHub Copilot and OpenCode.

## Installation

### With the distil CLI

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
distil skill install
```

`distil skill install` puts the skill in `~/.agents/skills`, the shared directory that Codex,
Cursor, GitHub Copilot, Gemini CLI, OpenCode, Cline and many other agents read. It also links the
skill into Claude Code. Run it again to update the skill. `distil skill uninstall` removes it. The command needs `git`. It does not
need Node.

### With the skills CLI

If you have Node, [`skills`](https://github.com/vercel-labs/skills) installs the skill without the
distil CLI:

```bash
npx skills add distil-labs/distil-cli-skill
```

Use this route for an agent that `distil skill install` does not cover, with `--agent <name>`.

## Prerequisites

The skill runs every stage through the `distil` CLI. Install it if you have not, and authenticate:

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
distil auth
```

`distil auth` shows a one-time code and opens your browser to confirm it. Use `distil signup`
instead if you don't have an account yet. The skill does this itself at the start of a project
if you skip it. On Windows, run the CLI inside WSL.

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
| `references/execution/` | The only files that know how stages actually run. Its `README.md` sets up the CLI |

The skill separates *what to do* from *how to run it*. Stages and workflows hold the
model-building logic. The execution backend under `references/execution/` holds the commands.

## Quick Start

Once the skill is installed, ask your agent to build you a model:

> "Help me build a classification model for customer support intent detection"

The agent starts from the model you already run in production: it puts an inference endpoint in
front of it to collect traces, or takes a trace file or a labeled dataset if you have one, and
walks the pipeline with you: preparing the input directory, checking feasibility with a teacher
evaluation, generating synthetic training data, training the student, and serving it behind a
new endpoint so it takes the traffic the traces came from. Every stage confirms the setup and the expected credit cost with you before it
submits anything. Where a smoke run is worth running, it runs a cheap one first.

## Documentation

- [distil labs documentation](https://www.distillabs.ai/docs)
- [CLI reference](https://www.distillabs.ai/docs/reference/cli)

## License

Apache 2.0. See [LICENSE](LICENSE) for details.
