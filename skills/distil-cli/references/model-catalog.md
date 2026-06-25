# Model Catalog

Single source of truth for supported student and teacher models, task compatibility, and default recommendations. Other reference files should link here rather than duplicating these tables.

---

## Student Models

Student models are the small language models that get fine-tuned for your task. 24 models are available across seven families, 135M to 9B parameters.

| Model | Config value | Parameters | Family |
|-------|--------------|------------|--------|
| SmolLM2 135M | `SmolLM2-135M-Instruct` | 135M | SmolLM2 |
| Gemma 3 270M | `gemma-3-270m-it` | 270M | Gemma 3 |
| FunctionGemma 270M | `functiongemma-270m-it` | 270M | Gemma 3 |
| Liquid LFM2 350M | `LFM2-350M` | 350M | Liquid LFM |
| Liquid LFM2.5 350M | `LFM2.5-350M` | 350M | Liquid LFM |
| Qwen3 0.6B | `Qwen3-0.6B` | 0.6B | Qwen3 |
| Qwen3.5 0.8B | `Qwen3.5-0.8B` | 0.8B | Qwen3.5 |
| Gemma 3 1B | `gemma-3-1b-it` | 1B | Gemma 3 |
| Llama 3.2 1B Instruct | `Llama-3.2-1B-Instruct` | 1B | Llama 3 |
| Liquid LFM2 1.2B | `LFM2-1.2B` | 1.2B | Liquid LFM |
| Liquid LFM2.5 1.2B Instruct | `LFM2.5-1.2B-Instruct` | 1.2B | Liquid LFM |
| SmolLM2 1.7B | `SmolLM2-1.7B-Instruct` | 1.7B | SmolLM2 |
| Qwen3 1.7B | `Qwen3-1.7B` | 1.7B | Qwen3 |
| Qwen3.5 2B | `Qwen3.5-2B` | 2B | Qwen3.5 |
| Gemma 4 E2B | `gemma-4-E2B-it` | 2B (effective) | Gemma 4 |
| Liquid LFM2 2.6B | `LFM2-2.6B` | 2.6B | Liquid LFM |
| Llama 3.2 3B Instruct | `Llama-3.2-3B-Instruct` | 3B | Llama 3 |
| Gemma 3 4B | `gemma-3-4b-it` | 4B | Gemma 3 |
| Qwen3 4B | `Qwen3-4B-Instruct-2507` | 4B | Qwen3 |
| Qwen3.5 4B | `Qwen3.5-4B` | 4B | Qwen3.5 |
| Gemma 4 E4B | `gemma-4-E4B-it` | 4B (effective) | Gemma 4 |
| Llama 3.1 8B Instruct | `Llama-3.1-8B-Instruct` | 8B | Llama 3 |
| Qwen3 8B | `Qwen3-8B` | 8B | Qwen3 |
| Qwen3.5 9B | `Qwen3.5-9B` | 9B | Qwen3.5 |

### Size tier guidance

| Size range | When to choose | Typical models |
|------------|----------------|-----------------|
| 135M – 350M | Extreme latency constraints, edge/on-device, very simple tasks | SmolLM2 135M, Gemma 3 270M, LFM2 350M, LFM2.5 350M |
| 0.6B – 1.2B | Cost-sensitive production, well-defined tasks | Qwen3 0.6B, Qwen3.5 0.8B, Gemma 3 1B, Llama 3.2 1B, LFM2 1.2B |
| 1.7B – 3B | Balanced default for most production workloads | SmolLM2 1.7B, Qwen3 1.7B, Qwen3.5 2B, Gemma 4 E2B, LFM2 2.6B, Llama 3.2 3B |
| 4B – 9B | Accuracy-critical, complex reasoning, nuanced outputs | Gemma 3 4B, Qwen3 4B, Qwen3.5 4B, Gemma 4 E4B, Llama 3.1 8B, Qwen3 8B, Qwen3.5 9B |

---

## Teacher Models

Teacher models generate synthetic training data and provide the knowledge distilled into the student.

| Model | Config value | Tool calling | Notes |
|-------|--------------|--------------|-------|
| GPT OSS 120B | `openai.gpt-oss-120b` | ✓ | Default teacher. Strong general-purpose. |
| GPT OSS 120B Thinking | `openai.gpt-oss-120b-thinking` | ✓ | Chain-of-thought variant, medium reasoning effort. |
| GPT OSS 20B | `openai.gpt-oss-20b` | ✓ | Smaller, faster. Useful for quick experimentation. |
| GPT OSS 20B Thinking | `openai.gpt-oss-20b-thinking` | ✓ | Thinking variant of the 20B model. |
| DeepSeek R1 | `deepseek.r1` | — | General-purpose DeepSeek. |
| DeepSeek R1 Thinking | `deepseek.r1-thinking` | — | Thinking variant. |
| DeepSeek V3.1 | `deepseek.v3.1` | — | General-purpose DeepSeek. |
| DeepSeek V3.2 | `deepseek.v3.2` | ✓ | General-purpose DeepSeek. |
| Qwen3 235B A22B | `Qwen3-235B-A22B-Instruct-2507` | ✓ | Large MoE. |
| Qwen3 480B A35B Coder | `Qwen3-480B-A35B-Coder` | — | Coding-focused MoE. |
| Qwen2.5 VL 72B | `Qwen2.5-VL-72B-Instruct` | — | Vision-language. |
| ZAI GLM 5 | `zai.glm-5` | ✓ | High quality. |
| Moonshot Kimi K2 Thinking | `moonshotai.kimi-k2-thinking` | ✓ | Thinking variant. |
| Moonshot Kimi K2.5 | `moonshotai.kimi-k2.5` | ✓ | General-purpose Kimi. |
| MiniMax M2 Thinking | `minimax.minimax-m2-thinking` | ✓ | Thinking variant. |

