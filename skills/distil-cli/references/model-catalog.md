# Model Catalog

## Defaults

**Recommended default student: a 4B-class model**, for example `Qwen3.5-4B` or
`Qwen3-4B-Instruct-2507`. Go smaller only when the deployment target demands it. The config
default when `student_model_name` is omitted is `Llama-3.2-1B-Instruct`, so always set the
student explicitly. Default teacher: `openai.gpt-oss-120b`.

The judge and trace-processing models default to `base.teacher_model_name`, but that resolves
once, when the config is first expanded. A config read back for an override already carries
them as explicit values. So switching the teacher on an existing config leaves the judge on the
old model. Change `evaluation.llm_as_a_judge_model_name` and
`trace_processing.teacher_model_name` in the same edit.

## Student models (`base.student_model_name`)

| Family | Values | Tool calling |
|---|---|---|
| Llama 3 | `Llama-3.2-1B-Instruct`, `Llama-3.2-3B-Instruct`, `Llama-3.1-8B-Instruct` | Yes |
| Qwen3 | `Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-4B-Instruct-2507`, `Qwen3-8B` | Yes |
| Qwen3.5 | `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`, `Qwen3.5-9B` | Yes |
| LFM2 / LFM2.5 | `LFM2-350M`, `LFM2-1.2B`, `LFM2-2.6B`, `LFM2.5-350M`, `LFM2.5-1.2B-Instruct` | Yes |
| Gemma 4 | `gemma-4-E2B-it`, `gemma-4-E4B-it` | Yes |
| Qwen3.6 | `Qwen3.6-35B-A3B` | Yes |
| Nemotron | `Nemotron-3.5-Lightning-30B-A3B` | Yes |
| FunctionGemma | `functiongemma-270m-it` | Yes |
| Gemma 3 | `gemma-3-270m-it`, `gemma-3-1b-it`, `gemma-3-4b-it` | No |
| SmolLM2 | `SmolLM2-135M-Instruct`, `SmolLM2-1.7B-Instruct` | No |

### Size tiers

| Size range | When to choose |
|---|---|
| 135M - 350M | Extreme latency constraints, edge/on-device, very simple tasks |
| 0.6B - 1.2B | Cost-sensitive production, well-defined tasks |
| 1.7B - 3B | Tight latency or memory budgets |
| **4B** | **The best default: start here** |
| 8B - 9B | Accuracy-critical, complex reasoning, nuanced outputs |
| 30B - 35B MoE (3B active) | Highest quality; inference cost is that of a 3B model but the weights need the memory of a 30B one |

## Teacher models (`base.teacher_model_name`)

All open-weight. All available on the default provider.

| Family | Values |
|---|---|
| GPT OSS | `openai.gpt-oss-20b`, `openai.gpt-oss-20b-thinking`, `openai.gpt-oss-120b`, `openai.gpt-oss-120b-thinking` |
| DeepSeek | `deepseek.v3.1`, `deepseek.v4-pro`, `deepseek.v4-pro-thinking`, `deepseek.v4-pro-0813-low-thinking`, `deepseek.v4-pro-0813-high-thinking`, `deepseek.v4.1-flash-low-thinking`, `deepseek.v4.1-flash-high-thinking` |
| Qwen | `Qwen3-235B-A22B-Instruct-2507`, `Qwen3-480B-A35B-Coder`, `Qwen2.5-VL-72B-Instruct`, `Qwen3.8-2.4T-A95B-minimal-thinking`, `Qwen3.8-2.4T-A95B-medium-thinking` |
| GLM | `zai.glm-5`, `zai.glm-5-thinking`, `zai.glm-5.2`, `zai.glm-5.2-thinking`, `zai.glm-5.3-low-thinking`, `zai.glm-5.3-high-thinking`, `zai.glm-5.3-flash-low-thinking`, `zai.glm-5.3-flash-high-thinking` |
| Kimi | `moonshotai.kimi-k2.6`, `moonshotai.kimi-k2.6-thinking`, `moonshotai.kimi-k3`, `moonshotai.kimi-k3-low-thinking`, `moonshotai.kimi-k3-max-thinking` |
| Nemotron | `nvidia.nemotron-3-ultra` |

A plain value and its `-thinking` twin point at the same model. The `-thinking` value turns the
reasoning mode on, the plain value turns it off. Newer families name the effort instead of a
toggle: `-low-thinking` / `-high-thinking` (GLM 5.3, DeepSeek V4 Pro 0813, DeepSeek V4.1 Flash),
`-minimal-thinking` / `-medium-thinking` (Qwen3.8), `-low-thinking` / `-max-thinking` (Kimi K3).
`moonshotai.kimi-k3-thinking` is an alias for `moonshotai.kimi-k3-max-thinking`; the platform
accepts it with a rename warning, so write the new name.

All teachers count as reasoning models except `Qwen3-235B-A22B-Instruct-2507`,
`Qwen3-480B-A35B-Coder` and `Qwen2.5-VL-72B-Instruct`. Reasoning models require
`synthgen.teacher_temperature` in [0.5, 0.7].

`openai.gpt-oss-20b` and `openai.gpt-oss-120b` run at `low` reasoning effort. Their `-thinking`
values run at `medium`.

## Reasoning students (`base.enable_thinking`)

`Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-8B`, `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`,
`Qwen3.5-9B`, `Qwen3.6-35B-A3B`, `Nemotron-3.5-Lightning-30B-A3B`. Not `Qwen3-4B-Instruct-2507`.
See `reasoning-models.md`.

## Vision compatibility (`base.visual_task`)

Vision teachers: `moonshotai.kimi-k2.6`, `moonshotai.kimi-k2.6-thinking`, `moonshotai.kimi-k3`,
`moonshotai.kimi-k3-low-thinking`, `moonshotai.kimi-k3-max-thinking`,
`zai.glm-5.3-flash-low-thinking`, `zai.glm-5.3-flash-high-thinking`,
`deepseek.v4.1-flash-low-thinking`, `deepseek.v4.1-flash-high-thinking`, `Qwen2.5-VL-72B-Instruct`.

`visual_task: true` requires one of these in every teacher and judge role:
`base.teacher_model_name`, `trace_processing.teacher_model_name`,
`evaluation.llm_as_a_judge_model_name`, and each entry of
`trace_processing.relabelling_committee_models`. One non-vision model in any role fails config
load. Vision students: `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`, `Qwen3.5-9B`, `gemma-4-E2B-it`,
`gemma-4-E4B-it`.

## Tool-calling compatibility

- **Students**: only Llama 3, Qwen3, Qwen3.5, Qwen3.6, LFM2/LFM2.5, Gemma 4, Nemotron, and
  FunctionGemma. Config validation rejects others for both tool-calling tasks.
- **Teachers**: everything except `deepseek.v3.1`, `Qwen3-480B-A35B-Coder` and
  `Qwen2.5-VL-72B-Instruct`. Config validation enforces this only for
  `multi-turn-tool-calling-closed-book`, and single-turn tool calling validates the student
  alone. Respect the deny-list for both anyway.
- **Chat completion tasks** (`chat-completion`, `chat-completion-agentic`): the same student
  AND teacher restrictions apply whenever the job description declares tools (always, for
  agentic). A tool-free `chat-completion` job has no model restriction.

## LLM providers

`openrouter` (default), `together_ai`, `bedrock`. Selected via the `DISTIL_LIB_LLM_PROVIDER`
env var or the submission script's `--llm-provider` flag. Not every teacher runs on every
provider. Keep the default unless you have a reason to change it.
