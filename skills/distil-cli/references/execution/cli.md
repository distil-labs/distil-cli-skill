# distil CLI

The commands that run each stage. How the platform behaves (entities, expand operations, job
status, smoke runs, overrides, credits, outputs): `../platform.md`. Installing the CLI and
signing in: `README.md` § Set up the CLI.

This is the default backend and `backend-api.md` is the alternative. Take that one when the
user prefers it, or when the work is already scripted in Python. `README.md` § Choose the backend
decides between the two, and the choice goes in `run.md`. Two things differ from the API
backend: the CLI stages and uploads the files for you, so a directory is one argument, and an
override is a file path rather than a JSON body.

Check `distil --help` and `distil <group> --help` before trusting a flag; `distil update`
installs the latest release.

## Prerequisites

```bash
distil whoami                                        # prints the current user
distil access-token                                  # prints a fresh access token for the API backend
distil logout
```

The snippets also use `jq`. Each command runs in its own shell, so nothing carries state from
one command to the next: record each entity id in `run.md` and pass it back as a literal
argument. Paths in the snippets are illustrative; the stage files own the working directories.

## The entity model

| Stage | Entity | Submit | Read | Override |
|---|---|---|---|---|
| (job input) | PreparedTraces | `distil traces upload --traces <file>`, or `distil traces create-from-inference-endpoint <unique-endpoint-name>` | `distil traces {list,show,download}` | no config of its own |
| (job input) | Dataset | `distil dataset create --data <dir>`, or `distil dataset create-from-traces --config <file> --job-description <file> <traces-id>` | `distil dataset {list,show,status,logs,metrics,download,download-metadata}` | the files on disk |
| relabel-traces | Dataset → Dataset | `distil dataset relabel-traces-{train,test} [--smoke] <dataset-id>` | same | yes |
| synthetic-data-generation | Dataset → Dataset | `distil dataset generate-synthetic-data-{train,test} [--smoke] <dataset-id>` | same | yes |
| teacher-evaluation | TeacherEvaluation | `distil teacher-evaluation create-from-dataset <dataset-id>` | `distil teacher-evaluation {list,show,status,logs,metrics,download-metadata,download-predictions}` | yes |
| model-training | SLM | `distil slm create-from-dataset [--smoke] <dataset-id>` | `distil slm {list,show,status,logs,metrics,download,download-metadata,download-predictions}` | yes |
| inference-endpoint | Deployment, InferenceEndpoint | `distil deployment create-from-slm <slm-id>`, then `distil inference-endpoint create-from-deployment <deployment-id>`; or `distil inference-endpoint create` with no SLM | `distil deployment {status,logs}`, `distil inference-endpoint {list,show,download-traces}` | no config of its own |

Seed datasets and training datasets: `../migrating-old-entities.md`.

### Which commands speak JSON

`--output json` (`-o`) is registered on:

- the read commands: `list`, `show`, `status`, `logs`, `metrics`, plus `whoami` and
  `credits-balance`;
- the creates that name a parent id: the four `dataset relabel-traces-*` and
  `dataset generate-synthetic-data-*` commands, `teacher-evaluation create-from-dataset`,
  `slm create-from-dataset`, `deployment create-from-slm`;
- `inference-endpoint create` and `inference-endpoint create-from-deployment`.

It is not registered on the commands that read local files (`traces upload`,
`dataset create`, `dataset create-from-traces`, `slm create`), on
`traces create-from-inference-endpoint`, on `inference-endpoint link-api-key` and
`unlink-api-key`, or on any download command. Passing it there fails with
`No flag registered for --output` and creates nothing. Those commands print the new id in a
sentence (`dataset create`: `Upload successful. Dataset ID: <id>`; `dataset create-from-traces`
and `traces upload`: `… created. ID: <id>`); read it from the output.

Without `--output json` a read prints a panel for a human reader. Parse the JSON form.

### Supplying files

The CLI does the presigned-URL exchange that the API backend does by hand. `--data <dir>` names
a directory, and the CLI uploads the files in it by name:

| Command | Required in `--data <dir>` | Optional |
|---|---|---|
| `distil traces upload` | `traces.jsonl` | none |
| `distil dataset create` | `config.yaml`, `job_description.json` | `train.jsonl`, `test.jsonl`, `traces.jsonl`; a missing one is stored as an empty split |
| `distil slm create` | `model.tar`, `config.yaml` | none |

