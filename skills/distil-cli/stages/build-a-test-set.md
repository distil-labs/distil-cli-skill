# Stage: Build a Test Set

Fills the Dataset's `test.jsonl` with rows that represent production, and locks it. Every
verdict in the project (teacher evaluation, the production-model baseline, every trained
student) is read on this test set, so it is built once, reviewed with the user as a hard gate,
and then carried unchanged through every Dataset after it: the train-side expands copy
`test.jsonl` through, so any Dataset in the chain from here on has the same test set, and
scores along the chain are comparable.

Two mechanics fill it, in this order:

1. **Relabelled traces** (`relabel-traces.md` on the test split): real production inputs with
   reference answers written by the teacher. The stronger signal; use it for as many rows as the
   trace budget allows.
2. **Synthetic rows** (`synthetic-data-generation.md` on the test split): rows the teacher
   writes from the relabelled ones, the job description and the traces as context. They measure
   agreement with the teacher rather than with production, so they top the set up when the
   traces do not reach the target.

## Working Directory

The mechanics stages own their directories (`relabel-traces-test/`,
`synthetic-data-generation-test/`). This stage keeps `test-set.md` in the project root: the
target, the plan, the Dataset id that holds the approved test set, and the review notes.

## Step 1: Take Stock

Download the Dataset (`../references/execution/cli.md` § Reading a Dataset) and record:

- the rows already in `test.jsonl` (an uploaded test set stays in and is added to);
- the distinct traces in `traces.jsonl`;
- how many traces the train set will need after this stage (`build-a-train-set.md` Step 2:
  `num_train_relabelled` plus the generation context).

If the user uploaded a test set they consider complete, this stage is only Steps 4 and 5 on
it.

## Step 2: Plan and Confirm with the User

Agree the target: a few hundred rows, between 100 and 1000. More rows narrow the noise band
(`../references/evaluation-metrics.md` § Verdicts) and cover more of production; past a
thousand the gain is small and the relabelling cost is not. Then split the target:

- `trace_processing.num_test_relabelled` = the target, capped at what the trace pool can spare
  after the train set's needs;
- `synthgen.test_generation_target` = the rest, when the relabelled rows and the uploaded ones
  do not reach the target. Zero otherwise. A generation run of T rows also takes `T +
  min(T, 1000)` traces as context, or all that are left (`../references/platform.md` § The
  expand operations); count them against the pool before the train set's share, or set
  `synthgen.use_traces_as_context: false` for this run when the traces are scarce.

For classification, the relabelled rows must cover every class, or the expand fails
validation; plan synthetic rows for the rare classes. Present the target, the split of it, the
relabelling teacher (the default large teacher, `../references/model-catalog.md` § Defaults,
unless the user asks for a stronger one: it writes the reference answers), the judge
(`evaluation.llm_as_a_judge_model_name` and `llm_as_a_judge_instructions`, fixed from here on)
and the credits: one `datasets_from_datasets_relabel_traces_post`, one
`datasets_from_datasets_generate_synthetic_data_post` when synthetic rows are planned, and the
smoke credits each mechanics stage spends (`../references/platform.md` § Credits).

## Step 3: Fill the Test Set

Run `relabel-traces.md` on the test split with `num_test_relabelled` from the plan. Then, when
the plan has synthetic rows, run `synthetic-data-generation.md` on the test split of the
Dataset relabelling produced, with `test_generation_target` from the plan. Each run is a new
Dataset; the second runs on the first's output, and `test-set.md` records both ids.

Skip the first run when there are no traces, and the second when the relabelled rows reach the
target.

## Step 4: Review with the User

This review is a hard gate: nothing goes to teacher evaluation without the user's approval.
Download the last Dataset and read `test.jsonl`: the uploaded rows, then the relabelled traces,
then the synthetic rows. Review:

- **Size**: the row count against the target. A large loss on the relabelling run means
  filtering or schema checks dropped traces; the job log has the cause.
- **Correctness**: reference answers that are correct against the job description, and better
  than the original answers where they differ. Read traces and rows side by side.
- **Coverage**: the rows against the traces on the dimensions that matter for the task (topic,
  length, class balance). Read the synthetic rows with more care than the relabelled ones.

Drop or fix what fails. To change the settings, run the mechanics stage again on its parent
(a new side branch; the old one is abandoned). To edit rows by hand, download the Dataset,
edit `test.jsonl`, and upload the whole directory as a new Dataset with
`distil dataset create --data` (`../references/execution/cli.md` § The Dataset). The traces come
along, so the work continues from the upload, but the platform records no parent for an
upload: note the jump in `run.md`.

## Step 5: Lock It

Record the approved Dataset id in `test-set.md` as the test-set Dataset. It is the input of
`teacher-evaluation.md`, and its `test.jsonl` is the project's test set from here on. Scores
measured on different test sets are not comparable, so when the test rows change later
(`../workflows/model-iterations.md` re-entering `build-a-model.md` Step 5), every earlier
score is stale: re-run the teacher evaluations on the new test set before reading any verdict.

## What Can Be Changed to Improve the Next Iteration

- **More relabelled rows** (`num_test_relabelled`, with more traces): a larger test set makes
  smaller differences between iterations readable.
- **Synthetic rows** (`test_generation_target`, mutators on the test run): rows beyond what the
  traces hold, measuring agreement with the teacher rather than with production.
- **The reference answers** (the relabelling teacher or committee,
  `trace_processing_instructions`): who writes them and with what guidance. They define what
  every score measures.
- **New traces**: a test set can only measure what the traces contain. Traces that show a
  failure the user cares about, relabelled into the test split, add it to the test set.

A new test set makes every earlier score stale, so it needs the user's approval
(`../workflows/model-iterations.md` Step 0).
