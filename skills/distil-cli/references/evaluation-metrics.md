# Evaluation Metrics

Metrics per task plus the canonical verdict thresholds. Aggregated scores come back inline as
the `*_performance` object in an entity's metrics response; per-example predictions are a
separate download alongside it (the execution backend § Fetch metrics).

## Metrics by task

| Suite | Tasks | Metric keys |
|---|---|---|
| QA | `question-answering`, `-open-book`, `-closed-book` (and deprecated QA variants) | `rouge`, `binary`, `llm-as-a-judge`, `llm-as-a-judge-reference-free` |
| Classification | `classification` | `accuracy`, plus one key per class label (see below) |
| Tool calling | `tool-calling-closed-book`, `multi-turn-tool-calling-closed-book` | `rouge`, `tool_call_equivalence`, `binary_tool_call`, `staged_tool_call`, `llm-as-a-judge`, `llm-as-a-judge-reference-free` |

All 0-1:

- `llm-as-a-judge` — LLM verdict (good=1, bad=0) vs the reference; `-reference-free` omits
  the reference. The judge sees the full prefix (question and context). Unparseable verdicts
  become NaN and drop from the mean.
- `binary` — exact string equality on the raw strings. Only useful as an extra check but
  should not be relied on.
- `rouge` — text overlap; secondary signal.
- `accuracy` — exact label match.

The classification performance object is **not flat**. Alongside `accuracy` it carries one key
per class label, each holding a per-class breakdown, which gives precision and recall for free:

```json
{"accuracy": 1.0,
 "lane_cold":    {"precision": 1.0, "recall": 1.0, "f1-score": 1.0, "support": 9.0},
 "lane_general": {"precision": 1.0, "recall": 1.0, "f1-score": 1.0, "support": 7.0}}
```

So iterate the object by key rather than assuming every value is a number — `accuracy` is a
float and every other entry is a dict.
- `tool_call_equivalence` — exact match, except an argument set to its schema default equals
  omitting it.
- `binary_tool_call` — strict name + arguments equality.
- `staged_tool_call` — 0.25 per stage: one call each side → name matches → argument keys
  match → arguments match. 0.5 means right tool, wrong arguments.

## Primary metric per task

- Classification → `accuracy`.
- QA (all variants) → `llm-as-a-judge`.
- Tool calling (all variants) → `llm-as-a-judge`.

## Verdicts (relative gates)

Judge results against reference points on the primary metric, not against absolute numbers.
The thresholds below are on the *relative* gap closed, never on a raw score: 0.72 is a good
result against a teacher at 0.75 and a poor one against a teacher at 1.00.

**Teacher evaluation** — the teacher score is the ceiling everything downstream distills
from. Review the score and failure patterns with the user and confirm this quality would be
acceptable in production: that is PROCEED. Otherwise iterate on the teacher choice (ITERATE)
or re-examine the task setup (RETHINK).

**Scores are samples, not constants.** Evaluation runs the judge at non-zero temperature, so
the same model on the same test set scores differently run to run — a measured ±0.03 on a
50-row set. Any difference smaller than that band is noise.

The per-example predictions file behind each `/metrics` response is where failure patterns
live. Group the rows the model got wrong by whatever the task's failure modes are — a class, a
format, a rule — rather than reading the aggregate and guessing.

**Training** — compare the tuned student against its reference points. Express the result as
the **fraction of the base→teacher gap the student closed**, which makes one number decide the
branch:

```
closed = (tuned - base) / (teacher - base)
```

| `closed` | Verdict | Action |
|---|---|---|
| **≥ 0.8** | Deploy candidate | Continue to deployment; for a sweep take the smallest student clearing the bar |
| **0.4 – 0.8** | Retune | Clearly transferring but short of the ceiling. Re-enter `../stages/model-training.md` from the same data with a different student or tuning parameters — nothing regenerates, so this is the cheap loop |
| **< 0.4** | Rerun the process | Barely above base despite a good teacher score: the knowledge is not transferring and retuning will not fix it. Go to `../workflows/improving-a-model.md` |

These are gates for judgement, not thresholds to apply mechanically: when `teacher - base` is
small the ratio is unstable, and when the gap is within the noise band above, no branch is
decidable from the numbers alone. Say so rather than picking one.

**A trace-derived build has a fourth reference point, and it is a floor rather than a gate.**
The original production model is the one being replaced, so a student that scores below it is
not shippable whatever `closed` says — a strong `closed` against a weak base still loses to the
incumbent. Treat it as a veto over the table above: below it, the verdict is Retune at best,
never Deploy. Read it from `base_model_performance` on the SeedDataset, which carries it only
when the SeedDataset came from prepared traces.
