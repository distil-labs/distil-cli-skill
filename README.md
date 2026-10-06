# distil labs Agent Skill

An [Agent Skill](https://agentskills.io) for building task-specific small language models on the [distil labs](https://distillabs.ai)
platform: from raw data or production traces, through teacher evaluation and synthetic data
generation, to a finetuned, evaluated, deployable student model.

Everything runs on the distil labs platform through the `distil` CLI, or through the REST API
from Python when you prefer it. The only prerequisite is an account. The API backend also needs
`pip install requests pyyaml`, and takes its access tokens from `distil access-token`.

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

The skill runs every stage through the `distil` CLI, installed above. Sign in with
`distil auth`, or `distil signup` without an account; the skill does this itself at the start of
a project if you skip it. On Windows, run the CLI inside WSL.

## Supported Task Types

| Task Type | Use Case | Example |
|-----------|----------|---------|
| Question Answering | Extract answers from documents | Invoice parsing, contract analysis, ticket extraction |
| Classification | Categorize text into fixed classes | Intent detection, sentiment analysis, ticket triage |
| Chat Completion | Conversational turns with text, tool calls, or both | Support chatbots, API routing, chatbot actions |
| Agentic Chat Completion | Tool-calling loops where tool results feed back into the answer | DevOps assistants, file system assistants, database interfaces |

## Layout

`SKILL.md` is the entry point. `stages/` holds one unit of pipeline work each, `workflows/` the
two workflows (the build loop and the improvement loop), and `references/` the shared knowledge,
with every `distil` command in `references/execution/cli.md` and every REST API request in
`references/execution/backend-api.md`. Swapping the CLI for the REST API changes the commands and
nothing else.

## Quick Start

Once the skill is installed, ask your agent to build you a model:

> "Help me build a classification model for customer support intent detection"

The agent starts from the model you already run in production: it puts an inference endpoint in
front of it to collect traces, or takes a trace file or a labelled dataset if you have one. It
then checks feasibility with a teacher evaluation, generates synthetic training data, trains the
student, and serves it behind a new endpoint that takes the traffic the traces came from. Every
stage confirms the setup and the expected credit cost with you before it submits anything.

## Documentation

- [distil labs documentation](https://www.distillabs.ai/docs)
- [CLI reference](https://www.distillabs.ai/docs/reference/cli)

## License

Apache 2.0. See [LICENSE](LICENSE) for details.
