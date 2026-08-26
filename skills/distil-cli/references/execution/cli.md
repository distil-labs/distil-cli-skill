# Execution Backend: distil CLI

How stages run on the platform through the `distil` command. Stage files link here by operation
name. Every stage is an entity created by one command, and every entity is created either from
local files or by running a job over the entity before it. The only prerequisite is a distil
labs account, which `distil signup` creates from the terminal.

This is the default backend and `backend-api.md` is the alternative. Take that one when the
user prefers it, when the work is already scripted in Python, or when the CLI cannot be
installed. `README.md` § Choose the backend decides between them, and the choice goes in
`run.md`.

Two things differ from the API backend. The CLI stages and uploads the files for you, so a
directory is one argument. And an override is a file path rather than a JSON body: read the
parent's config to disk, edit it, then pass it to `--config`.

## Prerequisites

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
distil signup                                        # create an account; opens a browser
distil auth                                          # sign in; opens a browser
distil auth --email <email> --password <password>    # sign in, headless
distil whoami                                        # prints the current user
```

`distil signup` and `distil auth` are the same browser handoff against a different page. Both
finish signed in, so signup needs no separate login. There is no headless `signup`.

The snippets also use `jq`. `README.md` § Choose the backend gives the full install procedure
and the conditions that make the API backend necessary.

The CLI holds its own session in `~/.config/distillabs/token` (under `$XDG_CONFIG_HOME` when
that is set) and refreshes it, so a run spanning hours needs no second login. The API backend
re-authenticates per request. `distil --version` prints the installed version, and
`distil update` replaces the binary in place.

This file describes CLI 0.25.2. Check the version before trusting a flag.

## Preamble

Paths in the snippets below are illustrative. The stage files own the working-directory layout.

Each command runs in its own shell, so nothing below carries state from one command to the
next. Record each entity id in `run.md` as it is created, and pass it back as a literal
argument.

The job creates that name a parent id take `--output json`, so `jq -r .id` reads the new id
back. The commands that read local files do not (§ Which commands speak JSON). They print the
id in a sentence, so read it from the output.

Poll a job by asking for its status:

```bash
distil <group> status --output json <id> | jq -r .status
```

Call that every 20 seconds or so. `JOB_SUCCESS` means the outputs are readable. `JOB_FAILURE`
or `JOB_STOPPED` means stop and read the log. Anything else means the job is still running, so
report where it is and check again. Jobs run for tens of minutes (§ Submitting jobs), so poll
between other work and tell the user what is happening rather than blocking on one long wait.

```bash
distil <group> logs --output json <id> | jq -r .logs
```

Read the `status` field, not the exit code of `distil <group> status`. A status command that
reaches the platform exits 0 whatever the job did. Creates report through the exit code: a
rejected bundle or an invalid config exits 1 with the reason on stderr. Downloads do too. A
download of something the job has not produced yet names the reason and exits 1 rather than
writing an empty file, so the exit code is worth trusting.

## The entity model

The commands for each stage. What the entities are and how they chain: `../platform.md`
§ Entities and jobs. The `Override` column says whether a job's configuration can be varied at
submission. See § Overrides.

| Stage | Entity | Submit | Read | Override |
|---|---|---|---|---|
| trace-processing | PreparedTraces → SeedDataset | `distil traces upload --data <dir>`, then `distil seed-dataset create-from-traces <traces-id>` | `distil seed-dataset {status,logs,metrics,download,download-metadata,download-traces-predictions}` | yes |
| (job input) | SeedDataset | `distil seed-dataset create --data <dir>` | `distil seed-dataset {status,download,download-metadata}` | the files on disk |
| teacher-evaluation | TeacherEvaluation | `distil teacher-evaluation create-from-seed-dataset <seed-dataset-id>` | `distil teacher-evaluation {status,logs,metrics,download-metadata,download-predictions}` | yes |
| synthetic-data-generation | TrainingDataset | `distil training-dataset create-from-seed-dataset <seed-dataset-id>`, or `distil training-dataset create --data <dir>` from local files | `distil training-dataset {status,logs,metrics,sample,download,download-metadata}` | yes |
| test-set-expansion | TrainingDataset | as synthgen, from a **new** SeedDataset with train and test inverted | as synthgen | data change, so no |
| model-training | SLM | `distil slm create-from-training-dataset <training-dataset-id>` | `distil slm {status,logs,metrics,download,download-metadata,download-predictions}` | yes |
| model-deployment | Deployment | `distil deployment create-from-slm <slm-id>` | `distil deployment {status,endpoint,logs}`, `distil deployment delete` | no config of its own |

`upload` and `uploads` are aliases of `seed-dataset`, and `create-from-upload` of
`create-from-seed-dataset`. Write the `seed-dataset` spelling. Each group also answers to its
plural (`slms`, `deployments`, `teacher-evaluations`, and so on), and `list` answers to `ls`.

### Which commands speak JSON

`--output json` is registered on the read commands (`list`, `show`, `status`, `logs`,
`metrics`, `sample`, `endpoint`, plus `whoami` and `credits-balance`) and on the job creates
that name a parent id: `teacher-evaluation create-from-seed-dataset`,
`training-dataset create-from-seed-dataset`, `slm create-from-training-dataset` and
`deployment create-from-slm`.

It is not registered on the commands that read local files (`traces upload`,
`seed-dataset create`, `training-dataset create`, `slm create`), on
`seed-dataset create-from-traces`, or on any download command. Passing it there fails with
`No flag registered for --output` and creates nothing, so read the id out of what those
commands print instead.

Without `--output json` a read prints a panel for a human reader and suggests the next command
underneath. Parse the JSON form.

### Supplying files

The CLI does the presigned-URL exchange that the API backend does by hand. `--data <dir>` names
a directory and the CLI reads the files out of it by name:

| Command | Required in `--data <dir>` | Optional |
|---|---|---|
| `distil traces upload` | `traces.jsonl`, `config.yaml`, `job_description.json` | `test.jsonl` |
| `distil seed-dataset create` | `train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json` | `unstructured.jsonl` |
| `distil training-dataset create` | `train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json` | none (an `unstructured.jsonl` left in the directory is skipped with a warning) |
| `distil slm create` | `model.tar`, `config.yaml` | none |

`config.yml` is accepted for `config.yaml`. The per-file flags (`--traces`, `--train`,
`--test`, `--unstructured`, `--config`, `--job-description`, `--model`) replace one path each,
and combining them with `--data` is refused rather than merged. A file missing from the
directory is named before anything uploads.

"Required" means the file has to be there, not that it has to hold rows. `train.jsonl` and
`test.jsonl` can be empty (`../data-preparation/overview.md` § Empty splits). Leaving one out
is still an error.

The download commands write these same names, so a downloaded directory feeds straight back
into the matching `create --data`. That round trip is how a change to the *data* is made, since
no override can reach it.

`distil training-dataset create --data <dir>` builds a TrainingDataset from local files instead
of from a SeedDataset. It returns a dataset id and no SeedDataset id. The SeedDataset id is the
address of teacher evaluation and of every later iteration, so this is not a substitute for
`create-from-seed-dataset` in the main path. Its one use in this skill is the training smoke,
§ The TrainingDataset.

`distil slm create --data <dir>` is not part of the stage path either. It registers an existing
`model.tar` and `config.yaml` as an SLM. It uploads a model. It does not train one.

## Credits

`distil credits-balance` prints the calls remaining on each metered route. The read is free and
answers at zero balance. How metering works, the route table and the starting balances:
`../platform.md` § Credits.

```bash
distil credits-balance
# prepared_traces_post                       100    <- one row per metered route,
# seed_datasets_post                         100       sorted by name
# teacher_evaluations_post                    20
# training_datasets_from_seed_datasets_post    5
# slms_from_training_datasets_post             2
```

`--output json` returns `{"balances": {…}}` under the same keys. Only metered routes appear. A
route the platform never charges for is absent rather than reported as unlimited, so a missing
key is not a zero, and an account with nothing metered reads `No metered endpoints.` Read an
unfamiliar key defensively.

**The route keys are the platform's, not the CLI's.** They agree with the command names, so
`seed_datasets_*` is what `distil seed-dataset` spends.

Read the balance before synthgen, not before training. A synthgen run with no training credit
left leaves the generation spend with nothing to use it. A submission against an exhausted
route is refused.

## Submitting jobs

Every job is created by naming the id of the entity before it. Nothing uploads at this point,
because the parent's files are already on the platform.

| Stage | Command group | Typical duration |
|---|---|---|
| Trace processing | `seed-dataset` | 45 min |
| Teacher evaluation | `teacher-evaluation` | 30 min |
| Synthetic data generation | `training-dataset` | 90 min |
| Model training | `slm` | 90 min |
| Deployment | `deployment` | 40 min |

### Trace processing

Upload the traces, then process them into a SeedDataset.

```bash
distil traces upload --data traces-input              # prints the PreparedTraces id
distil seed-dataset create-from-traces <traces-id>   # prints the SeedDataset id
distil seed-dataset status --output json <seed-dataset-id> | jq -r .status
```

`traces-input/` holds `traces.jsonl`, `config.yaml`, `job_description.json` and an optional
`test.jsonl`. Supplying that test file replaces the generated test split and makes
`num_traces_as_testing_base` inert. Neither command takes `--output json`, so read each id from
the output it prints.

Iterating is an override on the same PreparedTraces, which re-uploads nothing:

```bash
distil traces download-metadata -d traces/iter-2 <traces-id>   # free
# in traces/iter-2/config.yaml, under trace_processing:
#   relevance_filtering: true
distil seed-dataset create-from-traces \
  --config traces/iter-2/config.yaml <traces-id>               # prints the new SeedDataset id
