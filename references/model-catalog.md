# Model Catalog

Single source of truth for supported student and teacher models, task compatibility, and default recommendations. Other reference files should link here rather than duplicating these tables.

---

## Student Models

Student models are the small language models that get fine-tuned for your task. 20 models are available across six families, 135M to 8B parameters.

| Model | Config value | Parameters | Family |
|-------|--------------|------------|--------|
| SmolLM2 135M | `SmolLM2-135M-Instruct` | 135M | SmolLM2 |
| Gemma 3 270M | `gemma-3-270m-it` | 270M | Gemma 3 |
| FunctionGemma 270M | `functiongemma-270m-it` | 270M | Gemma 3 |
| Liquid LFM2 350M | `LFM2-350M` | 350M | Liquid LFM |
| Liquid LFM2.5 350M | `LFM2.5-350M` | 350M | Liquid LFM |
| Qwen3 0.6B | `Qwen3-0.6B` | 0.6B | Qwen3 |
| Gemma 3 1B | `gemma-3-1b-it` | 1B | Gemma 3 |
| Llama 3.2 1B Instruct | `Llama-3.2-1B-Instruct` | 1B | Llama 3 |
| Liquid LFM2 1.2B | `LFM2-1.2B` | 1.2B | Liquid LFM |
| Liquid LFM2.5 1.2B Instruct | `LFM2.5-1.2B-Instruct` | 1.2B | Liquid LFM |
| SmolLM2 1.7B | `SmolLM2-1.7B-Instruct` | 1.7B | SmolLM2 |
| Qwen3 1.7B | `Qwen3-1.7B` | 1.7B | Qwen3 |
| Liquid LFM2 2.6B | `LFM2-2.6B` | 2.6B | Liquid LFM |
| Llama 3.2 3B Instruct | `Llama-3.2-3B-Instruct` | 3B | Llama 3 |
| Gemma 3 4B | `gemma-3-4b-it` | 4B | Gemma 3 |
| Qwen3 4B | `Qwen3-4B-Instruct-2507` | 4B | Qwen3 |
| Llama 3.1 8B Instruct | `Llama-3.1-8B-Instruct` | 8B | Llama 3 |
| Qwen3 8B | `Qwen3-8B` | 8B | Qwen3 |
| IBM Granite 3.1 8B | `granite-3.1-8b-instruct` | 8B | Granite |
| IBM Granite 3.3 8B | `granite-3.3-8b-instruct` | 8B | Granite |

### Size tier guidance

| Size range | When to choose | Typical models |
|------------|----------------|-----------------|
| 135M – 350M | Extreme latency constraints, edge/on-device, very simple tasks | SmolLM2 135M, Gemma 3 270M, LFM2 350M, LFM2.5 350M |
| 0.6B – 1.2B | Cost-sensitive production, well-defined tasks | Qwen3 0.6B, Gemma 3 1B, Llama 3.2 1B, LFM2 1.2B |
| 1.7B – 3B | Balanced default for most production workloads | SmolLM2 1.7B, Qwen3 1.7B, LFM2 2.6B, Llama 3.2 3B |
| 4B – 8B | Accuracy-critical, complex reasoning, nuanced outputs | Gemma 3 4B, Qwen3 4B, Llama 3.1 8B, Qwen3 8B, Granite 3.x |

---

## Teacher Models

Teacher models generate synthetic training data and provide the knowledge distilled into the student.

| Model | Config value | Notes |
|-------|--------------|-------|
| GPT OSS 120B | `openai.gpt-oss-120b` | Default teacher. Strong general-purpose. Supports tool calling (single- and multi-turn). |
| GPT OSS 120B Thinking | `openai.gpt-oss-120b-thinking` | Chain-of-thought variant, medium reasoning effort. Supports tool calling (single- and multi-turn). |
| GPT OSS 20B | `openai.gpt-oss-20b` | Smaller, faster. Useful for quick experimentation. |
| GPT OSS 20B Thinking | `openai.gpt-oss-20b-thinking` | Thinking variant of the 20B model. |
| DeepSeek R1 | `deepseek.r1` | Reasoning model. Temperature must be 0.5–0.7 (enforced). |
| DeepSeek R1 Thinking | `deepseek.r1-thinking` | Thinking variant of R1. Temperature must be 0.5–0.7 (enforced). |
| DeepSeek V3.1 | `deepseek.v3.1` | General-purpose DeepSeek. |
| DeepSeek V3.2 | `deepseek.v3.2` | Newer DeepSeek generation, high quality. Supports tool calling (single- and multi-turn). |
| DeepSeek V3.2 Thinking | `deepseek.v3.2-thinking` | Thinking variant. Supports tool calling (single- and multi-turn). |
| Llama 3.1 8B Instruct | `Llama-3.1-8B-Instruct` | Smaller teacher. |
| Llama 3.1 405B Instruct | `Llama-3.1-405B-Instruct` | Large Llama. Supports tool calling (single- and multi-turn). |
| Llama 3.3 70B Instruct | `Llama-3.3-70B-Instruct` | Good general-purpose choice. |
| Qwen3 235B A22B | `Qwen3-235B-A22B-Instruct-2507` | Large MoE. Supports tool calling (single- and multi-turn). |
| Qwen3 480B A35B Coder | `Qwen3-480B-A35B-Coder` | Largest available. Coding-focused. |
| Qwen2.5 VL 72B | `Qwen2.5-VL-72B-Instruct` | Vision-language. |
| ZAI GLM 5 | `zai.glm-5` | High quality. Supports tool calling (single- and multi-turn). |
| ZAI GLM 5 Thinking | `zai.glm-5-thinking` | Thinking variant. Supports tool calling (single- and multi-turn). |
| Moonshot Kimi K2 Thinking | `moonshotai.kimi-k2-thinking` | Thinking variant. Supports tool calling (single- and multi-turn). |
| MiniMax M2 Thinking | `minimax.minimax-m2-thinking` | Thinking variant. Supports tool calling (single- and multi-turn). |

