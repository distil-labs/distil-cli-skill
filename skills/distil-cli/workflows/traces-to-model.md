# Workflow: Traces to Model

End-to-end model building starting from production traces. Identical to
`dataset-to-model.md` after the first step, with one extra baseline: the original production
model's score, which the trained student must beat.

Before Step 1, settle the execution backend: install the `distil` CLI per
`../references/execution/README.md` § Choose the backend, and record the result in `run.md`.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
1 Trace Processing ─── raw traces ► seed train/test + original-model baseline
      │      (smoke ► full; review the generated test set with the user)
      ▼
2 Teacher Eval ──► 3 Synthetic Data ──► 4 Training ──► 5 Decide ──► 6 Deploy
└───────────────── dataset-to-model.md Steps 2-6 ──────────────────────────┘
                   extra floor at Decide: below the original model, never Deploy
```

## Step 1: Trace Processing

Run `../stages/trace-processing.md`: convert the raw logs, iterate on smokes until processing
is right, and review the generated test set before accepting the full run. That test set
gates everything downstream. The processed output is the input directory for the next step,
and the original-model evaluation is the baseline to record.

## Steps 2-6: Continue as Dataset to Model

Follow `dataset-to-model.md` from Step 2 (teacher evaluation) onward. The data-preparation
work of its Step 1 is already done. Two trace-specific additions:

- In the training decision (its Step 5), the original-model baseline is a floor. A student
  that does not beat the model it replaces is not deployable, whatever `closed` says
  (`../references/evaluation-metrics.md` § Verdicts). The two can disagree, because a good
  `closed` against a weak base can still lose to the incumbent. The floor wins.
- When iteration points at the data itself (bad test set, bad labels), the fix is usually
  re-running trace processing with different settings rather than editing files by hand.
