# Analyze Model Predictions

Produce a structured analysis report from per-example model predictions. Used after teacher evaluation (to decide whether to proceed to training) and after training (to decide whether to deploy or retune).

The report is the basis for iteration decisions — it surfaces *patterns*, not just aggregate scores. Aggregate metrics tell you *whether* the model is good enough; per-example analysis tells you *what to fix* when it isn't.

**Prerequisites:** Predictions downloaded as JSONL. See `references/tasks/retrieve-predictions.md` for how to download from teacher evaluation, training, or trace processing.

## Always Use `--output json` to Get Metrics

The default text output of `distil model teacher-evaluation`, `distil model training`, etc. **omits some metrics** (notably LLM-as-a-Judge) and only surfaces the strict metrics like ROUGE and tool_call_equivalence. This has caused real iteration loops where users were ready to discard a fine model because the visible scores looked low — when LLM-as-a-Judge was actually passing.

For analysis, always pull the full metric set:

```bash
distil model teacher-evaluation <model-id> --output json | jq '.aggregateMetrics'
distil model training <model-id> --output json | jq '.aggregateMetrics'
```

Use the JSON output as the source of truth for the report's "Aggregate Metrics" section.

---

## Three Report Variants

The report structure is shared. The variants differ in which model(s) are being analyzed, the metrics table layout, and the verdict.

| | Original Model Report | Teacher Evaluation Report | Training Report |
|---|---|---|---|
| **When used** | After trace processing (traces workflow only) | After teacher evaluation | After training |
| **Predictions used** | Original production model (from traces) | Teacher only | Base student + Teacher + Tuned student (+ Original if from traces) |
| **Metrics table** | Single column (Original) | Single column (Teacher) | Multi-column with deltas |
| **Verdict** | INFORMATIONAL (no gate) | PROCEED / ITERATE / RETHINK | DEPLOY / RETUNE / ESCALATE |
| **Decides** | How much relabeling changed the data | Whether to start training | Whether to deploy or retune |

The shared sections (Overview, Input/Output, Test Set Statistics, Configuration Summary, Patterns, Recommended Actions) follow the same structure across all three.

---

## Original Model Analysis Report

Used in the traces workflow only, after trace processing produces the relabeled ground truth. Compares the production model that generated the traces against the committee-relabeled ground truth. The goal is to understand how much relabeling changed the data — there's no gate here, just context for the rest of the workflow.

```markdown
# Original Model Analysis Report

## 1. Overview
- **Model ID:** <model-id>
- **Task type:** <task-type>
- **Original model:** <model that generated the production traces, if known>
- **Goal:** <one-line description of what the model should do>

### 1.1 Input/Output
Input: <one-line description of the input, template variables, message format>
<one example of the input>
Output: <one-line description of the output, JSON schema with field descriptions if relevant>
<one example of the output>

## 2. Test Set Statistics
- **Total examples:** <N>
- **Label distribution:** <for classification: count per class; for QA: answer length distribution>
- **Field lengths:** <min/median/max character length for question, answer, and context columns>
- **Provenance:** <how many examples came from traces vs. user-provided test set>

## 3. Trace Processing Configuration
- **Observation format:** <openai_messages | langfuse | unstructured_with_openai_messages>
- **Relabeling enabled:** <true | false>
- **Convert to single turn:** <true | false>
- **Relabeling committee:** <models used for relabeling>
- **Non-default trace_processing parameters:** <list any that differ from defaults>

## 4. Aggregate Metrics

| Metric | Original Model |
|--------|----------------|
| LLM-as-a-Judge | <score> |
| Exact-Match | <score> |
| ROUGE-L | <score> |
| METEOR | <score> |

(For tool calling, replace with tool_call_equivalence, binary_tool_call, staged_tool_call.)

**Verdict:** INFORMATIONAL — this report doesn't gate the workflow.
- High agreement (≥ 0.8) → relabeling didn't change much; original production data is already high quality.
- Moderate agreement (0.5–0.8) → relabeling improved data quality. Expected for noisy production traces.
- Low agreement (< 0.5) → relabeling significantly changed the data. Inspect samples to confirm relabeling is correct, not introducing noise.

## 5. Agreement Breakdown
- **Original agrees with relabeled ground truth:** <N> (<percentage>%)
- **Original disagrees:** <N> (<percentage>%)
- For classification: confusion matrix (original predicted vs. relabeled)
- For score/numeric fields: distribution of differences

## 6. Analysis of Disagreements

**Patterns identified:**
- <Pattern 1: e.g., "Original model picks 'general' when relabeled goes more specific">
- <Pattern 2: e.g., "Score field differs by ±1 in 40% of cases — drift, not error">
- <Pattern 3: e.g., "Markdown fences in relabeled answers — see fix in upload-and-process-traces.md">

**Implications for training:**
- <e.g., "The student will learn the relabeled distribution. If you trust relabeling, this is the goal.">
- <e.g., "If relabeled outputs include markdown fences, fix before training (see Step in workflow).">

**Details:**
For each disagreement (or a representative sample):

| # | Question (truncated) | Original Output | Relabeled Output | Cause |
|---|---------------------|-----------------|------------------|-------|
| 1 | ... | ... | ... | <specific cause> |
| 2 | ... | ... | ... | <specific cause> |
```