distil seed-dataset status --output json <new-seed-dataset-id> | jq -r .status
```

A PreparedTraces owns the trace file it was uploaded with, so a different set of traces means
`distil traces upload` again. Config and job-description changes over the same traces are
overrides.

### The SeedDataset

A job-input directory (`../data-preparation/overview.md`), uploaded whole:

```bash
distil seed-dataset create --data input      # prints the SeedDataset id; record it in run.md
```

`seed-dataset create` validates the bundle before creating anything and prints the platform's
validation error when it fails. That is this backend's dryrun. Three failures look different
and all name the cause:

```
# a file missing from the directory: caught locally, nothing uploads
Required file not found: test.jsonl in input

# a task the platform does not have
Only the following tasks are supported: ['classification', 'question-answering-open-book', …]

# a row in the wrong shape, named down to the row index
1 validation error for JobInputParser
job_input.1.data.question-answering.train_dataset.0.messages
  Field required [type=missing, input_value={'wrong': 'shape'}, input_type=dict]
```

Each exits 1 and creates nothing. Correct the directory and run the command again.

The route is metered (`seed_datasets_post`), but only a successful create spends a credit. The
balance is checked first, and the call is recorded only after validation passes. So validating
a broken bundle repeatedly is free, and a refusal here means the balance was already zero
before the bundle was read.

A direct SeedDataset runs no job. It reports `JOB_SUCCESS` as soon as the command returns and
needs no poll, its `logs` are empty and its `metrics` are null. A trace-derived one runs a job
and needs a poll.

### The TrainingDataset (training smoke calibration)

Normally a TrainingDataset is *produced* by synthgen. It can also be created from local files,
which is how `../../stages/model-training.md` Step 3 builds the truncated dataset it calibrates
memory settings on:

```bash
distil training-dataset download -d training/smoke-1/input <training-dataset-id>
# Truncate in place: keep the ~100 longest rows of train.jsonl and the ~10
# longest of test.jsonl, leaving config.yaml and job_description.json as
# downloaded. `create --data` then reads the same directory.
distil training-dataset create --data training/smoke-1/input   # prints the new dataset id
```

The download is metered on `training_datasets_download_get`, the create on
`training_datasets_post`. The create runs the same validation as `seed-dataset create`
(`../data-preparation/overview.md` § Validation rules) and prints the failure the same way.

### Overrides: how a job is parameterised

Four job creates each accept `--config <file>` (`-c`) and `--job-description <file>`:
`seed-dataset create-from-traces`, `teacher-evaluation create-from-seed-dataset`,
`training-dataset create-from-seed-dataset` and `slm create-from-training-dataset`. Both flags
are optional and independent. Omit one and the job inherits the parent's.

Each replaces the parent's file whole. The CLI does not merge, and whatever the file omits
takes a *library default* rather than the parent's value (`../platform.md` § Overrides). So an
override is read-edit-resend, never hand-written:

```
distil <group> download-metadata -d <dir> <id>   →   edit one field   →   pass to --config
```

`download-metadata` writes the entity's `config.yaml` and `job_description.json` and costs no
credits. What it writes is the *fully expanded* config: every default materialized, keys
sorted, comments dropped. That is why editing that file and sending it back preserves
everything you did not touch.

PreparedTraces is the exception: it echoes the config it was staged with, comments intact and
nothing expanded. A trace-processing override therefore edits your own file.

```bash
distil seed-dataset download-metadata -d synthgen/smoke-1 <seed-dataset-id>
# edit synthgen/smoke-1/config.yaml
distil training-dataset create-from-seed-dataset --output json \
  --config synthgen/smoke-1/config.yaml <seed-dataset-id> | jq -r .id
