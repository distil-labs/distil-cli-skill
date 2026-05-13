# Teacher Evaluation

Teacher evaluation validates whether a large language model can solve your task before you invest time in training a small model. It serves as both a feasibility check and a performance benchmark.

## Purpose

- **Feasibility check:** If the teacher model can solve the task, the student model will be able to learn it effectively through knowledge distillation. If the teacher cannot solve it, you need to refine your inputs first.
- **Performance benchmark:** The teacher's accuracy on your test set provides the first approximation of expected SLM performance. Your trained student model should achieve results reasonably close to the teacher.

## Run Teacher Evaluation

After uploading data (via `upload-data` or `upload-traces`), start the evaluation:

```bash
distil model run-teacher-evaluation <model-id>
```

## Check Status and Results

Check the status and view results:

```bash
distil model teacher-evaluation <model-id>
```

The evaluation returns multiple scores on your test set. The specific metrics depend on your task type:

- **Text generation tasks** (question answering, classification, open/closed book QA): LLM-as-a-Judge, Exact-Match, ROUGE-L, METEOR
- **Tool calling tasks** (tool calling, multi-turn tool calling): tool_call_equivalence, binary_tool_call, staged_tool_call

For detailed explanations of each metric and how to interpret scorecards, see `evaluation-metrics.md`.

## Score Interpretation

### Default thresholds (canonical — referenced from workflows and analysis templates)

Use these as starting rules. Override them only when the judgment cases below apply.

**Text generation tasks** (QA, classification, open/closed book QA) — primary metric is **LLM-as-a-Judge**:

| Score | Verdict | Action |
|-------|---------|--------|
| ≥ 0.80 | PROCEED | Proceed to training. |
| 0.60 – 0.80 | ITERATE | Review job description and data quality. Iterate before training. |
| < 0.60 | RETHINK | Task may be under-specified. Do NOT proceed to training. |

**Tool calling tasks** — which metric to use depends on the tool schemas:

- **Tools with constrained arguments** (enums, IDs, booleans — e.g., `get_weather(city: enum, unit: enum)`) → use **tool_call_equivalence** as the primary metric.
- **Tools with fuzzy/free-text arguments** (e.g., `reply_to_user(message: str)`, `search(query: str)`) → **do not rely on tool_call_equivalence**. Use **LLM-as-a-Judge** as the primary metric instead, since exact string matching will undercount correct outputs.

| Score (primary metric) | Verdict | Action |
|-------|---------|--------|
| ≥ 0.70 | PROCEED | Proceed to training. |
| < 0.70 | ITERATE | Iterate first. |

Workflows and analysis report templates reference these thresholds — update here and everything downstream follows.

### When to override these thresholds (judgment required)

- If the task is inherently ambiguous (e.g., open-ended summarization), 0.7 LLM-as-a-Judge may be acceptable.
- If the judge instructions are weak, the score may understate actual quality — improve `llm_as_a_judge_instructions` in `job_description.json` first before concluding the task is infeasible.
- If only one specific failure pattern appears (e.g., one class always wrong in classification), a targeted fix may be more effective than re-evaluating the entire task.

## Interpreting Good Results

High scores on the primary metric mean:

- Your task is well-defined and the data is high quality.
- The teacher model understands what you are asking it to do.
- You can proceed to training with confidence.

The teacher's scores become the quality benchmark for your trained SLM.

## Interpreting Bad Results

Low teacher accuracy means the teacher model struggled with your task. This is a signal to iterate before proceeding to training, because if the teacher cannot solve the task, the student will not learn it effectively.

Common causes and actions:

| Issue | Action |
|-------|--------|
| Task description is too vague or ambiguous | Make the `task_description` in `job_description.json` more specific. Clarify what the model should do and what format the output should be in. |
| Training examples are inconsistent or low quality | Review your train and test data. Check for contradictory labels, ambiguous examples, or formatting errors. |
| Wrong task type selected | Verify that the task type in `config.yaml` matches what you are actually asking the model to do. |
| Task is inherently difficult | Try a more capable teacher model. Check `configuration.md` for available teacher models. |
| LLM-as-a-Judge score is lower than expected | The judge may not be evaluating correctly. Provide a well-structured `llm_as_a_judge_instructions` field in your `job_description.json` that clearly defines what constitutes a correct answer, what edge cases to accept, and what criteria to use for scoring. A well-defined judge instruction is critical for accurate evaluation. |
| tool_call_equivalence is low but outputs look correct | Your tools likely have free-text arguments. Switch to LLM-as-a-Judge as the primary metric. |

## When to Iterate Before Training

Iterate on your data and configuration before starting training if:

- Primary metric scores are below the thresholds above.
- All metrics are low, which indicates the task may be under-specified.
- You see a pattern of specific failure types in the results (e.g., the model always gets one category wrong in classification).

The iteration loop is: revise job description or data, re-upload with `upload-data` or `upload-traces`, and run teacher evaluation again. Repeat until the teacher performs well.
