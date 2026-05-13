# Analyze Uploads

Optional, user-opt-in deep dive into the full uploads object (train / test / unstructured) after trace processing. Informational — does not gate the workflow.

The traces workflow offers this right after test-set approval. The dataset workflow may also offer the quantitative section on demand if the user wants an extra consistency check on their own data, but it is not a standard step there.

## When to Offer

After `upload-traces` or `reprocess-traces` completes **and** the user has approved the test set. Ask the user verbatim:

```
Want a deeper look at the full uploads object (train / test / unstructured), beyond just the test set? This pulls the upload down and analyzes cross-split consistency and categorical coverage. Reply 'yes' to run it, otherwise we continue.
```

If the user declines, skip this step entirely and continue to teacher evaluation.

## Inputs

- **Working directory:** the current `iteration-N/` directory (see `workflows/improving-a-model.md`'s Iteration Discipline section).
- **Data source:** `distil model download-data <model-id>` — downloads the processed train / test / unstructured files locally. See `references/tasks/upload-dataset.md`.

## Token-Burn Guard

The qualitative section reads many examples and produces a categorical breakdown. On large uploads this is expensive. **Sample at most 200 examples per split** for the qualitative pass. Say so in your report so the user knows the conclusions are drawn from a sample, not the whole set.

## Quantitative Section (mechanical)

- **Label / class distribution train vs. test.** Flag any class below 5% of either split, or any class present in train but missing from test (or vice versa). For QA, report answer-length quartiles per split.
- **Field-length percentiles (p10 / p50 / p90 / max) per split** for `question`, `context`, `answer`. Flag tails that approach or exceed `synthgen.validation_max_total_length` (default 10,000 chars — see `references/configuration.md`).
- **Schema conformance:**
  - Tool calling → every `answer` parses as valid JSON; record the count that failed, if any.
  - Classification → every class named in `classes_description` appears in train; flag missing classes.
- **Unstructured split coverage:** does it span the same input-structure space as train/test? Compare input-length and (where detectable) structural markers.

## Qualitative Section (Claude-side judgment)

- **Generate 3–8 categorical axes** that actually partition this dataset. Pick axes that matter for the domain, not generic ones. Examples: programming language, domain (e.g., finance / legal / support), task sub-type (summarize / extract / rewrite), input-length tier, difficulty, style (terse / verbose). Derive axes from the data, not from a fixed list.
- **Report breakdown per axis** for train and test (counts + percentages). Flag axes where train and test diverge meaningfully.
- **Cross-check `job_description.json`:**
  - Does the data support what the JD claims to cover?
  - Does the JD miss scenarios that are clearly present in the data?
  - See `references/job-description-guide.md` for what a good JD looks like.

## Output

Save the report to `<iteration-N>/upload-consistency.md`. Use this template:

```markdown
# Upload Consistency Report

## 1. Overview
- **Model ID:** <model-id>
- **Iteration:** <N>
- **Splits inspected:** train (<N>), test (<N>), unstructured (<N>)
- **Sampling:** <whole split | first 200 per split | stratified sample of 200 per split>

## 2. Quantitative Findings
- **Label / class distribution:** <summary + any flags>
- **Field-length percentiles:** <summary + any tails flagged>
- **Schema conformance:** <pass | N failures of type X>
- **Unstructured coverage:** <aligned | diverges on X>

## 3. Qualitative Breakdown
Axes derived from the data:
- **<Axis 1 name>:** train <counts> vs. test <counts>. <finding>
- **<Axis 2 name>:** ...
- ...

## 4. Job-Description Cross-Check
- <e.g., "JD mentions handling 'empty tool arguments' but no such examples in train or test">
- <e.g., "Train contains heavy Spanish-language representation; JD only mentions English">

## 5. Verdict and Next Moves

**Verdict:** <PROCEED | INVESTIGATE>

If INVESTIGATE, concrete next moves:
- Edit `job_description.json` to <specific gap> and re-run `distil model upload-traces` (see `workflows/traces-to-model.md`).
- Add `synthgen.mutation_topics` targeting <missing scenario> — see `references/mutations-guide.md`.
- Adjust `trace_processing` params (e.g., `num_traces_as_training_base`) and run `distil model reprocess-traces` — see `references/tasks/upload-and-process-traces.md`.
```

## Log

Append a log entry at `model-building-log-<name>.md` with the verdict and the headline finding. See `references/tasks/maintain-run-log.md`.
