# Operation: Create a Dataset

Creates a Dataset without running a job, in one of two ways: upload a prepared directory
(`config.yaml`, `job_description.json` and any of `train.jsonl`, `test.jsonl`, `traces.jsonl`),
or create a traces object and a Dataset from it with a config and a job description. The new
Dataset has no parent Dataset; every later expand, evaluation and training starts from a chain
that begins here (`../references/platform.md` § Entities and jobs).

| | |
|---|---|
| **Called by** | the start phase (the first Dataset), the task-description phase (the root Dataset with the agreed job description), and the test-set and deploy phases when a test set is carried next to new traces |
| **Price** | free: uploads, traces objects and Datasets from traces are not billed |

## Settings that control the output

| Setting | File | What it changes |
|---|---|---|
| `base.task` | `config.yaml` | The task type, required (`../references/task-types.md`). A config without it is refused |
| `base.teacher_model_name` | `config.yaml` | The teacher every later expand and evaluation inherits. Set it explicitly: the library default is not the catalog's default teacher (`../references/model-catalog.md` § Defaults) |
| `base.student_model_name` | `config.yaml` | The student training inherits. Set it explicitly for the same reason |
| `evaluation.llm_as_a_judge_model_name` | `config.yaml` | The judge for every later score. Set it here and never change it |
| `base.llm_num_parallel_requests` | `config.yaml` | Parallel LLM calls in later jobs (16 is a good value); speed only |
| `trace_processing.observation_format` | `config.yaml` | The shape of `traces.jsonl`: `langfuse` for a traces object from an endpoint, `openai_messages` for a converted trace file (`../references/data-preparation/traces.md`) |
| `task_description`, `llm_as_a_judge_instructions` and the other fields | `job_description.json` | What a correct answer is, and what the judge accepts (`../references/job-description.md`) |

A draft config and job description are enough for a first Dataset: the task type, the user's
description of the task, and the defaults above. A later Dataset (the root Dataset, for example)
carries the agreed values. Every other parameter: `../references/configuration.md`.

## Working directory

```
dataset-uploads/<name>/          # the directory uploaded, exactly as sent
├── config.yaml
├── job_description.json
├── train.jsonl, test.jsonl, traces.jsonl   # any subset
└── run.md                        # the command, the new id, the row counts
```

For the traces route, `dataset-uploads/<name>/` holds the config and the job description sent
with `create-from-traces`, and `run.md` holds the traces object id too.

## Step 1: Prepare the input

**From files.** Build the directory per `../references/data-preparation/overview.md` and the file
for the task type (`../references/data-preparation/question-answering.md`,
`../references/data-preparation/chat-completion.md`). Converting the user's files into this
format includes removing fields that leak the answer. The same upload carries rows that were
downloaded and filtered or fixed locally (SKILL.md § Rules); record the script and the counts in
the run log. A missing data file is stored as an empty split; what each job does with an
empty split: `../references/data-preparation/overview.md` § Empty splits.

**From an inference endpoint's records.** Two routes
(`../references/inference-endpoints.md` § From records to a traces object):

- **Direct**: the platform pulls every record into a traces object. A call becomes available
  about 24 hours after it was made. Use `observation_format: langfuse`.
- **Download, convert, upload**: download the records, convert them
  (`../references/data-preparation/traces.md` § From an inference endpoint), and upload the
  converted file as `traces.jsonl` in a directory (the files route above), with
  `observation_format: openai_messages`. Use it when the traces are needed now, must be filtered
  (by `metadata.status` or `metadata.source`), or must sit next to an existing `test.jsonl` and
  `train.jsonl`.

**Carrying a test set next to new traces.** Download the traces object (`distil traces download`)
and upload it together with the current `test.jsonl` and `train.jsonl`, keeping
`observation_format: langfuse` (`../references/execution/cli.md` § The Dataset). The old
Dataset's leftover traces are not carried.

## Step 2: Price it

Free. Nothing to add to the run log's spending.

## Step 3: Submit

From a directory (`../references/execution/cli.md` § The Dataset, § Supplying files):

```bash
distil dataset create --data dataset-uploads/<name>              # Upload successful. Dataset ID: <id>
distil dataset create --config <config.yaml> --job-description <job_description.json> \
  --traces <traces.jsonl> --test <test.jsonl> --train <train.jsonl>
```

The per-file flags cannot be combined with `--data`; with them, `--config` and
`--job-description` are required and only the files named are uploaded.

From a traces object (`../references/execution/cli.md` § The traces object):

```bash
distil traces create-from-inference-endpoint <unique-endpoint-name>   # direct route: the traces id
distil traces upload --traces <traces.jsonl>                          # or a file you have
distil dataset create-from-traces --config dataset-uploads/<name>/config.yaml \
  --job-description dataset-uploads/<name>/job_description.json <traces-id>   # the Dataset id
```

`dataset create` validates the bundle before creating anything: an invalid bundle exits 1 with
the reason (a missing file is caught locally; an unsupported task or a row in the wrong shape
comes back from the platform with the row named). `traces upload` checks only that the file is
there: a malformed trace fails the first expand that reads it, and its log names the cause.

Record the traces id and the Dataset id in the run log's lineage as they are printed. An upload
has no parent on the platform, so write what it was built from in the lineage row. Give the user
the dashboard page, https://app.distillabs.ai/datasets/<id>.

## Step 4: Wait and pull

No job runs. The Dataset's status is `JOB_SUCCESS` as soon as the create returns, and its logs
are empty.

## Step 5: Read the output

Download it and count the rows, so the run log records what exists
(`../references/execution/cli.md` § Reading a Dataset):

```bash
distil dataset download -d dataset-uploads/<name>/check <dataset-id>          # free
wc -l dataset-uploads/<name>/check/{train,test,traces}.jsonl
sort -u dataset-uploads/<name>/check/traces.jsonl | wc -l                     # distinct traces
```

Record the counts of test rows, train rows and distinct traces. Relabelling refuses to run with
fewer distinct traces than it asks for, and every expand uses traces up, so the trace count
limits every later expand (`../references/platform.md` § The expand operations).

Read the config back before the first override on this Dataset
(`distil dataset download-metadata`). A Dataset from `dataset create --data` returns its config
as uploaded, with only the fields that were set; one from `create-from-traces` returns it
normalised, comments dropped. Every override replaces the whole file, and a field it omits takes
the library default, not this Dataset's value (`../references/platform.md` § Overrides).
