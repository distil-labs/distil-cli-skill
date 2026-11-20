# Workflow: Build a Vision Model

For OCR and other image-input tasks: transcribing scanned pages, reading forms or receipts,
extracting fields from screenshots, describing or classifying photos. A single pass from the
user's images to a trained, served vision student.

It is `build-a-model.md` entry B with two differences that shape everything else: synthetic
data generation is rejected for visual tasks, and an endpoint's records cannot become image
traces (`../references/visual-tasks.md`). Every row, train and test alike, is an image the user
supplies with an answer the teacher writes by relabelling, and the student is trained on
exactly those rows. The image collection therefore sets the size, the coverage and the quality
ceiling of the model.

Before the first step, install the `distil` CLI and sign in per
`../references/execution/README.md` § Set up the CLI, then settle the execution backend per
§ Choose the backend and record it in `run.md`. Record the chain of Dataset ids in `run.md` as
it grows.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
 1 Collect the images ─── a representative spread from the real source; nothing adds images later
   ▼
 2 Prepare the inputs ─── test and train images as two trace files, no answers; vision models
   ▼
 3 Test set ─── Dataset of the test images ► relabel into the test split, no synthetic rows
   │            gate: the user approves it against the images; locked from here on
   ▼
 4 Teacher evaluation ─── vision candidates and the production model, one run each
   │  └─ gate: pick the teacher, PROCEED only
   ▼
 5 Train set ─── new Dataset: the locked test set + the train images ► relabel: smoke ► full
   ▼
 6 Train and decide ─┬─ Deploy candidate, above the baseline ─► 7
   │                 └─ Not good enough ─► model-iteration.md, with more images
   ▼
 7 Serve ─── local vLLM; the next iteration's images come from files again
```

## Step 1: Collect the Images

The images are the dataset: nothing generates more, so the collection has to cover what the
model will see in production, in roughly the proportions it will see it.

- **Spread.** Sample at random from the real source rather than picking examples. Cover every
  document or image type, layout, language, handwriting or print style, image quality (blurred,
  dark, skewed, cropped) and the empty cases (blank pages, nothing to extract).
- **Rare cases.** A case that matters but is rare (tables, diagrams, crossed-out text,
  multi-column pages) needs enough examples to be learned. Oversample it in the train images if
  needed, but keep the test images at the production proportions so the test score stays
  honest.
- **Grouped splits.** Split the images into test and train by source document (booklet, user,
  session), not by image. Pages of one document share handwriting and content, and a test page
  whose document is also in train inflates the score.
- **Counts.** A few hundred test images, between 100 and 1000 (`../stages/build-a-test-set.md`
  Step 2), and as many train images as the budget allows. Relabelling drops some traces, so
  expect fewer rows than images.
- **A holdout.** The platform has a train and a test split only. To check the final model on
  images no iteration has seen, keep a further set of images aside.

Images the user already has answers for, and vouches for (human transcriptions, reviewed
labels), are labelled rows, not traces (`../stages/build-a-train-set.md` § Where New Rows Come
From).

## Step 2: Prepare the Inputs

Write the job description and the config as in `build-a-model.md` Step 4, with these settings
(`../references/visual-tasks.md` § Models, § Data format):

- `base.task: question-answering` and `base.visual_task: true`;
- a vision model in `base.teacher_model_name`, `trace_processing.teacher_model_name` and
  `evaluation.llm_as_a_judge_model_name`, and a vision student. The defaults are text-only and
  fail config load;
- `trace_processing.observation_format: openai_messages_with_images`;
- `task_description`: the whole system prompt, with the output format and the conventions a
  correct answer follows. The same conventions go into `llm_as_a_judge_instructions`
  (`../references/job-description.md` § What each field feeds). For a JSON response, also
  `synthgen.output_is_json: true` (`../references/data-preparation/question-answering.md`);
- `tuning.max_completion_length` above the longest response in tokens.

Then write two trace files, `test-traces.jsonl` from the test images and `train-traces.jsonl`
from the train images: one user message per line with the user prompt and the image or images,
and no assistant turn (`../references/visual-tasks.md` § Traces without an answer). Show a few
lines of each to the user, with the config and the job description, before uploading.

## Step 3: Build the Test Set

Create a Dataset from the config, the job description and `test-traces.jsonl` only
(`../references/execution/cli.md` § The Dataset). Relabelling picks traces at random from the
whole pool, so a pool holding both splits' images would mix documents across the splits.

Run `../stages/build-a-test-set.md` on it with relabelling only:
`trace_processing.num_test_relabelled` is the number of distinct test traces and
`synthgen.test_generation_target` is 0. At its Step 4 review, read the rows next to their
images: the reference answers decide what every score measures. Lock the approved Dataset per
its Step 5.

## Step 4: Teacher Evaluation

`build-a-model.md` Step 6, unchanged: `../stages/teacher-evaluation.md` on the test-set Dataset,
one run per vision candidate and one with the production model as the baseline. The judge sees
the images, and the primary metric is `llm-as-a-judge-reference-free` because the teacher wrote
the references (`../references/visual-tasks.md` § Evaluation).

**Gate:** continue only on PROCEED (`../references/evaluation-metrics.md` § Verdicts), with the
best teacher in both `base.teacher_model_name` and `trace_processing.teacher_model_name` for
Step 5.

## Step 5: Build the Train Set

Create a new Dataset from the config with the chosen teacher, the job description, the locked
`test.jsonl` and `train-traces.jsonl` (`../references/execution/cli.md` § The Dataset). It is an
upload, so the platform records no parent: note the jump in `run.md`.

Run `../stages/build-a-train-set.md` on it with its Step 3 only: `../stages/relabel-traces.md`
on the train split, a smoke of 128 traces first, then the full run with
`trace_processing.num_train_relabelled` set to the number of distinct train traces. Skip its
Step 4: there is no generation. At its Step 5 analysis, read a sample of rows next to their
images, and compare the row count with the traces: the job log says how many were dropped.

## Step 6: Train and Decide

`build-a-model.md` Step 8 on the Dataset Step 5 produced: `../stages/model-training.md`, then
the verdict with the user. Training is SFT only, on images downscaled to 1024 pixels on their
long side (`../references/visual-tasks.md` § Training).

When the model is not good enough, `../stages/model-iteration.md` runs as usual, within what a
visual task allows (`../references/visual-tasks.md` § What does not apply):

- **More or better images**: the main lever. For the cases the predictions get wrong, add
  traces and relabel them into the train split of a new Dataset that holds the current
  `train.jsonl`, `test.jsonl` and the new traces.
- **The system prompt or the teacher**: relabel the train split again, and the test split only
  with the user's approval (`../stages/model-iteration.md` § Changing the Test Set).
- **The student and its settings**: a larger vision student, more epochs, a longer
  `max_completion_length`.

For `autoresearch.md` on a visual task, its Step 1 runs Steps 1-6 of this workflow instead of
`build-a-model.md`.

## Step 7: Serve

Serve the deploy candidate with `../stages/local-deployment.md`, adding the vision flag to the
`vllm serve` command and sending images as content parts (`../references/visual-tasks.md`
§ Deployment). Hosted serving through `build-a-model.md` Step 9 needs confirming for image
requests first.

The next model starts from new images in files, not from an endpoint's records: create a new
Dataset with the current `test.jsonl` and `train.jsonl` next to the new traces, keep the test
set, and continue at Step 5 as a top-up. Extending the test set makes every earlier score stale
(`../stages/build-a-test-set.md` Step 5).
