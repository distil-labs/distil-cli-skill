# Visual Tasks

Everything about image inputs. One config key controls them: `base.visual_task` (default
`false`). With it on, user messages carry images, the teacher and the judge see them, and the
student is a vision model trained, evaluated and served with them. It is allowed for
`question-answering` only. The end-to-end route is `../workflows/build-a-vision-model.md`.

## When to propose it

- When what the model must read is an image: scanned or photographed pages (OCR, handwriting),
  forms, receipts and invoices, screenshots, photos to describe or classify.
- Not when the application already extracts the text reliably: a text task on the extracted
  text is simpler and its train set can grow with synthetic data generation.

## What it changes, stage by stage

| Stage | With `visual_task: true` |
|---|---|
| Dataset create | Every image is a data URI, images sit on user messages only, `trace_processing.observation_format` is `openai_messages_with_images`, and every model role is vision-capable (§ Data format, § Models). Each fails the create otherwise |
| Relabelling traces | The teacher sees the images and writes or corrects the answer. The row keeps the trace's images unchanged; a trace whose images the rewrite moves or drops is dropped |
| Synthetic data generation | Rejected: `Synthetic data generation does not support visual tasks (base.visual_task=true): the generators cannot produce images`. Every row comes from a relabelled trace or from the user's own rows |
| Teacher evaluation | The teacher answers from the images, and both judge metrics see them (§ Evaluation) |
| Training | SFT only, and every image is stored at most 1024 pixels on its long side (§ Training) |
| Deployment | Local vLLM needs one extra flag, and requests carry the images as content parts (§ Deployment) |

## Models

Every teacher and judge role needs a vision model, and the student a vision student; the lists
are in `model-catalog.md` § Vision compatibility. One non-vision model in any role fails config
load, and the usual default teacher and judge are text-only, so set `base.teacher_model_name`,
`trace_processing.teacher_model_name` and `evaluation.llm_as_a_judge_model_name` explicitly to
the vision defaults (`model-catalog.md` § Defaults). Answers with counts or computed values
(word counts, coordinates, totals) can need a higher reasoning effort to be exact: put a
higher-reasoning variant on the teacher evaluation shortlist for them.

Proprietary teachers are vision-capable for teacher evaluation only, which is how a production
model such as Gemini is scored as the baseline.

## Data format

Traces use the `openai_messages_with_images` observation format
(`data-preparation/traces.md` § Observation formats): the user message's `content` is a list of
text parts and `image_url` parts.

```jsonl
{"messages": [{"role": "user", "content": [{"type": "text", "text": "Transcribe the handwriting on this page."}, {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}]}]}
```

Train and test rows carry the images as an `images` list on the user message. Relabelling
writes its rows this way, and rows the user uploads use the same form:

```jsonl
{"messages": [{"role": "user", "content": "Transcribe the handwriting on this page.", "images": ["data:image/jpeg;base64,..."]}, {"role": "assistant", "content": "..."}]}
```

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

## Traces without an answer

Image traces usually have no answer: the line ends on the user message, and relabelling writes
the answer from the image (one assistant turn is appended). Leave existing answers out of the
traces unless they are worth keeping. Relabelling corrects an observed answer while staying as
close to it as possible, so a trace that carries another model's answer keeps that answer's
habits and mistakes. Answers the user vouches for, such as human transcriptions, are labelled
rows instead (`../stages/build-a-train-set.md` § Where New Rows Come From).

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
  can be legible to the teacher and not to the student. Compare a few failed predictions with
  their images before blaming the student size; when detail is lost, crop the images to the
  region that matters, in the traces and in production alike.
- **Response length.** A full-page transcription or a JSON record is long: raise
  `tuning.max_completion_length` above the longest response (`configuration.md` § tuning), or
  student evaluation truncates it.

## Evaluation

Both judge metrics receive the images with the prompt. When the teacher wrote the test set, the
primary metric is `llm-as-a-judge-reference-free` (`evaluation-metrics.md` § Primary metric per
task), and here the judge checks the prediction against the image itself. Write the conventions
a correct answer follows (what to transcribe, what to leave out, how to mark unclear content)
into `llm_as_a_judge_instructions` as well as `task_description`. A pass/fail verdict cannot
tell a page with one wrong word from a perfect one; ask for a graded score when small errors
matter (`job-description.md` § Judge score modes).

## Deployment

- **Local vLLM.** A vision adapter needs one extra `vllm serve` flag,
  `--enable-tower-connector-lora` (`deployment.md` § Serving locally).
- **Requests.** `model_client.py` sends the conversation unchanged, so build each user message
  as content parts: a text part, then one `image_url` part per image, as data URIs. That is the
  shape student evaluation sends. A test row's `images` key is not understood by the server:
  convert it before using a test row in a smoke test.
- **Hosted endpoints.** Serving image requests through an inference endpoint is not documented
  yet: confirm it with the distil labs team before planning on it.
- **Records.** A traces object made directly from an endpoint (`langfuse`,
  `inference-endpoints.md` § From records to a traces object) rejects image content, so the next
  iteration's images come from files, not from the endpoint's records.

## What does not apply

Everything that tunes generation: `synthgen.train_generation_target`, `test_generation_target`,
mutators, `synthetic_data_generation_instructions`, and a top-up by generation
(`../stages/build-a-train-set.md` § Topping Up). The train set grows with more images,
relabelled into the train split. Tools are not available either: visual tasks are question
answering.