Tool calling indicates which models can be used for the `tool-calling-closed-book` and `multi-turn-tool-calling-closed-book`. We exclude vision, coder, or models that we have not tested yet.

### Teacher recommendations

- **Default:** `openai.gpt-oss-120b`. Strong general-purpose teacher.
- **Highest quality:** `zai.glm-5`. Richer synthetic data.
- **Code-heavy:** `Qwen3-480B-A35B-Coder`.
- **Fast iteration:** `openai.gpt-oss-20b`.

---

## Task Compatibility

### Student model compatibility

Most task types work with all student models. Tool calling is the exception.

| Task | Supported student families |
|------|----------------------------|
| `question-answering` | All |
| `classification` | All |
| `question-answering-open-book` | All |
| `question-answering-closed-book` | All |
| `tool-calling-closed-book` | Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4 only |
| `multi-turn-tool-calling-closed-book` | Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma, and Gemma 4 only |

**Tool-calling student shortlist:** `Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-4B-Instruct-2507`, `Qwen3-8B`, `Qwen3.5-0.8B`, `Qwen3.5-2B`, `Qwen3.5-4B`, `Qwen3.5-9B`, `Llama-3.2-1B-Instruct`, `Llama-3.2-3B-Instruct`, `Llama-3.1-8B-Instruct`, `LFM2-350M`, `LFM2-1.2B`, `LFM2-2.6B`, `LFM2.5-350M`, `LFM2.5-1.2B-Instruct`, `functiongemma-270m-it`, `gemma-4-E2B-it`, `gemma-4-E4B-it`.

### Teacher model compatibility

Most task types work with all teacher models. Both tool-calling tasks are the exception — they exclude the teachers marked in the **Tool calling** column of [Teacher Models](#teacher-models).

| Task | Teacher constraint |
|------|--------------------|
| `question-answering` | All |
| `classification` | All |
| `tool-calling-closed-book` | See the **Tool calling** column in [Teacher Models](#teacher-models) |
| `question-answering-open-book` | All |
| `question-answering-closed-book` | All |
| `multi-turn-tool-calling-closed-book` | Same as `tool-calling-closed-book` |

---

## Default Recommendations

If the user hasn't specified anything, use these defaults:

| Field | Value |
|-------|-------|
| `student_model_name` | `Llama-3.2-1B-Instruct` |
| `teacher_model_name` | `openai.gpt-oss-120b` |

### Common pairings by use case

| Use case | Student | Teacher |
|----------|---------|---------|
| General starting point | `Llama-3.2-1B-Instruct` | `openai.gpt-oss-120b` |
| High-accuracy production | `Qwen3-8B` or `Llama-3.1-8B-Instruct` | `zai.glm-5` |
| Cost-sensitive / edge | `SmolLM2-135M-Instruct` or `gemma-3-270m-it` | `openai.gpt-oss-120b` |
| Tool calling (single-turn) | `Qwen3-1.7B` or `Llama-3.2-1B-Instruct` | `openai.gpt-oss-120b` |
| Tool calling (multi-turn) | `Qwen3-4B-Instruct-2507` | `openai.gpt-oss-120b` |
| Code-heavy tasks | `Qwen3-8B` | `Qwen3-480B-A35B-Coder` |
| Fast experimentation | `Llama-3.2-1B-Instruct` | `openai.gpt-oss-20b` |

---

## Family Notes

- **Qwen3** (0.6B, 1.7B, 4B, 8B) — Strong general-purpose. Supports tool calling. Good multilingual.
- **Qwen3.5** (0.8B, 2B, 4B, 9B) — Latest Qwen generation. Supports tool calling. Good multilingual.
- **Llama 3** (1B, 3B, 8B) — Well-established Meta family. Supports tool calling.
- **SmolLM2** (135M, 1.7B) — Efficiency-focused. Does not support tool calling.
- **Gemma 3** (270M, 1B, 4B, plus FunctionGemma 270M) — Google's compact models. Base Gemma 3 (270M/1B/4B) does not support tool calling, but the FunctionGemma 270M variant does.
- **Gemma 4** (E2B, E4B) — Google's latest compact models. Supports tool calling.
- **Liquid LFM** (350M, 1.2B, 2.6B across LFM2 and LFM2.5) — Supports tool calling.

---

## Hard Rules

- Never recommend SmolLM2 or base Gemma 3 (`gemma-3-270m-it`, `gemma-3-1b-it`, `gemma-3-4b-it`) for a tool-calling task — the platform rejects it. Supported tool-calling students: Qwen3, Qwen3.5, Llama 3-family, LFM2/LFM2.5, FunctionGemma (`functiongemma-270m-it`), and Gemma 4 (`gemma-4-E2B-it`, `gemma-4-E4B-it`).
- For tool calling (both single-turn `tool-calling-closed-book` and `multi-turn-tool-calling-closed-book`), any teacher works **except** `deepseek.r1`, `deepseek.r1-thinking`, `deepseek.v3.1`, `Qwen3-480B-A35B-Coder`, and `Qwen2.5-VL-72B-Instruct` (see the **Tool calling** column in [Teacher Models](#teacher-models)).
- Reasoning teacher models (the GPT OSS, DeepSeek, GLM, Kimi, and MiniMax families) require a teacher temperature between 0.5 and 0.7 (validation error otherwise).
