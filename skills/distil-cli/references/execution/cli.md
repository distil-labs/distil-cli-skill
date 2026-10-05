# distil CLI

The commands that run each stage. How the platform behaves (entities, job status, smoke runs,
overrides, credits, outputs): `../platform.md`. Installing the CLI and signing in: `README.md`
§ Set up the CLI.

This file describes CLI 0.31.0. Check `distil --version` before trusting a flag.

## Prerequisites

```bash
distil whoami                                        # prints the current user
distil logout
```

The snippets also use `jq`. Each command runs in its own shell, so nothing carries state from
one command to the next: record each entity id in `run.md` and pass it back as a literal
argument. Paths in the snippets are illustrative; the stage files own the working directories.

## The entity model

| Stage | Entity | Submit | Read | Override |
|---|---|---|---|---|
| (job input) | PreparedTraces | `distil traces upload --data <dir>`, or `distil traces create-from-inference-endpoint --data <dir> <unique-endpoint-name>` | `distil traces {status,download,download-metadata}` | the files on disk |
| test-set-from-traces | PreparedTraces → PreparedTraces | `distil traces expand-test-set <traces-id>` | `distil traces {status,logs,metrics,download,download-metadata,download-predictions}` | yes |
| trace-processing | PreparedTraces → SeedDataset | `distil seed-dataset create-from-traces <traces-id>` | `distil seed-dataset {status,logs,download,download-metadata}` | yes |
| (job input) | SeedDataset | `distil seed-dataset create --data <dir>` | `distil seed-dataset {status,download,download-metadata}` | the files on disk |
| teacher-evaluation | TeacherEvaluation | `distil teacher-evaluation create-from-seed-dataset <seed-dataset-id>` | `distil teacher-evaluation {status,logs,metrics,download-metadata,download-predictions}` | yes |
| synthetic-data-generation | TrainingDataset | `distil training-dataset create-from-seed-dataset [--smoke] <seed-dataset-id>` | `distil training-dataset {status,logs,metrics,sample,download,download-metadata}` | yes |
| model-training | SLM | `distil slm create-from-training-dataset [--smoke] <training-dataset-id>` | `distil slm {status,logs,metrics,download,download-metadata,download-predictions}` | yes |
| inference-endpoint | Deployment, InferenceEndpoint | `distil deployment create-from-slm <slm-id>`, then `distil inference-endpoint create-from-deployment <deployment-id>`; or `distil inference-endpoint create` with no SLM | `distil deployment {status,logs}`, `distil inference-endpoint {list,show,download-traces}` | no config of its own |

### Which commands speak JSON

`--output json` is registered on:

- the read commands: `list`, `show`, `status`, `logs`, `metrics`, `sample`, plus `whoami` and
  `credits-balance`;
- the creates that name a parent id: `traces expand-test-set`,
  `teacher-evaluation create-from-seed-dataset`, `training-dataset create-from-seed-dataset`,
  `slm create-from-training-dataset`, `deployment create-from-slm`;
- `inference-endpoint create` and `inference-endpoint create-from-deployment`.

It is not registered on the commands that read local files (`traces upload`,
`traces create-from-inference-endpoint`, `seed-dataset create`, `training-dataset create`,
`slm create`), on `seed-dataset create-from-traces`, on `inference-endpoint link-api-key` and
`unlink-api-key`, or on any download command. Passing it there fails with
`No flag registered for --output` and creates nothing. Those commands print the new id in a
sentence; read it from the output.

Without `--output json` a read prints a panel for a human reader. Parse the JSON form.

### Supplying files

`--data <dir>` names a directory, and the CLI uploads the files in it by name:

| Command | Required in `--data <dir>` | Optional |
|---|---|---|
| `distil traces upload` | `traces.jsonl`, `config.yaml`, `job_description.json` | `test.jsonl` |
| `distil traces create-from-inference-endpoint` | `config.yaml`, `job_description.json` | `test.jsonl`; a `traces.jsonl` in the directory is refused |
| `distil seed-dataset create` | `train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json` | `unstructured.jsonl` |
| `distil training-dataset create` | `train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json` | none; an `unstructured.jsonl` is skipped with a warning |
| `distil slm create` | `model.tar`, `config.yaml` | none |