---

## Teacher Evaluation Analysis Report

```markdown
# Teacher Evaluation Analysis Report

## 1. Overview
- **Model ID:** <model-id>
- **Task type:** <task-type>
- **Goal:** <one-line description of what the model should do>

### 1.1 Input/Output
Input: <one-line description of the input to the model, template variables, message format>
<one example of the input>
Output: <one-line description of the output to the model, JSON schema with field descriptions if relevant>
<one example of the output>

## 2. Test Set Statistics
- **Total examples:** <N>
- **Label distribution:** <for classification: count per class; for QA: answer length distribution>
- **Field lengths:** <min/median/max character length for question, answer, and context columns>

## 3. Configuration Summary
- **Task:** <task-type>
- **Student:** <student-model>
- **Teacher:** <teacher-model>
- **Non-default parameters:** <list any config parameters that differ from defaults>

## 4. Aggregate Metrics

| Metric | Score |
|--------|-------|
| LLM-as-a-Judge | <score> |
| Exact-Match | <score> |
| ROUGE-L | <score> |
| METEOR | <score> |

(For tool calling tasks, replace with tool_call_equivalence, binary_tool_call, staged_tool_call.
 For tools with fuzzy/free-text arguments, use LLM-as-a-Judge as the primary metric,
 not tool_call_equivalence.)

**Verdict:** <PROCEED / ITERATE / RETHINK> — apply thresholds from `references/tasks/teacher-evaluation.md` "Default thresholds". That file is the canonical source; do not hardcode thresholds here.

## 5. Agreement Breakdown
- **Correct predictions:** <N> (<percentage>%)
- **Incorrect predictions:** <N> (<percentage>%)
- For classification: confusion matrix showing predicted vs. actual class
- For QA: distribution of LLM-as-a-Judge scores across examples

## 6. Analysis of Disagreements

**Patterns identified:**
- <Pattern 1: e.g., "Teacher consistently misses entity type X">
- <Pattern 2: e.g., "Output format doesn't match for examples with nested JSON">
- <Pattern 3: e.g., "Judge may be too strict — outputs are semantically correct but marked bad due to whitespace">

**Recommended actions:**
- <Action 1: e.g., "Add explicit handling of entity type X to task_description">
- <Action 2: e.g., "Relax llm_as_a_judge_instructions to ignore whitespace differences">
- <Action 3: e.g., "Add 5 more training examples covering nested JSON inputs">

**Details:**
For each incorrect prediction (or a representative sample if there are many):

| # | Question (truncated) | Expected Answer | Teacher Output | Failure Reason |
|---|---------------------|-----------------|----------------|----------------|
| 1 | ... | ... | ... | <specific cause> |
| 2 | ... | ... | ... | <specific cause> |
```

---

## Training Analysis Report

