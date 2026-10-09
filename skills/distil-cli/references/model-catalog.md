# Model Catalog

## Defaults

**Default student family: Qwen3.5.** Start with `Qwen3.5-4B`, and go smaller (`Qwen3.5-2B`,
`Qwen3.5-0.8B`) only when the deployment target demands it, or larger (`Qwen3.5-9B`) when
accuracy matters more than cost. The config default when `student_model_name` is omitted is
`Llama-3.2-1B-Instruct`, so always set the student explicitly.

**Default teachers: GLM 5.3.** `zai.glm-5.3-flash-low-thinking` is the small teacher, for simple
tasks and for cost. `zai.glm-5.3-low-thinking` is the large teacher, for everything else. Move
either to its `-high-thinking` value when teacher evaluation shows the task needs more
reasoning. The config default when `teacher_model_name` is omitted is `openai.gpt-oss-120b`, so
always set the teacher explicitly.

**The judge stays fixed.** `evaluation.llm_as_a_judge_model_name` scores every run, so set it
explicitly in the first config (the large teacher, `zai.glm-5.3-low-thinking`, unless the user
chooses another) and never change it: scores from two judges are not comparable. It defaults to
`base.teacher_model_name`, resolved once when the config is first expanded (every Dataset an
expand writes carries the expanded config), so a config read back for an override already
names it, and changing the teacher there leaves the judge as it was.
`trace_processing.teacher_model_name`, the relabelling model, defaults the same way: once
expanded it is set on its own, so a new teacher for relabelling is set in both fields.

## Student models (`base.student_model_name`)

| Family | Values | Tool calling |
|---|---|---|
| Qwen3.5 | `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`, `Qwen3.5-9B` | Yes |
| Llama 3 | `Llama-3.2-1B-Instruct`, `Llama-3.2-3B-Instruct`, `Llama-3.1-8B-Instruct` | Yes |
| Qwen3 | `Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-4B-Instruct-2507`, `Qwen3-8B` | Yes |
| LFM2 / LFM2.5 | `LFM2-350M`, `LFM2-1.2B`, `LFM2-2.6B`, `LFM2.5-350M`, `LFM2.5-1.2B-Instruct` | Yes |
| Gemma 4 | `gemma-4-E2B-it`, `gemma-4-E4B-it` | Yes |
| Qwen3.6 | `Qwen3.6-35B-A3B` | Yes |
| Qwen3.8 | `Qwen3.8-27B` (QLoRA only) | Yes |
| FunctionGemma | `functiongemma-270m-it` | Yes |
| Gemma 3 | `gemma-3-270m-it`, `gemma-3-1b-it`, `gemma-3-4b-it` | No |
| SmolLM2 | `SmolLM2-135M-Instruct`, `SmolLM2-1.7B-Instruct` | No |

`Qwen3.8-27B` can only be trained with `tuning.use_qlora: true`. Config load does not check
this, so set it in every config that trains it.

### Size tiers

| Size range | When to choose |
|---|---|
| 135M - 350M | Extreme latency constraints, edge/on-device, very simple tasks |
| 0.6B - 1.2B | Cost-sensitive production, well-defined tasks |
| 1.7B - 3B | Tight latency or memory budgets |
| **4B** | **The best default: start here** |
| 8B - 9B | Accuracy-critical, complex reasoning, nuanced outputs |
| 27B dense | Highest quality from a dense model; needs a large GPU to serve |
| 35B MoE (3B active) | Highest quality; inference cost is that of a 3B model but the weights need the memory of a 35B one |

## Teacher models (`base.teacher_model_name`)

All open-weight. All available on the default provider.

| Family | Values |
|---|---|
| GPT OSS | `openai.gpt-oss-20b`, `openai.gpt-oss-20b-thinking`, `openai.gpt-oss-120b`, `openai.gpt-oss-120b-thinking` |
| DeepSeek | `deepseek.v3.1`, `deepseek.v4-pro`, `deepseek.v4-pro-thinking`, `deepseek.v4-pro-0813-low-thinking`, `deepseek.v4-pro-0813-high-thinking`, `deepseek.v4.1-flash-low-thinking`, `deepseek.v4.1-flash-high-thinking` |
| Qwen | `Qwen3-235B-A22B-Instruct-2507`, `Qwen3-480B-A35B-Coder`, `Qwen2.5-VL-72B-Instruct`, `Qwen3.8-2.4T-A95B-minimal-thinking`, `Qwen3.8-2.4T-A95B-medium-thinking` |
| GLM | `zai.glm-5.3-low-thinking`, `zai.glm-5.3-high-thinking`, `zai.glm-5.3-flash-low-thinking`, `zai.glm-5.3-flash-high-thinking` |
| Kimi | `moonshotai.kimi-k2.6`, `moonshotai.kimi-k2.6-thinking`, `moonshotai.kimi-k3`, `moonshotai.kimi-k3-low-thinking`, `moonshotai.kimi-k3-max-thinking` |

A plain value and its `-thinking` twin point at the same model: the `-thinking` value turns the
reasoning mode on, the plain value turns it off. Some families name the effort instead:
`-low-thinking` / `-high-thinking` (GLM 5.3, DeepSeek V4 Pro 0813, DeepSeek V4.1 Flash),
`-minimal-thinking` / `-medium-thinking` (Qwen3.8), `-low-thinking` / `-max-thinking` (Kimi K3).

All teachers are reasoning models except `Qwen3-235B-A22B-Instruct-2507`,
`Qwen3-480B-A35B-Coder` and `Qwen2.5-VL-72B-Instruct`, which matters for
`synthgen.teacher_temperature` (`configuration.md` § Cross-field validation).

`openai.gpt-oss-20b` and `openai.gpt-oss-120b` run at `low` reasoning effort. Their `-thinking`
values run at `medium`.

## Reasoning students (`base.enable_thinking`)

The students that support it: `reasoning-models.md`.

## Vision compatibility (`base.visual_task`)

Vision teachers: `moonshotai.kimi-k2.6`, `moonshotai.kimi-k2.6-thinking`, `moonshotai.kimi-k3`,
`moonshotai.kimi-k3-low-thinking`, `moonshotai.kimi-k3-max-thinking`,
`zai.glm-5.3-flash-low-thinking`, `zai.glm-5.3-flash-high-thinking`,
`deepseek.v4.1-flash-low-thinking`, `deepseek.v4.1-flash-high-thinking`, `Qwen2.5-VL-72B-Instruct`.

`visual_task: true` requires one of these in every teacher and judge role:
`base.teacher_model_name`, `trace_processing.teacher_model_name`,
`evaluation.llm_as_a_judge_model_name`, and each entry of
`trace_processing.relabelling_committee_models`. One non-vision model in any role fails config
load. Vision students: `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`, `Qwen3.5-9B`, `Qwen3.8-27B`,
`gemma-4-E2B-it`, `gemma-4-E4B-it`.

## Tool-calling compatibility

Applies to the chat completion tasks whenever the job description declares tools (always,
for `chat-completion-agentic`). A tool-free `chat-completion` job has no model restriction.

- **Students**: only Llama 3, Qwen3, Qwen3.5, Qwen3.6, Qwen3.8, LFM2/LFM2.5, Gemma 4 and
  FunctionGemma.
- **Teachers**: everything except `deepseek.v3.1`, `Qwen3-480B-A35B-Coder` and
  `Qwen2.5-VL-72B-Instruct`.