### Teacher recommendations

- **Default:** `openai.gpt-oss-120b`. Strong general-purpose teacher.
- **Highest quality:** `zai.glm-5` or `deepseek.v3.2`. Richer synthetic data.
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
| `tool-calling-closed-book` | Qwen3 and Llama 3-family only |
| `multi-turn-tool-calling-closed-book` | Qwen3 and Llama 3-family only |

**Tool-calling student shortlist:** `Qwen3-0.6B`, `Qwen3-1.7B`, `Qwen3-4B-Instruct-2507`, `Qwen3-8B`, `Llama-3.2-1B-Instruct`, `Llama-3.2-3B-Instruct`, `Llama-3.1-8B-Instruct`.

### Teacher model compatibility

Most task types work with all teacher models. Both tool-calling tasks are the exception — they share a 10-teacher allowlist.

| Task | Teacher constraint |
|------|--------------------|
| `question-answering` | All |
| `classification` | All |
| `tool-calling-closed-book` | Only the 10-teacher allowlist (same as `multi-turn-tool-calling-closed-book`, see row below) |
| `question-answering-open-book` | All |
| `question-answering-closed-book` | All |
| `multi-turn-tool-calling-closed-book` | Only these 10 teachers: `openai.gpt-oss-120b`, `openai.gpt-oss-120b-thinking`, `Llama-3.1-405B-Instruct`, `Qwen3-235B-A22B-Instruct-2507`, `deepseek.v3.2`, `deepseek.v3.2-thinking`, `zai.glm-5`, `zai.glm-5-thinking`, `moonshotai.kimi-k2-thinking`, `minimax.minimax-m2-thinking` |

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
| High-accuracy production | `Qwen3-8B` or `Llama-3.1-8B-Instruct` | `zai.glm-5` or `deepseek.v3.2` |
| Cost-sensitive / edge | `SmolLM2-135M-Instruct` or `gemma-3-270m-it` | `openai.gpt-oss-120b` |
| Tool calling (single-turn) | `Qwen3-1.7B` or `Llama-3.2-1B-Instruct` | `openai.gpt-oss-120b` |
| Tool calling (multi-turn) | `Qwen3-4B-Instruct-2507` | `openai.gpt-oss-120b` |
| Code-heavy tasks | `Qwen3-8B` | `Qwen3-480B-A35B-Coder` |
| Fast experimentation | `Llama-3.2-1B-Instruct` | `openai.gpt-oss-20b` |

---

## Family Notes

- **Qwen3** (0.6B, 1.7B, 4B, 8B) — Strong general-purpose. Supports tool calling. Good multilingual.
- **Llama 3** (1B, 3B, 8B) — Well-established Meta family. Supports tool calling.
- **SmolLM2** (135M, 1.7B) — Efficiency-focused. Does not support tool calling.
- **Gemma 3** (270M, 1B, 4B, plus FunctionGemma 270M) — Google's compact models. Does not support tool calling.
- **IBM Granite** (3.1, 3.3 at 8B) — Enterprise-focused. Does not support tool calling.
- **Liquid LFM** (350M, 1.2B, 2.6B across LFM2 and LFM2.5) — Does not support tool calling.

---

## Hard Rules

- Never recommend SmolLM2, Gemma 3, IBM Granite, or Liquid LFM for a tool-calling task — the platform rejects it.
- For tool calling (both single-turn `tool-calling-closed-book` and `multi-turn-tool-calling-closed-book`), the teacher **must** be one of the 10 tested teachers: `openai.gpt-oss-120b`, `openai.gpt-oss-120b-thinking`, `Llama-3.1-405B-Instruct`, `Qwen3-235B-A22B-Instruct-2507`, `deepseek.v3.2`, `deepseek.v3.2-thinking`, `zai.glm-5`, `zai.glm-5-thinking`, `moonshotai.kimi-k2-thinking`, `minimax.minimax-m2-thinking`. No other teacher works.
- `deepseek.r1` teacher temperature must be 0.5 – 0.7 (validation error otherwise).