# record the id it prints in run.md
```

Two failure shapes:

- A config naming one field under `base` is refused, because `base.task` has no default:

  ```
  Invalid config: 1 validation error for DistilConfiguration
  base.task
    Field required [type=missing, input_value={'student_model_name': 'Qwen3-0.6B'}, input_type=dict]
  ```

- A config carrying a complete `base` but no `synthgen` or `tuning` section is accepted, and
  those sections revert silently. Submitted against a parent that set `generation_target: 512`,
  `output_is_json: true`, `per_device_train_batch_size: 8` and a list of `mutators`, a
  base-only override ran with `generation_target: 10000`, `output_is_json: false`,
  `per_device_train_batch_size: 1` and no mutators. Nothing errored.

Because the read-back config is complete, checking what a submission actually ran with is a
diff rather than an audit:

```bash
distil training-dataset download-metadata -d check <training-dataset-id>
diff synthgen/smoke-1/config.yaml check/config.yaml
```

An override reaches the config and the job description and nothing else. A change to the data
is a new SeedDataset. That is why test-set expansion inverts train and test into a fresh one
rather than overriding the existing one. `distil seed-dataset download -d <dir> <id>` writes a
directory that `distil seed-dataset create --data <dir>` accepts unchanged, so making one is an
edit between two commands.

Since the parent never changes, an iteration is the same parent submitted again: `smoke-1`,
`smoke-2` and `full-1` all name one SeedDataset. Record each child's UUID in the iteration's
`run.md`.

### Teacher evaluation

```bash
distil teacher-evaluation create-from-seed-dataset --output json <seed-dataset-id> | jq -r .id
distil teacher-evaluation status --output json <teacher-evaluation-id> | jq -r .status
```

A different teacher model is a config change. Sharper judge instructions are a job-description
change. Both are overrides on the same SeedDataset, and neither needs a new one:

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
distil training-dataset status --output json <training-dataset-id> | jq -r .status
```

