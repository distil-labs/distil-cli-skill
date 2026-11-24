# Visual Tasks

Everything about image inputs. One config key controls them: `base.visual_task` (default
`false`). With it on, user messages carry images, the teacher and the judge see them, and the
student is a vision model trained, evaluated and served with them. It is allowed for
`question-answering` only.

The workflow in `../SKILL.md` is written for text. This page lists every change to it for a
visual task: first the changes phase by phase, then the facts they rely on (§ When to use it to
§ Deployment).

## Changes to the workflow

Synthetic data generation is not available: `Synthetic data generation does not support visual
tasks (base.visual_task=true): the generators cannot produce images`. Every row comes from a
relabelled trace or from the user's own rows. Mutators, `synthetic_data_generation_instructions`
and generation targets do not apply, and neither do tools. The rest of the workflow runs as
written, with these changes.

| Phase or protocol | Change |
|---|---|
| Phase 0 | Read this page before the first upload. The first config sets `base.task: question-answering`, `base.visual_task: true`, `trace_processing.observation_format: openai_messages_with_images` and a vision model in every role (§ Models); the create fails otherwise. Upload a sample first and price it with `--dry-run` (§ Data format). The production traces come as a traces file (Entry B): Entry A is not confirmed for images (§ Deployment). When several images come from one source, split the traces by source before the upload (§ Splitting by source) |
| Phase 1 | The input inventory lists the kinds of images (scans, photos, screenshots), their quality and resolution, and what text comes with them. Facts the image cannot show go in the user text (§ Data format) |
| Phase 2 | Every teacher and judge role, and the student, are vision models (§ Models). The judge sees the images (§ Evaluation). The agreed sizes are counts of traces: how many for the test set and how many for training |
| Phase 3 | A gap is filled with more traces of that kind: upload a Dataset with the current test rows and the new traces, and relabel them into the test split. Without such traces, the gap is recorded as a known blind spot. There are no per-gap generation settings |
| `../protocols/make-rows.md` | Only source `traces` applies. Where a fix table points to source `synthgen` for a missing kind of input, ask the user for traces of that kind instead. Read the reviewed rows next to their images |
| `../protocols/blind-review.md` | The reviewers must see the images: save each image in a batch as a file, and give each reviewer the file paths with the row numbers |
| Phase 4 | See below |
| `../protocols/error-analysis.md` | Before attributing an error to the student, compare a few failed predictions with their images at 1024 pixels on the long side: detail lost at that size needs cropping (§ Training), not more rows. A coverage fix is more traces of that kind, not a mutator value |
| Phase 5 | Step 1: until hosted image serving is confirmed (§ Deployment), the final scores are the platform's student evaluation, or a run against a local vLLM server with the extra flag; requests carry the images as content parts. Step 3: serving through an endpoint needs the same confirmation. The next cycle: new images come from files, not from the endpoint's records |

**Phase 4 without generation.** Every train row is a relabelled trace. Iteration 1 relabels the
first-train-set size agreed in Phase 2, not every trace: when there are far more traces than the
model needs, the rest stay in the Dataset for later top-ups. A top-up relabels more of them (the
usual 500–1,000 rows), but it cannot aim at a failure mode, because relabelling draws its traces
at random. Iterations aimed at a failure mode change how the rows are labelled or how the model
is trained instead:

- the relabelling instructions, or a stronger relabelling teacher or a committee: relabel again
  from the Dataset that holds the train traces (the test-set Dataset, or the train-traces
  upload of § Splitting by source);
- `tuning.max_completion_length`, the student or the epochs, on the best Dataset.

Two changes stop the loop and go to the user, because they change the test set or the user's
application: cropping the images (in the traces, the test rows and production alike), and more
traces of a kind the traces lack. Rows from new traces are a top-up: upload a Dataset with the
best Dataset's `train.jsonl` and `test.jsonl` and only the new traces, relabel them into the
train split, and record the upload in the run log's lineage.

## Splitting by source

