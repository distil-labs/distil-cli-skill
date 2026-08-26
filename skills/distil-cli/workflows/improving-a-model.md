# Workflow: Improving an Existing Model

For when a full pass of `dataset-to-model.md` (or `traces-to-model.md`) shipped a model, or
stalled at Decide, and the analyses or production feedback show gaps. Improvement is a
second iteration of the same pipeline that reuses the first iteration's artifacts instead of
starting over. The job description stays constant throughout. Each stage run lands as a new
iteration id in the same stage directories under the project root.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
1 Diagnose Gaps ─── name the failure modes; they become mutator values
      │
      ├─ gaps not measurable by the current test set ──► 2 Expand the Test Set ─┐
      │                                                                         │
      ▼                                                                         │
3 Teacher Eval (only if the test set or teacher changed; otherwise skip) ◄──────┘
      │
      ▼
4 Iteration-2 Synthgen ─┬─ iter-1 data good ► seed = iter-1 dataset, top-up target
      │                 └─ iter-1 data bad ─► seed = original data, full target
      │                 (either way: new mutators from Step 1)
      ▼
5 Train ─── fast path with the iteration-1 winner
      │
      ▼
6 Decide ─── iter 2 vs iter 1, on the same test set
      ├─ improved and good enough ──► model-deployment.md
      └─ still gapped ──► back to Step 1
```

## Step 1: Diagnose the Gaps

From the training analysis, the Decide review, and what the user knows from production, name
the failure modes concretely: specific scenarios, not "score too low". These names become
mutator values.

## Step 2: Expand the Test Set (when the original scope was too narrow)

If the gaps are not measurable by the current test set, meaning rare failure modes the
collected data never covered, expand it first (`../stages/test-set-expansion.md`). You cannot
fix what you cannot measure.

## Step 3: Teacher Evaluation, Only If Something It Measures Changed

Skip it when the teacher, the job description, and the test set are all unchanged. Run it
(`../stages/teacher-evaluation.md`) when the test set was expanded in Step 2 or the teacher
is being switched, so the ceiling is refreshed before anything is judged against it.

## Step 4: Generate Iteration-2 Data

Run `../stages/synthetic-data-generation.md` with two changes:

- **The seed depends on iteration-1 data quality.** Judge it from that run's synthgen
  analysis and the training results:
  - **good data → reuse it**: seed = the previous iteration's generated dataset (the root
    `train.jsonl` of its synthgen output, where seed and synthetic are already merged). Dedup
    runs against the seed, so the new run avoids repeating iteration 1, and the output root is
    automatically old + new, merged. Set `generation_target` to the TOP-UP amount. It counts
    new examples only, so a fresh 10k doubles the dataset rather than replacing it.

    Mechanically this is a new SeedDataset: download iteration 1's merged `train.jsonl`,
    pair it with a test set and the unchanged job description, and stage the directory as a
    fresh job input. Synthgen then runs from that. So the branch spends a `seed_datasets_post`
    credit on top of the generation one. It also needs `training_datasets_download_get`, which
    starts at zero, because holding the merged file is the whole point and `/sample` returns
    at most 128 rows. Check both balances before proposing it (`../references/platform.md`
    § Credits). At zero on the download route, the fresh-seed branch below is the path,
    whatever the data quality.

    Keep the test set the one you are judging against, not iteration 1's training rows.
  - **bad data → start fresh**: seed = the original seed data with a full
    `generation_target`, after fixing whatever made iteration 1's data bad (usually the
    synthgen config or the mutators). Blending bad data in would just carry the problem
    forward.
- **New `mutators` encoding the Step 1 gaps** (`../references/mutators.md`).

## Step 5: Train

Run `../stages/model-training.md` on the new synthgen output. The student is usually already
chosen: the fast path with iteration 1's winner is the default. Re-sweep only if the gaps
suggest a capacity problem.

## Step 6: Decide

The gates from `dataset-to-model.md` Step 5 apply, plus the iteration comparison: iteration 2
vs iteration 1 on the same metrics. Improved and good enough → deploy. Still gapped → back to
Step 1 with what the new analysis shows.

**Compare only scores measured on the same test set.** If Step 2 expanded the test set, the
expanded file is now the test set (`../stages/test-set-expansion.md` Step 5), and iteration 1's
recorded score was measured on the old one, so it is not a baseline. Either re-score
iteration 1 against the new test set first, or state in the analysis that iteration 2 has no
prior to beat and judge it against the teacher and base student alone. Do not put two numbers
from two different test sets in the same column.
