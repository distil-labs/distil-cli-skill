# Stage: Test-Set Expansion

Grows the test set into areas the collected data never covered: rare failure modes and
scenarios the user cares about that traces cannot show. Mechanically this is a
synthetic-data-generation job with the datasets inverted: the test set acts as the seed,
and new mutation topics steer generation into the uncovered areas. Read
`synthetic-data-generation.md` first. This file describes only what differs.

## Working Directory

```
test-set-expansion/
├── iteration-1/
│   ├── input/           # the exact directory submitted
│   ├── run.md           # submission command and job identifiers
│   └── output/          # generated candidates
└── iteration-2/         # next attempt (e.g. refined topics)
```

## Step 1: Prepare the Input Directory

A standard job-input directory with the datasets INVERTED, so generation mimics test-like
examples:

- `train.jsonl` := the current test set, the seed that anchors format, style, and difficulty
- `test.jsonl` := the current train set. This satisfies validation (no overlap with the
  inverted train split) and plays no quality role here
- `unstructured.jsonl`: include the project's unstructured data when it exists. In-domain
  contexts help generate realistic test cases, and the open/closed-book QA tasks require it
  anyway
- config.yaml: a small `generation_target` (a fraction of the current test-set size), NEW
  `mutation_topics` naming the uncovered areas (`../references/mutators.md`, where a
  single-item list acts as a constant directive), and
  `match_generated_distribution_to_seed` OFF, because
  the whole point is to leave the observed distribution
- job_description.json unchanged, as always

## Step 2: Confirm the Setup with the User

Present and confirm: the uncovered areas and the topics that encode them, the target count,
the remaining `seed_datasets_post` and `training_datasets_from_seed_datasets_post` credits,
and the review plan (the Step 4 gate). The inverted directory is a new SeedDataset, so a run
spends one of each (`../references/platform.md` § Credits). This stage runs at smoke scale by
nature, and there is no separate full run.

Step 4 reviews every generated candidate, which means downloading the output rather than
sampling it, so this stage also needs `training_datasets_download_get` credits. Those start at
zero. Without them the review gate cannot be met, and the stage is unavailable.

## Step 3: Run

Submit a synthetic-data-generation job via the execution backend (§ Submitting jobs) and
record the command and job identifiers in `run.md`.

## Step 4: Review with the User (hard gate)

Pull the generated data. The root `train.jsonl` is the seed and the new rows merged, so the
candidates are that file minus the seed, **matched on the user message's `content` rather than
by comparing whole rows**. The platform normalizes rows on storage (it adds an `images` key to
every message and re-serializes), so a structural or string comparison against your input file
matches nothing and hands you the entire file back as "new":

```python
seed_questions = {
    m["content"] for row in seed_rows for m in row["messages"] if m["role"] == "user"
}
candidates = [
    row for row in generated_rows
    if next(m["content"] for m in row["messages"] if m["role"] == "user")
    not in seed_questions
]
```

These rows will gate every downstream verdict, and they target areas with no observed ground
truth, so review them with the user: are the examples realistic, are the labels correct, do
they actually cover the named areas? Drop or fix what fails. Nothing merges without explicit
approval.

Teacher-generated test rows measure agreement with the teacher, which is a softer signal than
curated examples. It is acceptable because the teacher is the distillation ceiling, and it is
one more reason the review gate is mandatory.

## Step 5: Adopt the Expanded Set as THE Test Set

Append the approved candidates to the project's `test.jsonl` and verify no overlap with
`train.jsonl`. **From here on the expanded file is the test set**, not a merged set to be read
in slices. You expanded precisely because the old set could not measure success, so it is
superseded, not a baseline to compare against. Do not try to mark or track which rows are new:
the platform stores rows as `{"messages": [...]}` and drops any extra field you add, so a
marker silently vanishes anyway.

Everything measured on the OLD test set is now stale, and both baselines need refreshing before
any verdict is read:

- **The teacher ceiling.** Re-run `teacher-evaluation.md` on the new test set.
- **The incumbent student**, if you intend to compare a new model against it. Deploy it
  (`model-deployment.md`) and run the new test set through it. There is no route that
  re-scores a finished SLM against a different test set, so this is the way to get a
  like-for-like number. If that is not worth the deployment, say so plainly in the analysis
  and treat the new test set as a fresh baseline with no prior. An honest "no comparison"
  beats comparing two scores measured on different data.