`config.yml` is accepted for `config.yaml`. The per-file flags (`--traces`, `--train`, `--test`,
`--config`, `--job-description`, `--model`) replace one path each, and combining them with
`--data` is refused; with per-file flags, `dataset create` needs `--config` and
`--job-description` and any of the other three. Only the files given are uploaded: the
platform reads a missing data file as "this Dataset has none of this kind". A file named but
not found is reported before anything uploads.

The download commands write these same names, so a downloaded directory is accepted unchanged
by `dataset create --data`. That is how a change to the rows is made, since no override reaches
the data.

`distil dataset create-from-traces` takes `--config` and `--job-description` as required
inputs, because a PreparedTraces holds neither file. `distil slm create --data <dir>` registers
an existing `model.tar` and `config.yaml` as an SLM; it does not train one.

## Credits

```bash
distil credits-balance
# datasets_from_datasets_generate_synthetic_data_post    5    <- excerpt; one row per metered
# datasets_from_datasets_relabel_traces_post             5       route, sorted by name
# datasets_from_datasets_smoke_post                     20
# datasets_from_prepared_traces_post                     5
# datasets_post                                          5
# prepared_traces_post                                 100
# slms_from_datasets_post                                5
# slms_from_datasets_smoke_post                         20
# teacher_evaluations_post                               5
```

`--output json` returns `{"balances": {…}}` under the same keys. An account with nothing
metered prints `No metered endpoints.` The route keys and starting balances:
`../platform.md` § Credits. A `--smoke` submission spends the `*_smoke_post` key of its route.

## Creating the job inputs

### The traces object

```bash
distil traces upload --traces traces.jsonl                          # prints the PreparedTraces id
distil traces create-from-inference-endpoint <unique-endpoint-name> # prints the PreparedTraces id
distil traces list --output json
```

`traces upload` checks only that the file is there. A malformed trace fails the first expand
that reads it, and the job log names the cause. Which endpoint records `create-from-inference-
endpoint` includes, and when: `../inference-endpoints.md` § From records to a traces object.

### The Dataset

From a traces object, with the config and the job description written first:

```bash
distil dataset create-from-traces --config input/config.yaml \
  --job-description input/job_description.json <traces-id>         # prints the Dataset id
```

No job runs: the traces are copied, train and test are empty, and the Dataset is ready at
once. The trace file's shape must match `trace_processing.observation_format` in the config
(`../data-preparation/traces.md`): `langfuse` for a traces object from an endpoint. A
`traces.jsonl` downloaded from such a traces object uploads again with `dataset create
--traces` and the same `langfuse` setting, which is how a test set is put next to new
endpoint traces (`../../workflows/build-a-model.md` Step 10).

From files on disk (`../data-preparation/overview.md`), any subset of the three data files:

```bash
distil dataset create --data input          # Upload successful. Dataset ID: <id>
distil dataset create --config input/config.yaml \
  --job-description input/job_description.json --traces traces.jsonl --test test.jsonl
```

`dataset create` validates the bundle before creating anything: the files are parsed exactly
as the jobs will parse them, and an invalid bundle exits 1 with the reason on stderr and
creates nothing (a file missing from the directory is caught locally; an unsupported task, a
row in the wrong shape or a classification split missing a class come back from the platform
with the row named).

## Submitting jobs

Every job is created by naming the id of its parent. Nothing uploads, because the parent's files
are already on the platform.

### Relabel traces

```bash
distil dataset relabel-traces-test --output json <dataset-id> | jq -r .id     # the new Dataset
distil dataset relabel-traces-train --output json <dataset-id> | jq -r .id
distil dataset relabel-traces-test --smoke --output json <dataset-id> | jq -r .id
distil dataset status --output json <new-dataset-id> | jq -r .status
```

The count comes from `trace_processing.num_test_relabelled` or `num_train_relabelled` in the
parent's config, or from the override; `--smoke` forces it to 128 (`../platform.md` § Smoke
runs).

### Synthetic data generation

```bash
distil dataset generate-synthetic-data-train --output json <dataset-id> | jq -r .id
distil dataset generate-synthetic-data-test --output json <dataset-id> | jq -r .id
distil dataset generate-synthetic-data-train --smoke --output json \
  --config synthgen/smoke-2/config.yaml <dataset-id> | jq -r .id
```