A smoke run is a config override on the same SeedDataset:

```bash
distil seed-dataset download-metadata -d synthgen/smoke-1 <seed-dataset-id>
# in synthgen/smoke-1/config.yaml, under synthgen:
#   generation_target: 64
#   generation_iteration_size: 16
distil training-dataset create-from-seed-dataset --output json \
  --config synthgen/smoke-1/config.yaml <seed-dataset-id> | jq -r .id
```

The full run is the same command with the intended `generation_target` and
`generation_iteration_size`, or with no `--config` at all if the SeedDataset already carries
them.

### Model training

```bash
distil slm create-from-training-dataset --output json <training-dataset-id> | jq -r .id
distil slm status --output json <slm-id> | jq -r .status
```

Training reads its baseline config from the TrainingDataset, which inherited it from the
SeedDataset synthgen ran over. Setting sensible `tuning` values before synthgen means the common
case needs no override.

A sweep is N submissions against the same dataset, one per student. Read the dataset's config
once, then write one file per student, because the CLI sends each file whole:

```bash
distil training-dataset download-metadata -d sweep <training-dataset-id>
# copy sweep/config.yaml once per student, changing base.student_model_name in each

for config in sweep/student-*.yaml; do
  distil slm create-from-training-dataset --output json --config "$config" \
    <training-dataset-id> | jq -r .id
done
```

`per_device_train_batch_size`, `memory_optimized_training` and `use_qlora` vary the same way
(read, edit, resend), which is how OOM is handled without touching the data. Submissions run
concurrently, so start them all and poll them after.

## Monitor

Poll `distil <group> status --output json <id>` every 20 seconds. The status values and how to
run a poller: `../platform.md` § Job status.

```bash
distil training-dataset status --output json <training-dataset-id> | jq -r .status
```

