# Analyze Uploads

Deep dive into the data splits (train / test / unstructured) before spending credits on teacher evaluation. How each workflow uses this file:

- **Dataset workflow (standard, pre-upload):** run the Quantitative Section against the user's local files during the "Data consistency analysis" step of `workflows/dataset-to-model.md`, before `upload-data`. For this workflow the findings DO gate the upload: flagged issues should be fixed before uploading. The Qualitative Section is an opt-in extra.
- **Traces workflow (opt-in, post-approval):** offered right after test-set approval in `workflows/traces-to-model.md`, once `upload-traces` or `reprocess-traces` has completed and the user has approved the test set. Informational only; it does not gate the workflow. Ask the user verbatim:

```
Want a deeper look at the full uploads object (train / test / unstructured), beyond just the test set? This pulls the upload down and analyzes cross-split consistency and categorical coverage. Reply 'yes' to run it, otherwise we continue.
```

If the user declines, skip this step entirely and continue to teacher evaluation.

## Inputs

- **Working directory:** the current `iteration-N/` directory when one exists (see `workflows/improving-a-model.md`'s Iteration Discipline section). On a first dataset-workflow pass no iteration directory exists yet; work from the project root and do not create `iteration-1/` early (that convention belongs to the iteration loop).
- **Data source:**
  - Dataset workflow: the user's local train / test / unstructured files; no download needed. If no unstructured file was provided, skip the unstructured-coverage check.
  - Traces workflow: `distil model download-data <model-id>` downloads the processed train / test / unstructured files locally. See `references/tasks/upload-dataset.md`.

## Token-Burn Guard

The qualitative section reads many examples and produces a categorical breakdown. On large uploads this is expensive. **Sample at most 200 examples per split** for the qualitative pass. Say so in your report so the user knows the conclusions are drawn from a sample, not the whole set.

## Quantitative Section (mechanical)

- **Label / class distribution train vs. test.** Flag any class below 5% of either split, or any class present in train but missing from test (or vice versa). For QA, report answer-length quartiles per split.
- **Field-length percentiles (p10 / p50 / p90 / max) per split** for `question`, `context`, `answer`. The platform **silently truncates** examples above its length limits (structured and unstructured data have different caps, see `references/configuration.md`) with no warning, so surface the longest examples and confirm with the user before uploading rather than letting content be cut silently.
- **Schema conformance:**
  - Tool calling → every `answer` parses as valid JSON; record the count that failed, if any.
  - Classification → every class named in `classes_description` appears in train; flag missing classes.
- **Train/test leakage:** count test rows that duplicate a train row exactly or near-duplicate one (near-duplicate: differs only in whitespace, casing, or trivial punctuation). Any overlap inflates teacher-evaluation and training scores and makes downstream verdicts unreliable; report the offending rows so the user can remove them from one split.
- **Unstructured split coverage:** does it span the same input-structure space as train/test? Compare input-length and (where detectable) structural markers.

## Qualitative Section (Claude-side judgment)

- **Generate 3–8 categorical axes** that actually partition this dataset. Pick axes that matter for the domain, not generic ones. Examples: programming language, domain (e.g., finance / legal / support), task sub-type (summarize / extract / rewrite), input-length tier, difficulty, style (terse / verbose). Derive axes from the data, not from a fixed list.
- **Report breakdown per axis** for train and test (counts + percentages). Flag axes where train and test diverge meaningfully.
- **Cross-check `job_description.json`:**
  - Does the data support what the JD claims to cover?
  - Does the JD miss scenarios that are clearly present in the data?
  - See `references/job-description-guide.md` for what a good JD looks like.

## Output

**Dataset workflow, pre-upload run:** report the quantitative findings inline in 3-5 lines; no report file is required. Write the full report below only if the user opts into the qualitative deep dive, saving it to the current `iteration-N/` if one exists, otherwise to the project root as `upload-consistency.md`.

**Traces workflow (and any run inside the iteration loop):** save the report to `<iteration-N>/upload-consistency.md`.

Report template:

```markdown
# Upload Consistency Report

## 1. Overview
- **Model ID:** <model-id>
- **Iteration:** <N | pre-upload>
- **Splits inspected:** train (<N>), test (<N>), unstructured (<N | not provided>)
- **Sampling:** <whole split | first 200 per split | stratified sample of 200 per split>

## 2. Quantitative Findings
- **Label / class distribution:** <summary + any flags>
- **Field-length percentiles:** <summary + any tails flagged>
- **Schema conformance:** <pass | N failures of type X>
- **Train/test leakage:** <none | N overlapping rows listed>
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

If INVESTIGATE, concrete next moves (pick the ones that match the workflow):
- Dataset workflow: edit the local train/test files or `job_description.json` to close <specific gap>, re-run the validation checklist, then upload with `distil model upload-data`.
- Either workflow: add `synthgen.mutation_topics` targeting <missing scenario> (see `references/mutations-guide.md`).
- Traces workflow: edit `job_description.json` to <specific gap> and re-run `distil model upload-traces` (see `workflows/traces-to-model.md`).
- Traces workflow: adjust `trace_processing` params (e.g., `num_traces_as_training_base`) and run `distil model reprocess-traces` (see `references/tasks/upload-and-process-traces.md`).
```

## Log

Append a log entry at `model-building-log-<name>.md` with the verdict and the headline finding. See `references/tasks/maintain-run-log.md`.