The target comes from `synthgen.train_generation_target` or `test_generation_target`;
`--smoke` forces it to 128 and combines with the override flags.

### Overrides

Six creates accept `--config <file>` (`-c`) and `--job-description <file>`: the four expand
commands, `teacher-evaluation create-from-dataset` and `slm create-from-dataset`. Both flags are
optional and independent; an omitted one is inherited from the parent. Each file replaces the
parent's whole (`../platform.md` § Overrides), so read the parent's file, edit it, and send it
back:

```bash
distil dataset download-metadata -d synthgen/smoke-1 <dataset-id>   # free
# edit synthgen/smoke-1/config.yaml
distil dataset generate-synthetic-data-train --output json \
  --config synthgen/smoke-1/config.yaml <dataset-id> | jq -r .id
```

`dataset download-metadata` writes `config.yaml` and `job_description.json` and nothing else.
The config of a Dataset written by an expand is fully expanded: every default written out, keys
sorted, comments dropped. A Dataset from `dataset create --data` returns its config as uploaded;
one from `dataset create-from-traces` returns it normalised (comments dropped).

A config with a complete `base` but no `synthgen` or `tuning` section is accepted, and those
sections take their defaults with no error: against a parent with `train_generation_target: 512`
and a list of `mutators`, a base-only override runs with `train_generation_target: 10000` and
no mutators. A config missing `base.task` is refused with `Invalid config: ... base.task Field
required`.

To check what a submission ran with, read the new Dataset's config back and compare:

```bash
distil dataset download-metadata -d check <new-dataset-id>
diff synthgen/smoke-1/config.yaml check/config.yaml
```

### Teacher evaluation

```bash
distil teacher-evaluation create-from-dataset --output json <dataset-id> | jq -r .id
distil teacher-evaluation status --output json <teacher-evaluation-id> | jq -r .status
```

A different teacher is a config override and different judge instructions a job-description
override, both on the same Dataset:

```bash
distil dataset download-metadata -d te/iter-2 <dataset-id>
# te/iter-2/config.yaml           → base.teacher_model_name
# te/iter-2/job_description.json  → llm_as_a_judge_instructions
distil teacher-evaluation create-from-dataset --output json \
  --config te/iter-2/config.yaml \
  --job-description te/iter-2/job_description.json <dataset-id> | jq -r .id
```

### Model training

```bash
distil slm create-from-dataset --output json <dataset-id> | jq -r .id
distil slm status --output json <slm-id> | jq -r .status
```

Training takes its config from the Dataset. A sweep is one submission per config against the
same Dataset; submissions run concurrently:

```bash
distil dataset download-metadata -d sweep <dataset-id>
# copy sweep/config.yaml once per student, changing base.student_model_name in each

for config in sweep/student-*.yaml; do
  distil slm create-from-dataset --output json --config "$config" <dataset-id> | jq -r .id
done
```

A smoke takes the same arguments plus `--smoke` (`../platform.md` § Smoke runs):

```bash
distil slm create-from-dataset --smoke --output json \
  --config sweep/student-qwen3.5-4b.yaml <dataset-id> | jq -r .id
```

## Monitor

```bash
distil <group> status --output json <id> | jq -r .status
distil <group> logs --output json <id> | jq -r .logs
distil <group> list --output json
```

`status` answers `{"status": "JOB_RUNNING"}`; the values and what they mean:
`../platform.md` § Job status. A Dataset that no job produced (an upload, or one from traces)
answers `JOB_SUCCESS` at once and has empty logs. Deployments answer
`{"deployment_status": …, "endpoint_status": …}` with no `status` field, so read
`.deployment_status` for them.

Read the `status` field, not the exit code: a status command that reaches the platform exits 0
whatever the job did. Creates and downloads report through the exit code: a rejected bundle, an
invalid config or a download of something not produced yet exits 1 with the reason on stderr
(`Dataset <id> is not ready. Its job status is JOB_RUNNING.`). With `--output json` an error is
printed to stdout as `{"error": …}` and the command exits 1, so `| jq -r .id` prints `null`:
check the exit code.

`logs` returns `{"logs": "…"}`, the whole job log as one string. It fills while the job runs;
in the first seconds it reads `No logs for this dataset.`