`status` answers `{"status": "JOB_RUNNING"}`. Deployments answer
`{"deployment_status": …, "endpoint_status": …}` and have no `status` field at all, so read
`.deployment_status` for those.

`distil <group> logs --output json <id>` returns `{"logs": "…"}`, the whole job log as one
string, cause at the end. It fills while the job runs, so it is also how a long job is watched.

`distil <group> list --output json` returns one object per entity, newest first, carrying `id`,
`created_at`, the parent's id under its own key (`seed_dataset_id`, `training_dataset_id`,
`slm_id`, and so on) and, where an entity can come from more than one kind of parent, a
`source` naming which. It is how a lost id is recovered and how a child is traced back to its
parent. It carries no status, so checking state costs one `status` call per id.

## Output layout

The CLI fetches nothing by object-storage path. Metrics arrive as JSON on stdout, and a
download command writes files to disk. Which outputs each stage produces, and the conditions on
them: `../platform.md` § What each stage produces.

| Entity | Metrics | Data files | Config + job description | Predictions |
|---|---|---|---|---|
| PreparedTraces | none | `traces download` | `traces download-metadata` | none |
| SeedDataset | `seed-dataset metrics` (trace-derived only) | `seed-dataset download` | `seed-dataset download-metadata` | `seed-dataset download-traces-predictions` |
| TeacherEvaluation | `teacher-evaluation metrics` | none | `teacher-evaluation download-metadata` | `teacher-evaluation download-predictions` |
| TrainingDataset | `training-dataset metrics` (byte sizes) | `training-dataset download` (metered), `training-dataset sample` (free) | `training-dataset download-metadata` | none |
| SLM | `slm metrics` | `slm download` | `slm download-metadata` | `slm download-predictions` |
| Deployment | none | none | none | `deployment endpoint` |

The `Config + job description` column is the free read every override starts from. Each
`download-metadata` writes exactly `config.yaml` and `job_description.json`, on every entity,
and nothing else.

Every command that writes a directory takes `--destination <dir>` (`-d`) and otherwise names one
after the entity: `<id>-traces`, `<id>-data`, `<id>-slm`, `<id>-metadata`. The `*-predictions`
commands write a single file, so they take `--file-name` instead and default to
`<id>-<kind>-predictions.jsonl` in the current directory. Downloads overwrite what is already
there.

Metrics and data downloads answer only once that entity's own job reaches `JOB_SUCCESS`. Until
then a metrics field is `null`, and the matching download names the job's state
(`SLM is still training`, `No data is available for this training dataset`) and exits 1 rather
than writing an empty file.

`download-metadata` is the partial exception. Config and job description are settled at
submission, so a TeacherEvaluation or SLM answers within seconds of the create, and an override
can be read back from a job still running. A TrainingDataset does not answer until its job
finishes, and a failed job never answers at all, so record what you submitted rather than
planning to read it back.

Two entries carry conditions:

- **`seed-dataset metrics` is empty for a directly created SeedDataset.** Both fields are
  `null`: the command reports the trace-processing job and a direct SeedDataset runs none. On a
  trace-derived one they hold the score of the model that produced the traces on your test set,
  which is the production model being distilled from, not the untuned student. The student
  cannot be scored before it is trained.
- **`slm metrics` gives two scores but only the tuned model's predictions.** The base-vs-tuned
  gap is reportable as numbers. A base-model failure case is not.

## Fetch metrics

Aggregated metrics arrive as the `*_performance` object: a flat metric-name-to-score dict for
every task except classification. The matching download command writes the per-example detail.

```bash
distil teacher-evaluation metrics --output json <teacher-evaluation-id> | jq .teacher_performance
# {"rouge": 1, "binary": 0.82, "llm-as-a-judge": 1, "llm-as-a-judge-reference-free": 1}

distil slm metrics --output json <slm-id> \
  | jq '{base: .base_model_performance, tuned: .tuned_model_performance}'

distil teacher-evaluation download-predictions <teacher-evaluation-id>
```

A metric the run did not compute is `null` rather than absent. Filter those before averaging.

Three things about the predictions file:

- It is JSONL, one test example per line, carrying `prompt`, `completion`, `prediction` and
  that example's own score under each metric name. Read one row and work from what is there.