`config.yml` is accepted for `config.yaml`. The per-file flags (`--traces`, `--train`, `--test`,
`--unstructured`, `--config`, `--job-description`, `--model`) replace one path each, and
combining them with `--data` is refused. A missing file is named before anything uploads.
"Required" means present, not non-empty: `train.jsonl` and `test.jsonl` can be empty
(`../data-preparation/overview.md` § Empty splits).

The download commands write these same names, so a downloaded directory is accepted unchanged
by the matching `create --data`. That is how a change to the data is made, since no override
reaches the data.

`distil training-dataset create --data <dir>` creates a TrainingDataset from local files.
`distil slm create --data <dir>` registers an existing `model.tar` and `config.yaml` as an SLM;
it does not train one.

## Credits

```bash
distil credits-balance
# prepared_traces_post                              100    <- excerpt; one row per metered
# prepared_traces_with_expanded_test_set_post        20       route, sorted by name
# seed_datasets_from_prepared_traces_post            20
# seed_datasets_post                                100
# training_datasets_from_seed_datasets_post           5
# slms_from_training_datasets_post                    2
```

`--output json` returns `{"balances": {…}}` under the same keys. An account with nothing
metered prints `No metered endpoints.` The route keys and starting balances:
`../platform.md` § Credits. A `--smoke` submission spends the `*_smoke_post` key of its route.

## Submitting jobs

Every job is created by naming the id of its parent. Nothing uploads, because the parent's files
are already on the platform. Typical durations: `../platform.md` § Job status.

### Test set from traces and trace processing

```bash
distil traces upload --data traces-input                    # prints the PreparedTraces id
distil traces expand-test-set --output json <traces-id> | jq -r .id   # the updated PreparedTraces
distil traces status --output json <updated-traces-id> | jq -r .status
distil seed-dataset create-from-traces <updated-traces-id>  # prints the SeedDataset id
distil seed-dataset status --output json <seed-dataset-id> | jq -r .status
```

`traces upload` checks only that the files are there and that the config parses. A malformed
trace, test row or job description fails the first job that reads it, and the job log names
the cause.

`expand-test-set` creates a PreparedTraces whose `parent_prepared_traces_id` is the first and
whose `source` is `test_set_expansion`. Its `test.jsonl` holds the supplied test rows, the
relabelled traces and the synthetic rows; its `traces.jsonl` holds the traces the job did not
use. `seed-dataset create-from-traces` copies the PreparedTraces' `test.jsonl` to the
SeedDataset unchanged, so without one the test split is empty.

Both jobs take `--config` and `--job-description` overrides on the same PreparedTraces:

```bash
distil traces download-metadata -d traces/iter-2 <traces-id>   # free
# in traces/iter-2/config.yaml, under trace_processing:
#   relevance_filtering: true
distil seed-dataset create-from-traces \
  --config traces/iter-2/config.yaml <traces-id>               # prints the new SeedDataset id
```

A different set of traces is a new `distil traces upload`.

### The SeedDataset

A job-input directory (`../data-preparation/overview.md`), uploaded whole:

```bash
distil seed-dataset create --data input      # prints the SeedDataset id
```

`seed-dataset create` validates the bundle before creating anything. The three failures, each
exiting 1 and creating nothing:

```
# a file missing from the directory: caught locally, nothing uploads
Required file not found: test.jsonl in input

# a task the platform does not have
Only the following tasks are supported: ['classification', 'question-answering', …]

# a row in the wrong shape, named down to the row index
1 validation error for JobInputParser
job_input.1.data.question-answering.train_dataset.0.messages
  Field required [type=missing, input_value={'wrong': 'shape'}, input_type=dict]
```

### Overrides

Five creates accept `--config <file>` (`-c`) and `--job-description <file>`:
`traces expand-test-set`, `seed-dataset create-from-traces`,
`teacher-evaluation create-from-seed-dataset`, `training-dataset create-from-seed-dataset` and
`slm create-from-training-dataset`. Both flags are optional and independent; an omitted one is
inherited from the parent. Each file replaces the parent's whole (`../platform.md`
§ Overrides), so read the parent's file, edit it, and send it back:

```bash
distil seed-dataset download-metadata -d synthgen/smoke-1 <seed-dataset-id>   # free
# edit synthgen/smoke-1/config.yaml
distil training-dataset create-from-seed-dataset --output json \
  --config synthgen/smoke-1/config.yaml <seed-dataset-id> | jq -r .id
```

`download-metadata` writes `config.yaml` and `job_description.json` and nothing else. The config
is fully expanded: every default written out, keys sorted, comments dropped. An uploaded
PreparedTraces returns its config as uploaded; an updated one created by `expand-test-set`
returns the fully expanded config.

A config with a complete `base` but no `synthgen` or `tuning` section is accepted, and those
sections take their defaults with no error: against a parent with `generation_target: 512` and
a list of `mutators`, a base-only override ran with `generation_target: 10000` and no mutators.
A config missing `base.task` is refused with `Invalid config: ... base.task Field required`.

To check what a submission ran with, read its config back and compare:

```bash
distil training-dataset download-metadata -d check <training-dataset-id>
diff synthgen/smoke-1/config.yaml check/config.yaml
```

### Teacher evaluation

```bash
distil teacher-evaluation create-from-seed-dataset --output json <seed-dataset-id> | jq -r .id
distil teacher-evaluation status --output json <teacher-evaluation-id> | jq -r .status
```

A different teacher is a config override and different judge instructions a job-description
override, both on the same SeedDataset:

```bash
distil seed-dataset download-metadata -d te/iter-2 <seed-dataset-id>
# te/iter-2/config.yaml           → base.teacher_model_name
# te/iter-2/job_description.json  → llm_as_a_judge_instructions
distil teacher-evaluation create-from-seed-dataset --output json \
  --config te/iter-2/config.yaml \
  --job-description te/iter-2/job_description.json <seed-dataset-id> | jq -r .id
```

### Synthetic data generation

```bash
distil training-dataset create-from-seed-dataset --output json <seed-dataset-id> | jq -r .id
distil training-dataset create-from-seed-dataset --smoke --output json \
  --config synthgen/smoke-2/config.yaml <seed-dataset-id> | jq -r .id
distil training-dataset status --output json <training-dataset-id> | jq -r .status
```

`--smoke` runs the smoke version of the job (`../platform.md` § Smoke runs) and combines with
the override flags.

### Model training

```bash
distil slm create-from-training-dataset --output json <training-dataset-id> | jq -r .id
distil slm status --output json <slm-id> | jq -r .status
```

Training takes its config from the TrainingDataset, which inherited it from the SeedDataset. A
sweep is one submission per config against the same dataset; submissions run concurrently:

```bash
distil training-dataset download-metadata -d sweep <training-dataset-id>
# copy sweep/config.yaml once per student, changing base.student_model_name in each

for config in sweep/student-*.yaml; do
  distil slm create-from-training-dataset --output json --config "$config" \
    <training-dataset-id> | jq -r .id
done
```

A smoke takes the same arguments plus `--smoke` (`../platform.md` § Smoke runs):

```bash
distil slm create-from-training-dataset --smoke --output json \
  --config sweep/student-qwen3.5-4b.yaml <training-dataset-id> | jq -r .id
```

## Monitor

```bash
distil <group> status --output json <id> | jq -r .status
distil <group> logs --output json <id> | jq -r .logs
distil <group> list --output json
```

`status` answers `{"status": "JOB_RUNNING"}`; the values and
what they mean: `../platform.md` § Job status. Deployments answer
`{"deployment_status": …, "endpoint_status": …}` with no `status` field, so read
`.deployment_status` for them.

Read the `status` field, not the exit code: a status command that reaches the platform exits 0
whatever the job did. Creates and downloads report through the exit code: a rejected bundle, an
invalid config or a download of something not produced yet exits 1 with the reason on stderr,
and a download never writes an empty file.

`logs` returns `{"logs": "…"}`, the whole job log as one string. It fills while the job runs.

`list` returns one object per entity, newest first, with `id`, `created_at`, the parent's id
under its own key (`seed_dataset_id`, `training_dataset_id`, `slm_id`, and so on) and, where an
entity can come from more than one kind of parent, a `source`. It recovers a lost id. It
carries no status.