```markdown
# Training Analysis Report

## 1. Overview
- **Model ID:** <model-id>
- **Task type:** <task-type>
- **Student model:** <student-model>
- **Teacher model:** <teacher-model>
- **Training duration:** <hours>
- **Goal:** <one-line description of what the model should do>

### 1.1 Input/Output
Input: <one-line description of the input to the model, template variables, message format>
<one example of the input>
Output: <one-line description of the output, JSON schema with field descriptions if relevant>
<one example of the output>

## 2. Test Set Statistics
- **Total examples:** <N>
- **Label distribution:** <for classification: count per class; for QA: answer length distribution>
- **Field lengths:** <min/median/max character length for question, answer, and context columns>

## 3. Configuration Summary
- **Task:** <task-type>
- **Student:** <student-model>
- **Teacher:** <teacher-model>
- **Non-default parameters:** <list any config parameters that differ from defaults>
- **Synthetic data generated:** <count if available>
- **Training epochs:** <N>

## 4. Aggregate Metrics

| Metric | Base Student | Teacher | Tuned Student | Δ (Tuned − Base) | Δ (Tuned − Teacher) |
|--------|--------------|---------|---------------|------------------|---------------------|
| LLM-as-a-Judge | <score> | <score> | <score> | <diff> | <diff> |
| Exact-Match    | <score> | <score> | <score> | <diff> | <diff> |
| ROUGE-L        | <score> | <score> | <score> | <diff> | <diff> |
| METEOR         | <score> | <score> | <score> | <diff> | <diff> |

(For tool calling, replace with tool_call_equivalence, binary_tool_call, staged_tool_call.
 For tools with fuzzy/free-text arguments, use LLM-as-a-Judge as the primary metric.)

**Training from traces?** Add an **Original Model** column (the production model that generated the traces, scored in the Original Model Analysis Report). The new column order is:
`Original Model | Base Student | Teacher | Tuned Student | Δ (Tuned − Original) | Δ (Tuned − Teacher)`.
The most important comparison becomes Tuned vs. Original — that's whether the trained student beats the production model it's meant to replace.

**Verdict:** <DEPLOY / RETUNE / ESCALATE>
- DEPLOY: tuned student is within ~0.05 of teacher on primary metric, and meets application requirements
- RETUNE: tuned student is significantly below teacher — try different config or larger student
- ESCALATE: tuned student barely beats base student despite good teacher scores — fundamental issue (data, task definition, or distillation isn't transferring knowledge)

## 5. Agreement Breakdown
- **Tuned student agrees with teacher:** <N> (<percentage>%)
- **Tuned correct, teacher wrong:** <N> (<percentage>%)
- **Teacher correct, tuned wrong:** <N> (<percentage>%) ← these are the interesting ones
- **Both wrong:** <N> (<percentage>%)
- **Improvement over base student:** <N> examples now correct that base got wrong; <N> examples now wrong that base got right (regressions)
- For classification: confusion matrix for tuned student predictions

## 6. Analysis of Disagreements

**Patterns identified:**
- <Pattern 1: e.g., "Tuned student fails on longest inputs — possible context length issue">
- <Pattern 2: e.g., "Tuned student confuses two similar classes — needs more examples of each">
- <Pattern 3: e.g., "Tuned student regressed on examples base got right — possible overfitting to synthetic data">

**Recommended actions if RETUNE:**
- <Action 1: e.g., "Try Llama-3.2-3B-Instruct as student (current 1B may be too small)">
- <Action 2: e.g., "Increase generation_target to 15000 for more training diversity">
- <Action 3: e.g., "Add mutation_topics targeting the weak areas">

**Details:**
Focus on cases where teacher was correct but tuned student was wrong — these reveal what the student failed to learn.

| # | Question (truncated) | Expected | Base Output | Teacher Output | Tuned Output | Failure Reason |
|---|---------------------|----------|-------------|----------------|--------------|----------------|
| 1 | ... | ... | ... | ... | ... | <specific cause> |
| 2 | ... | ... | ... | ... | ... | <specific cause> |
```

---

## Tips for Writing Useful Analysis

1. **Patterns first, examples second.** A long table of failures isn't analysis — it's data. The "Patterns identified" section is where you do the actual work: group failures, name what they have in common, propose what's causing them.

2. **Recommended actions must be concrete.** "Improve the data" is not actionable. "Add 5 examples covering inputs longer than 2000 chars, since 80% of failures are on long inputs" is.

3. **Look at the right disagreements.** In teacher eval, look at where the teacher was wrong. In training, focus on where teacher was right but the tuned student was wrong — that's what the student failed to learn during distillation.

4. **Watch for regressions in the training report.** If the tuned student got worse on examples the base student got right, that's a signal of overfitting to synthetic data — usually fixable by reducing `generation_target` or increasing diversity via mutations.

5. **Distinguish judge errors from model errors.** Sometimes the model's output is correct but the LLM-as-a-Judge marks it bad due to format quirks. If you see this pattern, the fix is in `llm_as_a_judge_instructions`, not in the data or model.