- `prompt` is the full prompt as a JSON-encoded message list, not the user text alone, and
  `completion` and `prediction` are JSON-encoded assistant messages. Parse them. Do not compare
  them as raw strings.
- For classification the performance object is not flat. Alongside `accuracy` it carries one
  key per class label, each holding `{precision, recall, f1-score, support}`. There is no
  `confusion_matrix` and no `classification_report`. Iterate by key rather than assuming
  numeric values: `accuracy` is a float and every other entry is a dict.

`metrics --output json` also carries the `*_download_url` the download command uses. It is
presigned and expires after an hour. Take it directly only to hand the data to another program.

### Reading a TrainingDataset

`training-dataset sample` is free and answers `{"rows": [{"messages": […]}, …]}`: at most 128
train rows, drawn deterministically from the first 384, never test rows. `download` costs a
credit on `training_datasets_download_get`.

```bash
distil training-dataset sample --output json <training-dataset-id> > sample.json
jq '.rows | length' sample.json
distil training-dataset metrics --output json <training-dataset-id> | jq .train_data_size_bytes
```

`metrics` reports bytes, not rows. At smoke scale the dataset is smaller than the cap, so the
sample is the whole thing and its row count is exact. For a full run, divide the byte count by
the mean row size in the sample and treat the result as an estimate.

## Fetch model artifacts

```bash
distil slm download --destination model <slm-id>
```

This writes `model.tar` and `config.yaml`. The tarball expands to the layout `../deployment.md`
§ Artifacts describes. The config is not inside it, and both files are needed to serve the
model. The tarball is gigabytes, about 1.2 GB for a Qwen3-0.6B run, and the command checks free
disk space before it starts writing.

### The inference client on its own

`slm download-metadata` writes the model's client alongside its config and job description, a
few kilobytes instead of the gigabytes of the tarball:

```bash
distil slm download-metadata --destination model <slm-id>
# model/config.yaml, model/job_description.json, model/model_client.py
```

What the client is for and how to call it: `../deployment.md`.

## Reuse synthetic data for training-only runs

Retrain the same dataset under a new config. Nothing is downloaded, copied or reassembled:

```bash
distil training-dataset download-metadata -d retrain <training-dataset-id>
# in retrain/config.yaml, under tuning:
#   num_train_epochs: 6
distil slm create-from-training-dataset --output json \
  --config retrain/config.yaml <training-dataset-id> | jq -r .id
```

This is the sweep command from § Submitting jobs with one field changed instead of the student.

## Deploy (hosted)

```bash
distil deployment create-from-slm --output json <slm-id> | jq -r .id
distil deployment status --output json <deployment-id> | jq -r .deployment_status

distil deployment endpoint --output json <deployment-id>
# {"url": "https://…", "api_key": "…"}
```

`endpoint` carries the `url` and the `api_key`. Take both from that output. Query the
deployment through the model's own client rather than a hand-built request:
`../deployment.md` § Serving hosted has the call and says why. The client takes an
OpenAI-style base URL, so drop the trailing slash from the URL and append `/v1`.

```bash
distil slm download-metadata -d model <slm-id>       # § The inference client on its own
uv run model/model_client.py --base-url https://<endpoint>/v1 --api-key <api-key> \
  --conversation '[{"role": "user", "content": "…"}]'
```

```bash
distil deployment delete <deployment-id>
```

The deployment serves the model with vLLM, and the job does not return until vLLM answers, so
`JOB_SUCCESS` means serving rather than merely scheduled. Before that, `endpoint` answers
`{"url": null, "api_key": null}` in JSON and `This deployment is not serving an endpoint.` for a
human. It exits 0 either way, so poll the status rather than probing the endpoint. The API key
protects the endpoint. The tunnel has no authentication of its own, and the URL is open to all.

**A deployment is a session, not a permanent endpoint.** It stops after six hours, or after one
hour with no traffic. It cannot be restarted, and a new deployment carries a new URL and a new
key. Replacing a stopped one spends another `deployments_from_slms_post` credit, so collect the
inputs to send before you create the deployment.

CAUTION: delete the deployment when finished. A running deployment bills until its idle timeout.
After the delete, `deployment_status` stays `JOB_SUCCESS`. `endpoint_status` going to `stopped`
is what confirms it is down.

To serve the model locally, download `model.tar` as above, then read `../deployment.md`
§ Serving locally.