When several images come from one source (pages of one document, one user's photos), a test
image whose source is also in train inflates the score, and relabelling draws traces at random
from the whole pool. So split the traces into test and train by source before the first upload:

- the first Dataset holds the test traces only, and Phase 3 relabels them into the test split;
- once the test set is agreed, Phase 4 iteration 1 uploads a new Dataset with the agreed
  `test.jsonl` and the train traces, with the same config and job description, and relabels into
  the train split. The upload has no parent on the platform, so record it in the run log's
  lineage;
- rare cases that matter may be oversampled in the train traces; keep the test traces at the
  production proportions.

## When to use it

- When what the model must read is an image: scanned or photographed pages (OCR, handwriting),
  forms, receipts and invoices, screenshots, photos to describe or classify.
- Not when the application already extracts the text reliably: a text task on the extracted
  text is simpler, and its train set can grow with synthetic data generation.

## Models

Every teacher and judge role needs a vision model, and the student a vision student; the lists
are in `model-catalog.md` § Vision compatibility. One non-vision model in any role fails config
load, and the usual default teacher and judge are text-only, so set `base.teacher_model_name`,
`trace_processing.teacher_model_name` and `evaluation.llm_as_a_judge_model_name` explicitly
(`model-catalog.md` § Defaults). Answers with counts or computed values (word counts,
coordinates, totals) can need a higher reasoning effort to be exact (`model-catalog.md`
§ Defaults says when to move the teacher up).

Proprietary teachers are vision-capable for teacher evaluation only, which is how a production
model such as Gemini is scored as the baseline.

## Data format

Traces are production requests and the production model's answers, as for text, with
`trace_processing.observation_format: openai_messages_with_images`
(`data-preparation/traces.md` § Observation formats): the user message's `content` is a list of
text parts and `image_url` parts.

```jsonl
{"messages": [{"role": "user", "content": [{"type": "text", "text": "Transcribe the handwriting on this page."}, {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}]}, {"role": "assistant", "content": "..."}]}
```

Train and test rows carry the images as an `images` list on the user message. Relabelling
writes its rows this way, keeping the trace's images unchanged (a trace whose images the rewrite
moves or drops is dropped), and rows the user uploads use the same form:

```jsonl
{"messages": [{"role": "user", "content": "Transcribe the handwriting on this page.", "images": ["data:image/jpeg;base64,..."]}, {"role": "assistant", "content": "..."}]}
```

The traces a relabelling leaves are written back in the `openai_messages_with_images` form.

- **Data URIs only.** Every image is a `data:image/...;base64,` URI. A URL fails the create with
  `base.visual_task=true requires every image to be inlined as a data:image/...;base64 URI`.
  Base64 adds a third to the file size, so upload a sample and price it with `--dry-run`
  before preparing the full set.
- **Images on user messages only.** A trace with an image on any other message fails to parse.
- **Order inside a message is not kept.** The text parts become one text and the images a
  separate list, so a prompt cannot rely on text sitting between two images. Say in the text
  what each image is, in the order the images are attached.
- **Facts the image cannot show** (an id, the image's pixel size, a source name) go in the
  user text, with an instruction in `task_description` to copy them. Nothing is added to the
  answers after the teacher writes them, and the student learns to produce the whole answer.
- `synthgen.validation_max_total_length` counts the text only, images excluded.

## Identical rows

Traces with the same text and different images are distinct: deduplication compares the image
bytes. The train-test overlap check does not. It compares rows with each image replaced by a
placeholder for its position, so two rows with the same user text and the same answer count as
one row whatever their images, for example blank pages answered the same way. Relabelling drops
a new row that matches a row of the other split, and an upload holding such a pair in train and
test fails with `train and test datasets have N overlapping examples`. When those rows matter, put
something image-specific in the user text, such as an id the application also sends.

## Training

- **SFT only.** RLVR is rejected: `tuning.rlvr_dataset_size` stays at 0.
- **Resolution.** Training stores every image at most 1024 pixels on its long side, downscaled
  and re-encoded, while the teacher saw the full image. Small text, dense pages or fine detail
  can be legible to the teacher and not to the student. When detail is lost, crop the images to
  the region that matters, in the traces and in production alike.
- **Response length.** A full-page transcription or a JSON record is long: raise
  `tuning.max_completion_length` above the longest response (`configuration.md` § tuning), or
  student evaluation truncates it.

## Evaluation

Both judge metrics receive the images with the prompt. When the teacher wrote the test set, the
primary metric is `llm-as-a-judge-reference-free` (`evaluation-metrics.md` § Primary metric per
task), and the judge checks the prediction against the image itself. Write the conventions a
correct answer follows (what to transcribe, what to leave out, how to mark unclear content) into
`llm_as_a_judge_instructions` as well as `task_description`. A pass/fail verdict cannot tell a
page with one wrong word from a perfect one; ask for a graded score when small errors matter
(`job-description.md` § Judge score modes).

## Deployment

- **Local vLLM.** A vision adapter needs one extra `vllm serve` flag,
  `--enable-tower-connector-lora` (`deployment.md` § Serving locally).
- **Requests.** `model_client.py` sends the conversation unchanged, so build each user message
  as content parts: a text part, then one `image_url` part per image, as data URIs. That is the
  shape student evaluation sends. A test row's `images` key is not understood by the server:
  convert it before sending a test row.
- **Hosted endpoints.** Serving image requests through an inference endpoint is not documented
  yet: confirm it with the distil labs team before planning on it.
- **Records.** A traces object made directly from an endpoint (`inference-endpoints.md` § From
  records to a traces object) rejects image content, so the next cycle's images come from
  files, not from the endpoint's records.
