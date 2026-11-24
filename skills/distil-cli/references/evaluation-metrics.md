# Evaluation Metrics

Metrics per task, the primary metric, and the reference points that scores are read against.
The aggregated scores are the `*_performance` object that a `metrics` command returns; the
per-example predictions are a separate download (`execution/cli.md` § Fetch metrics).

## Metrics by task

| Suite | Tasks | Metric keys |
|---|---|---|
| QA | `question-answering` | `rouge`, `binary`, `llm-as-a-judge`, `llm-as-a-judge-reference-free` |
| Conversation | `chat-completion` | `rouge`, `tool_call_equivalence`, `binary_tool_call`, `staged_tool_call`, `llm-as-a-judge`, `llm-as-a-judge-reference-free` |

All 0-1:

- `llm-as-a-judge`: LLM verdict (good=1, bad=0, or a graded score from 0 to 1) on whether the prediction matches the
  reference. The judge sees the full prefix (system message, question and context). Unparseable verdicts
  count as bad (0).
- `llm-as-a-judge-reference-free`: LLM verdict (good=1, bad=0, or a graded score from 0 to 1) on whether the prediction solves
  the problem, from the full prefix and the prediction alone. The judge never sees the
  reference, so a wrong reference does not lower the score. It is the default primary metric
  (§ Primary metric per task). Read it next to `llm-as-a-judge`: a
  prediction marked bad against the reference but good without it points at the reference row,
  not the model. Unparseable verdicts count as bad (0).
- Both judge metrics give a pass/fail verdict by default, and the metric is the fraction marked
  good. When `llm_as_a_judge_instructions` ask for a number from 0 to 1, the metric is the mean
  graded score (`job-description.md` § Judge score modes).
- `binary`: exact string equality, ignoring differences in whitespace. The faithful measure for a label or a
  set (§ Primary metric per task); for free text, an extra check only.
- `rouge`: text overlap, a secondary signal.
- `tool_call_equivalence`: exact match, except an argument set to its schema default equals
  omitting it. Defaults come from the row's own `tools` when it has them, else the job
  description's.
- `binary_tool_call`: strict name and arguments equality.
- `staged_tool_call`: 0.25 per stage, one call each side → name matches → argument keys
  match → arguments match. 0.5 means right tool, wrong arguments.

Conversation suite specifics: a predicted turn can carry text, tool calls, or both. Parallel
calls are compared positionally (right calls, wrong order = 0), and extra or missing parallel
calls are penalised. A turn with no calls on either side scores 1 on the tool metrics, so they
stay meaningful for tool-free rows; the judge metrics carry the text signal.

## Primary metric per task

The default primary metric, for every task, is `llm-as-a-judge-reference-free` with a strong
judge model (`evaluation.llm_as_a_judge_model_name`) and a rubric (`llm_as_a_judge_instructions`
in the job description, `job-description.md` § What each field feeds). The judge model and
the rubric are agreed with the user at the start of the project and never changed afterwards: a
different judge or rubric makes every earlier score incomparable. It needs no reference answer,
so it works when the teacher wrote the test set's answers.

Two alternatives, each chosen with the user:

- **`binary`** exact match, where exact string equality is the faithful measure: a label
  (`data-preparation/question-answering.md` § Classifying with question-answering) or a set.
  There is then no judge to agree. Never for free text or code, where correct answers differ in
  form.
- **`llm-as-a-judge`** (reference-based), when the user's gold labels are the reference and the
  user prefers it.

Every score in § Verdicts (teacher, base, student, production model, `closed`) is the agreed key.

## Verdicts

Judge results on the primary metric (§ Primary metric per task) against reference points,
never as absolute numbers: 0.72 is a good result against a teacher at 0.75 and a poor one
against a teacher at 1.00. This section defines the reference points; what to do next is
decided by the workflow.

**Noise.** A difference between two scores means something only when it is larger than the noise.
The noise has three sources, and a decision should use the ones that apply to it:

- **Evaluation noise.** The teacher and the judge run at non-zero temperature, so the same
  predictions score differently from run to run. Measure it as the spread between two teacher
  evaluations on the same test set.
- **Training noise.** Two trainings on the same Dataset with different `random_seed` values give
  different models. Their scores usually differ by more than the evaluation noise, and the
  difference is often a handful of the same borderline rows flipping between models. Measure it by
  training one Dataset twice. It is the noise that matters when you compare two trained models.
- **Test-set noise.** A small group measures little: one row in a group of 50 moves the group's
  score by 0.02. A group's spread between two runs says how stable that group is. It is not the
  noise for the whole test set.

Use judgement to put these together. Some guidelines:

- Read a decision on the scores it is about. A change aimed at one kind of input is judged on
  that group, and on the real-row groups that show what the change broke elsewhere.
- A group whose scores swing between runs is unstable. Report it as unstable and give its
  differences a wider margin. Do not widen the noise of every other comparison because of it.
- Measure the training noise before a close decision rests on it. That means retraining once
  with another seed when a change helps one group and hurts another by a few rows, or when
  several models in a row land on the same side of the best model. A best model that was
  trained once may be a lucky draw.
- Write down which noise estimate a decision used and why, so the decision can be checked later.

**Target.** By default, the mean score of one to three strong models other than the teacher
that wrote the test references: that teacher's own score on its references is inflated. The
target is reached when the best score is above it, or below it by no more than the noise.

**`closed`.** The share of the base → teacher gap the student closed:

```
closed = (score - base) / (target - base)     # each term is the primary metric
```

It is reported for every trained model in the ledger of the run log. When `target - base` is
small the ratio is unstable: say so.

**One base score per project.** Every training run scores the base student again, so the base
moves with the noise. Use one base score for the project: the first run's, or the mean of the
runs so far.

**The production model is a floor.** The model the user runs in production is scored on the
test set by a teacher evaluation with it as `base.teacher_model_name`: `teacher_performance` of
that run is its answers scored with the same metric suite as the student, against the same
references. A student below it is not deployed, whatever `closed` says. Compare the same
primary-metric key. When the production model is not in the catalog (`model-catalog.md`
§ Teacher models), there is no baseline, and the user judges the student on the absolute
score.

With an empty test split there are no reference points and no verdict
(`data-preparation/overview.md` § Empty splits): report the model as unmeasured.
