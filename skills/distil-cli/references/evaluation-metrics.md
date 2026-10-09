# Evaluation Metrics

Metrics per task and the verdict thresholds. The aggregated scores are the `*_performance`
object that a `metrics` command returns; the per-example predictions are a separate download
(`execution/cli.md` § Fetch metrics).

## Metrics by task

| Suite | Tasks | Metric keys |
|---|---|---|
| QA | `question-answering` | `rouge`, `binary`, `llm-as-a-judge`, `llm-as-a-judge-reference-free` |
| Classification | `classification` | `accuracy`, plus one key per class label (see below) |
| Conversation | `chat-completion`, `chat-completion-agentic` | `rouge`, `tool_call_equivalence`, `binary_tool_call`, `staged_tool_call`, `llm-as-a-judge`, `llm-as-a-judge-reference-free` |

All 0-1:

- `llm-as-a-judge`: LLM verdict (good=1, bad=0) on whether the prediction matches the
  reference. The judge sees the full prefix (question and context). Unparseable verdicts
  become NaN and drop from the mean.
- `llm-as-a-judge-reference-free`: LLM verdict (good=1, bad=0) on whether the prediction solves
  the problem, from the full prefix and the prediction alone. The judge never sees the
  reference, so a wrong reference does not lower the score. It is the primary metric for every
  task except classification (§ Primary metric per task). Read it next to `llm-as-a-judge`: a
  prediction marked bad against the reference but good without it points at the reference row,
  not the model. Unparseable verdicts become NaN and drop from the mean.
- `binary`: exact string equality on the raw strings. Use it as an extra check only. Do not
  rely on it.
- `rouge`: text overlap, a secondary signal.
- `accuracy`: exact label match.
- `tool_call_equivalence`: exact match, except an argument set to its schema default equals
  omitting it.
- `binary_tool_call`: strict name and arguments equality.
- `staged_tool_call`: 0.25 per stage, one call each side → name matches → argument keys
  match → arguments match. 0.5 means right tool, wrong arguments.

Conversation suite specifics: a predicted turn can carry text, tool calls, or both. Parallel
calls are compared positionally (right calls, wrong order = 0), and extra or missing parallel
calls are penalised. A turn with no calls on either side scores 1 on the tool metrics, so they
stay meaningful for tool-free rows; the judge metrics carry the text signal.

The classification performance object is not flat. Alongside `accuracy` it carries one key
per class label, each holding that class's precision, recall, f1-score and support:

```json
{"accuracy": 1.0,
 "lane_cold":    {"precision": 1.0, "recall": 1.0, "f1-score": 1.0, "support": 9.0},
 "lane_general": {"precision": 1.0, "recall": 1.0, "f1-score": 1.0, "support": 7.0}}
```

So iterate the object by key rather than assuming every value is a number: `accuracy` is a
float and every other entry is a dict.

## Primary metric per task

The default primary metric:

- `question-answering`, `chat-completion`, `chat-completion-agentic` →
  `llm-as-a-judge-reference-free`.
- `classification` → `accuracy`.

The user or the agent can name a different one per task, agreed at the start of the project:
for example `llm-as-a-judge` (scored against the reference), or `binary` exact match for set or
structured outputs where exact match is the faithful measure. Every score in § Verdicts
(teacher, base, tuned, production model, `closed`) is the agreed key.

## Verdicts

Judge results on the primary metric (§ Primary metric per task) against reference points,
never as absolute numbers: 0.72
is a good result against a teacher at 0.75 and a poor one against a teacher at 1.00.

**Noise.** The judge runs at non-zero temperature, so the same model on the same test set
scores differently from run to run. A difference between two scores means something only when
it is larger than that run-to-run variation. Measure the variation for the test set in use, for
example by running the same evaluation again, and call it the noise band.

**Teacher evaluation.**

| Verdict | When |
|---|---|
| PROCEED | The user has seen the teacher's score and its failure patterns and accepts that quality for production |
| ITERATE | The score or the failure patterns are not acceptable, and a different teacher, or a fix to wrong test rows or to the judge, could change that |
| RETHINK | No teacher does the task well: the task type, the task definition or the metric is likely wrong |

**Training.** The fraction of the base → teacher gap the student closed decides the branch:

```
closed = (tuned - base) / (teacher - base)     # each term is the primary metric
```

| `closed` | Verdict | Action |
|---|---|---|
| **≥ 0.8** | Deploy candidate | Continue to deployment; for a sweep, take the smallest student that clears the bar |
| **0.4 - 0.8** | Retune | Train again on the same Dataset with a different student or tuning settings; nothing is regenerated (`../workflows/model-iterations.md`, re-entry at Step 8) |
| **< 0.4** | Iterate | The teacher's knowledge is not reaching the student, and retuning will not fix it. `../workflows/model-iterations.md` |

A `closed` of 1 or above means the student matches or beats the teacher's score on this test
set, which is the default target of `../workflows/model-iterations.md`.

These are gates for judgement. When `teacher - base` is small the ratio is unstable, and when
the gap is inside the noise band no branch follows from the numbers: say so.

**The production model is a floor.** The model the user runs in production is scored on the
test set by a teacher evaluation with it as `base.teacher_model_name`
(`../stages/teacher-evaluation.md`): `teacher_performance` of that run is its answers scored
with the same metric suite as the student, against the same references. A student below it is
not deployed whatever `closed` says. Compare the same primary-metric key. When the production
model is not in the catalog, there is no baseline, and the user judges the student on the
absolute score.

With an empty test split there are no reference points and no verdict
(`data-preparation/overview.md` § Empty splits): report the model as unmeasured.