## Output layout

What each entity produces, and when: `../platform.md` § What each stage produces.

| Entity | Metrics | Data files | Config + job description | Predictions |
|---|---|---|---|---|
| PreparedTraces | `traces metrics` | `traces download` | `traces download-metadata` | `traces download-predictions` |
| SeedDataset | none | `seed-dataset download` | `seed-dataset download-metadata` | none |
| TeacherEvaluation | `teacher-evaluation metrics` | none | `teacher-evaluation download-metadata` | `teacher-evaluation download-predictions` |
| TrainingDataset | `training-dataset metrics` (byte sizes) | `training-dataset download` (metered), `training-dataset sample` (free) | `training-dataset download-metadata` | none |
| SLM | `slm metrics` | `slm download` | `slm download-metadata` | `slm download-predictions` |
| Deployment | none | none | none | none |

Every command that writes a directory takes `--destination <dir>` (`-d`) and otherwise names one
after the entity: `<id>-traces`, `<id>-data`, `<id>-slm`, `<id>-metadata`. The `*-predictions`
commands write a single file, take `--file-name`, and default to `<id>-<kind>-predictions.jsonl`
in the current directory. Downloads overwrite what is there. A download before the job has
produced its output names the state (`SLM is still training`,
`No data is available for this training dataset`) and exits 1.

## Fetch metrics

```bash
distil teacher-evaluation metrics --output json <teacher-evaluation-id> | jq .teacher_performance
# {"rouge": 1, "binary": 0.82, "llm-as-a-judge": 1, "llm-as-a-judge-reference-free": 1}

distil slm metrics --output json <slm-id> \
  | jq '{base: .base_model_performance, tuned: .tuned_model_performance}'

distil traces metrics --output json <updated-traces-id> | jq .base_model_performance

distil teacher-evaluation download-predictions <teacher-evaluation-id>
distil slm download-predictions <slm-id>
distil traces download-predictions <updated-traces-id>
```

The `*_performance` object maps metric name to score; for classification its shape differs
(`../evaluation-metrics.md` § Metrics by task). A metric the run did not compute is `null`
rather than absent, so filter nulls before averaging.

Two things about a predictions file:

- It is JSONL, one test example per line, with `prompt`, `completion`, `prediction` and that
  example's score under each metric name. `prompt` is the full prompt as a message list, and
  `completion` and `prediction` are assistant messages. Read one row and work from what is there.
- `slm download-predictions` holds the tuned model's predictions only. The base model's scores
  are in `slm metrics`; its per-example predictions are not available.

### Reading a TrainingDataset

```bash
distil training-dataset sample --output json <training-dataset-id> > sample.json   # free
jq '.rows | length' sample.json
distil training-dataset metrics --output json <training-dataset-id> | jq .train_data_size_bytes
distil training-dataset download -d data <training-dataset-id>     # training_datasets_download_get
```

`sample` answers `{"rows": [{"messages": […]}, …]}`. What the sample holds and how to estimate
the row count from it: `../platform.md` § What each stage produces.

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

A few kilobytes instead of the tarball. How to use the client: `../deployment.md`.

## Reuse synthetic data for training-only runs

Retrain the same dataset under a new config; nothing is downloaded or regenerated:

```bash
distil training-dataset download-metadata -d retrain <training-dataset-id>
# in retrain/config.yaml, under tuning:
#   num_train_epochs: 6
distil slm create-from-training-dataset --output json \
  --config retrain/config.yaml <training-dataset-id> | jq -r .id
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
distil traces create-from-inference-endpoint --data <dir> <unique-endpoint-name>
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
`../data-preparation/traces.md` § From an inference endpoint, then `distil traces upload` the
result.

### Traces from an endpoint

```bash
distil traces create-from-inference-endpoint --data traces-input <unique-endpoint-name>
# Prepared traces created. ID: <traces-id>
```

Creates a PreparedTraces from the endpoint's records. `traces-input/` holds the files in
§ Supplying files; set `trace_processing.observation_format: langfuse` in its config.
`--config`, `--job-description` and `--test` replace `--data` with one path each. Which records
it includes, and when: `../inference-endpoints.md` § From records to a traces object.