`list` returns one object per entity, newest first, with `id`, `created_at` and the parent's id
under its own key (`dataset_id`, `slm_id`, and so on). A Dataset also carries its `operation`
and its parent as `parent_dataset_id` or `parent_prepared_traces_id`; a PreparedTraces its
`source`. `dataset show <id>` prints the same for one Dataset, plus its status (`JOB_SUCCESS` at once
for a Dataset no job produced). `list` recovers a lost id, and `show` walks a chain back to its root.

## Output layout

What each entity produces, and when: `../platform.md` § What each entity produces.

| Entity | Metrics | Data files | Config + job description | Predictions |
|---|---|---|---|---|
| PreparedTraces | none | `traces download` | none | none |
| Dataset | `dataset metrics` (byte sizes) | `dataset download` | `dataset download-metadata` | none |
| TeacherEvaluation | `teacher-evaluation metrics` | none | `teacher-evaluation download-metadata` | `teacher-evaluation download-predictions` |
| SLM | `slm metrics` | `slm download` | `slm download-metadata` | `slm download-predictions` |
| Deployment | none | none | none | none |

Every command that writes a directory takes `--destination <dir>` (`-d`) and otherwise names one
after the entity: `<id>-traces`, `<id>-data`, `<id>-slm`, `<id>-metadata`. The `*-predictions`
commands write a single file, take `--file-name`, and default to `<id>-<kind>-predictions.jsonl`
in the current directory. Downloads overwrite what is there. A download before the job has
produced its output names the state (`SLM is still training`) and exits 1.

### Reading a Dataset

```bash
distil dataset download -d data <dataset-id>                          # free
wc -l data/train.jsonl data/test.jsonl data/traces.jsonl
distil dataset metrics --output json <dataset-id>
# {"train_data_size_bytes": …, "test_data_size_bytes": …, "traces_size_bytes": …}
```

`dataset download` writes all five files; an empty split is an empty file. The rows of a split are the parent's rows followed by the ones the expand
added; to see only the new rows, download the parent too and diff.

## Fetch metrics

```bash
distil teacher-evaluation metrics --output json <teacher-evaluation-id> | jq .teacher_performance
# {"rouge": 1, "binary": 0.82, "llm-as-a-judge": 1, "llm-as-a-judge-reference-free": 1}

distil slm metrics --output json <slm-id> \
  | jq '{base: .base_model_performance, tuned: .tuned_model_performance}'

distil teacher-evaluation download-predictions <teacher-evaluation-id>
distil slm download-predictions <slm-id>
```

Without `--output json`, `metrics` prints every metric, `llm-as-a-judge-reference-free`
included. The `*_performance` object maps
metric name to score; for classification its shape differs
(`../evaluation-metrics.md` § Metrics by task). A metric the run did not compute is `null`
rather than absent, so filter nulls before averaging.

Two things about a predictions file:

- It is JSONL, one test example per line, with `prompt`, `completion`, `prediction` and that
  example's score under each metric name. `prompt` is the test row's messages without the
  few-shot examples, and `completion` and `prediction` are assistant messages. Read one row and
  work from what is there.
- `slm download-predictions` holds the tuned model's predictions only. The base model's scores
  are in `slm metrics`; its per-example predictions are not available.

## Fetch model artifacts

```bash
distil slm download --destination model <slm-id>
```

Writes `model.tar` and `config.yaml`. The tarball's contents: `../deployment.md` § Artifacts.
The command checks free disk space before it starts writing.

### The inference client on its own

```bash
distil slm download-metadata --destination model <slm-id>
# model/config.yaml, model/job_description.json, model/model_client.py
```

A few kilobytes instead of the tarball; `model_client.py` is there once the SLM reaches
`JOB_SUCCESS`. How to use the client: `../deployment.md`.

## Reuse the data for training-only runs

Retrain the same Dataset under a new config; nothing is downloaded or regenerated:

```bash
distil dataset download-metadata -d retrain <dataset-id>
# in retrain/config.yaml, under tuning:
#   num_train_epochs: 6
distil slm create-from-dataset --output json --config retrain/config.yaml <dataset-id> | jq -r .id
```

## Inference endpoints

How endpoints behave: `../inference-endpoints.md`. The procedure:
`../../stages/inference-endpoint.md`.

```bash
distil inference-endpoint create --name <prefix> --fallback-model <owner/model>
distil inference-endpoint create-from-deployment --name <prefix> --fallback-model <owner/model> <deployment-id>
distil inference-endpoint list --output json
distil inference-endpoint show --output json <unique-endpoint-name>
distil inference-endpoint link-api-key <unique-endpoint-name> <key-name>
distil inference-endpoint unlink-api-key <unique-endpoint-name> <key-name>
distil inference-endpoint download-traces <unique-endpoint-name>
distil traces create-from-inference-endpoint <unique-endpoint-name>
```

### Deploy an SLM

```bash
distil deployment create-from-slm --output json <slm-id> | jq -r .id
distil deployment status --output json <deployment-id> | jq -r .deployment_status
distil deployment delete <deployment-id>
```

Poll `deployment_status` to `JOB_SUCCESS` before creating the endpoint;
`create-from-deployment` refuses earlier with `Deployment is not ready`. After
`deployment delete`, `deployment_status` stays `JOB_SUCCESS`; `endpoint_status` reading
`stopped` confirms the deployment is down.

### Create an endpoint

```bash
distil inference-endpoint create --name support --fallback-model "openai/gpt-4.1-mini"
# Endpoint Name:   support-yeOdAS
# Endpoint:        https://inference.distillabs.ai/v1/chat/completions

distil inference-endpoint create-from-deployment --output json --name support-slm \
  --fallback-model "openai/gpt-4.1-mini" <deployment-id> | jq -r .unique_endpoint_name
# support-slm-Qk3bZ1
```

`create` makes a collecting endpoint, `create-from-deployment` a serving one. `--name` is the
prefix; record the unique endpoint name the command prints. `--fallback-model` is an OpenRouter
slug in `owner/model` form. `--trace-sample-rate <0-1>` sets the fraction of calls recorded,
default 1; `show` reports it as `trace_sampling_rate`.

`create` also takes `--primary-url` and `--primary-api-key`, which connect a model already
deployed elsewhere as the primary. No stage in this skill uses them.

### Keys

```bash
distil api-keys create <key-name>      # prints the secret once, writes <key-name>.json
distil api-keys list --output json
distil api-keys delete <key-name>
```

`--no-file` on `create` skips writing `<key-name>.json`, for a key that goes straight into a
secret store. Never echo a secret into the transcript or into `run.md`; name the file instead.

### The call the application makes

```bash
curl https://inference.distillabs.ai/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"model": "support-yeOdAS", "messages": [{"role": "user", "content": "Say hi."}]}'
```

`model` carries the unique endpoint name; the rest is an ordinary chat completions request. A
trained model behind a serving endpoint is queried with its own client:

```bash
distil slm download-metadata -d model <slm-id>
uv run model/model_client.py --base-url https://inference.distillabs.ai/v1 \
  --api-key <endpoint-api-key> --model support-slm-Qk3bZ1 \
  --conversation '[{"role": "user", "content": "…"}]'
```

### Download the traces

```bash
distil inference-endpoint download-traces <unique-endpoint-name>                  # last day, newest 1000
distil inference-endpoint download-traces --from-start-time 2026-09-01 --all <unique-endpoint-name>
distil inference-endpoint download-traces --count 5000 <unique-endpoint-name>     # -c
distil inference-endpoint download-traces --file-name raw-traces.jsonl <unique-endpoint-name>
```

Writes JSONL, one record per call, to `<unique-endpoint-name>-traces.jsonl` unless `--file-name`
says otherwise. `--from-start-time` and `--to-start-time` take an ISO 8601 date or date-time;
the default window is the last day. Within it, `--count` caps the records (default 1000) and
`--all` takes every one; passing both is refused. With no records in the window, no file is
written.

The file is endpoint records, not a trace file: convert it per
`../data-preparation/traces.md` § From an inference endpoint, then upload the result with
`distil traces upload --traces` or `distil dataset create --traces`.

### Traces from an endpoint

```bash
distil traces create-from-inference-endpoint <unique-endpoint-name>
# Prepared traces created. ID: <traces-id>
```

Creates a PreparedTraces from the endpoint's records; the platform pulls them, so nothing is
downloaded or converted. The next step is `dataset create-from-traces` with a config whose
`trace_processing.observation_format` is `langfuse`. Which records it includes, and when:
`../inference-endpoints.md` § From records to a traces object.
