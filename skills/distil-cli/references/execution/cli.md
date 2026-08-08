# Execution Backend: distil CLI

How stages run on the platform through the `distil` command; stage files link here by operation
name. Every stage is an entity created by one command, and every entity is created either from
local files or by running a job over the entity before it. The only prerequisite is a distil
labs account.

This is the default backend and `backend-api.md` is the alternative — take that one when the
user prefers it, when the work is already scripted in Python, or when the CLI cannot be
installed. `README.md` § Choose the backend decides between them, and the choice goes in
`run.md`.

Two things differ from the API backend and they govern everything below. The CLI stages and
uploads the files for you, so a directory is one argument. And an override is a file path
rather than a JSON body: read the parent's config to disk, edit it, pass it to `--config`.

## Prerequisites

```bash
curl -fsSL https://cli-assets.distillabs.ai/install.sh | sh
distil auth                                          # opens a browser
distil auth --email <email> --password <password>    # headless
distil whoami                                        # prints the current user
```

The snippets also use `jq`. `README.md` § Choose the backend gives the full install procedure
and the conditions that make the API backend necessary.

The CLI holds its own session in `~/.config/distillabs/token` (under `$XDG_CONFIG_HOME` when
that is set) and refreshes it, so a run spanning hours needs no second login; the API backend
re-authenticates per request. `distil --version` prints the installed version and
`distil update` replaces the binary in place.

This file describes **0.24.1**. Check the version before trusting a flag.

## Preamble

Paths in the snippets below are illustrative; the stage files own the working-directory
layout. Every snippet uses these two helpers.

```bash
# The commands that read local files print their new id in a sentence instead of
# accepting --output json (§ The entity model), so grep reads the id back.
dl_id() { grep -oE '[0-9a-f]{8}(-[0-9a-f]{4}){3}-[0-9a-f]{12}' | head -1; }

# Block until a job reaches a final state, printing the job log on failure.
# $1 is the command group. $4 is the status field; only deployments change it.
# The state variable is not called `status`: that name is read-only in zsh and
# the assignment would kill the calling script.
dl_poll() {
  local group=$1 id=$2 timeout=$3 field=${4:-status} state
  local deadline=$(( $(date +%s) + timeout ))
  while [ "$(date +%s)" -lt "$deadline" ]; do
    state=$(distil "$group" status --output json "$id" | jq -r ".$field")
    case "$state" in
      JOB_SUCCESS) return 0 ;;
      JOB_FAILURE|JOB_STOPPED)
        distil "$group" logs --output json "$id" | jq -r .logs
        echo "$group/$id $state" >&2
        return 1 ;;
      ''|null)                      # unknown id, or no such field: fail now
        echo "$group/$id: no $field" >&2
        return 1 ;;
    esac
    echo "$group/$id: $state"
    sleep 20
  done
  echo "$group/$id did not finish within ${timeout}s" >&2
  return 1
}
```

Read the `status` field, not the exit code of `distil <group> status`: a status command that
reaches the platform exits 0 whatever the job did. Creates report through the exit code — a
rejected bundle or an invalid config exits 1 with the reason on stderr. Downloads do too, but
check that the file you wanted actually appeared: a download of something the job has not
produced yet is the one place the CLI can exit 0 having written nothing useful.

## The entity model

The commands for each stage. What the entities are and how they chain: `../platform.md`
§ Entities and jobs. The `Override` column says whether a job's configuration can be varied at
submission — see § Overrides.

| Stage | Entity | Submit | Read | Override |
|---|---|---|---|---|
| trace-processing | PreparedTraces → SeedDataset | `distil traces upload --data <dir>`, then `distil seed-dataset create-from-traces <traces-id>` | `distil seed-dataset {status,logs,metrics,download,download-metadata,download-traces-predictions}` | yes |
| (job input) | SeedDataset | `distil seed-dataset create --data <dir>` | `distil seed-dataset {status,download,download-metadata}` | the files on disk |
| teacher-evaluation | TeacherEvaluation | `distil teacher-evaluation create-from-seed-dataset <seed-dataset-id>` | `distil teacher-evaluation {status,logs,metrics,download-metadata,download-predictions}` | yes |
| synthetic-data-generation | TrainingDataset | `distil training-dataset create-from-seed-dataset <seed-dataset-id>`, or `distil training-dataset create --data <dir>` from local files | `distil training-dataset {status,logs,metrics,sample,download,download-metadata}` | yes |
| test-set-expansion | TrainingDataset | as synthgen, from a **new** SeedDataset with train and test inverted | as synthgen | data change, so no |
| model-training | SLM | `distil slm create-from-training-dataset <training-dataset-id>` | `distil slm {status,logs,metrics,download,download-metadata,download-predictions}` | yes |
| model-deployment | Deployment | `distil deployment create-from-slm <slm-id>` | `distil deployment {status,endpoint,logs}`, `distil deployment delete` | no config of its own |

The CLI once called the job-input entity an Upload. `upload` and `uploads` remain aliases of
`seed-dataset`, and `create-from-upload` of `create-from-seed-dataset`, so old scripts keep
working; write the current spelling. Each group also answers to its plural (`slms`,
`deployments`, `teacher-evaluations`, …) and `list` answers to `ls`.

### Which commands speak JSON

`--output json` is registered on the read commands — `list`, `show`, `status`, `logs`,
`metrics`, `sample`, `endpoint`, plus `whoami` and `credits-balance` — and on the job creates
that name a parent id: `teacher-evaluation create-from-seed-dataset`,
`training-dataset create-from-seed-dataset`, `slm create-from-training-dataset` and
`deployment create-from-slm`.

It is **not** registered on the commands that read local files (`traces upload`,
`seed-dataset create`, `training-dataset create`, `slm create`), on
`seed-dataset create-from-traces`, or on any download command. Passing it there fails with
`No flag registered for --output` and creates nothing, so `dl_id` on those is a rule rather
than a preference.

Without `--output json` a read prints a panel for a human reader and suggests the next command
underneath. Parse the JSON form.

### Supplying files

The CLI does the presigned-URL exchange that the API backend does by hand. `--data <dir>` names
a directory and the CLI reads the files out of it by name:

| Command | Required in `--data <dir>` | Optional |
|---|---|---|
| `distil traces upload` | `traces.jsonl`, `config.yaml`, `job_description.json` | `test.jsonl` |
| `distil seed-dataset create` | `train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json` | `unstructured.jsonl` |
| `distil training-dataset create` | `train.jsonl`, `test.jsonl`, `config.yaml`, `job_description.json` | — (an `unstructured.jsonl` left in the directory is skipped with a warning) |
| `distil slm create` | `model.tar`, `config.yaml` | — |

`config.yml` is accepted for `config.yaml`. The per-file flags — `--traces`, `--train`,
`--test`, `--unstructured`, `--config`, `--job-description`, `--model` — replace one path each,
and combining them with `--data` is refused rather than merged. A file missing from the
directory is named before anything uploads.

The download commands write these same names, so **a downloaded directory feeds straight back
into the matching `create --data`**. That round trip is how a change to the *data* is made,
since no override can reach it.

`distil training-dataset create --data <dir>` builds a TrainingDataset from local files instead
of from a SeedDataset. It returns a dataset id and no SeedDataset id, and the SeedDataset id is
the address of teacher evaluation and of every later iteration, so it is not a substitute for
`create-from-seed-dataset` in the main path. Its one use in this skill is the training smoke,
§ The TrainingDataset.

`distil slm create --data <dir>` is not part of the stage path either. It registers an existing
`model.tar` and `config.yaml` as an SLM. It uploads a model; it does not train one.

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

`--output json` returns `{"balances": {…}}` under the same keys. A route the platform does not
meter reads `unlimited` in the table and `"inf"` in the JSON, and a route it never charges for
is absent from both — so a missing key is not a zero. Read an unfamiliar key defensively.

**The route keys are the platform's, not the CLI's**, and since the CLI took the platform's
name for the entity they now agree: `seed_datasets_*` is what `distil seed-dataset` spends.

Read the balance **before synthgen**, not before training. A synthgen run with no training
credit left leaves the generation spend with nothing to use it. A submission against an
exhausted route is refused.

## Submitting jobs

Every job is created by naming the id of the entity before it. Nothing uploads at this point —
the parent's files are already on the platform.

| Stage | Command group | Typical timeout |
|---|---|---|
| Trace processing | `seed-dataset` | 45 min |
| Teacher evaluation | `teacher-evaluation` | 30 min |
| Synthetic data generation | `training-dataset` | 90 min |
| Model training | `slm` | 90 min |
| Deployment | `deployment` | 40 min |

### Trace processing

Upload the traces, then process them into a SeedDataset.

```bash
traces_id=$(distil traces upload --data traces-input | dl_id)

seed_dataset_id=$(distil seed-dataset create-from-traces "$traces_id" | dl_id)
dl_poll seed-dataset "$seed_dataset_id" $((60 * 45))
```

`traces-input/` holds `traces.jsonl`, `config.yaml`, `job_description.json` and an optional
`test.jsonl`; supplying that test file replaces the generated test split and makes
`num_traces_as_testing_base` inert. Neither command takes `--output json`, so both read their id
with `dl_id`.

Iterating is an override on the same PreparedTraces, which re-uploads nothing:

```bash
distil traces download-metadata -d traces/iter-2 "$traces_id"   # free
# in traces/iter-2/config.yaml, under trace_processing:
#   relevance_filtering: true
retry_id=$(distil seed-dataset create-from-traces \
  --config traces/iter-2/config.yaml "$traces_id" | dl_id)
dl_poll seed-dataset "$retry_id" $((60 * 45))
```

A PreparedTraces owns the trace file it was uploaded with, so a different set of traces means
`distil traces upload` again. Config and job-description changes over the same traces are
overrides.

### The SeedDataset

A job-input directory (`../data-preparation/overview.md`), uploaded whole:

```bash
seed_dataset_id=$(distil seed-dataset create --data input | dl_id)
```

`seed-dataset create` validates the bundle before creating anything and prints the platform's
validation error when it fails. That is this backend's dryrun. Three failures look different
and all name the cause:

```
# a file missing from the directory — caught locally, nothing uploads
Required file not found: test.jsonl in input

# a task the platform does not have
Only the following tasks are supported: ['classification', 'question-answering-open-book', …]

# a row in the wrong shape, named down to the row index
1 validation error for JobInputParser
job_input.1.data.question-answering.train_dataset.0.messages
  Field required [type=missing, input_value={'wrong': 'shape'}, input_type=dict]
```

Each exits 1 and creates nothing. Correct the directory and run the command again.

The route is metered (`seed_datasets_post`) but only a successful create spends a credit: the
balance is checked first and the call recorded only after validation passes. So validating a
broken bundle repeatedly is free, and a refusal here means the balance was already zero before
the bundle was read.

A direct SeedDataset runs no job. It reports `JOB_SUCCESS` as soon as the command returns and
needs no poll; its `logs` are empty and its `metrics` are null. A trace-derived one runs a job
and needs a poll.

### The TrainingDataset (training smoke calibration)

Normally a TrainingDataset is *produced* by synthgen. It can also be created from local files,
which is how `../../stages/model-training.md` Step 3 builds the truncated dataset it calibrates
memory settings on:

```bash
distil training-dataset download -d training/smoke-1/input "$dataset_id"
# Truncate in place: keep the ~100 longest rows of train.jsonl and the ~10
# longest of test.jsonl, leaving config.yaml and job_description.json as
# downloaded. `create --data` then reads the same directory.
smoke_dataset_id=$(distil training-dataset create --data training/smoke-1/input | dl_id)
```

The download is metered on `training_datasets_download_get`, the create on
`training_datasets_post`. The create runs the same validation as `seed-dataset create`
(`../data-preparation/overview.md` § Validation rules) and prints the failure the same way.

### Overrides: how a job is parameterised

The four job creates — `seed-dataset create-from-traces`,
`teacher-evaluation create-from-seed-dataset`, `training-dataset create-from-seed-dataset` and
`slm create-from-training-dataset` — each accept `--config <file>` (`-c`) and
`--job-description <file>`. Both are optional and independent; omit one and the job inherits the
parent's.

Each **replaces** the parent's file whole. The CLI does not merge, and whatever the file omits
takes a *library default* rather than the parent's value (`../platform.md` § Overrides). So an
override is read-edit-resend, never hand-written:

```
distil <group> download-metadata -d <dir> <id>   →   edit one field   →   pass to --config
```

`download-metadata` writes the entity's `config.yaml` and `job_description.json` and costs no
credits. What it writes is the *fully expanded* config — every default materialised, keys
sorted, comments dropped. That is exactly why editing that file and sending it back preserves
everything you did not touch.

PreparedTraces is the exception: it echoes the config it was staged with, comments intact and
nothing expanded. A trace-processing override therefore edits your own file.

```bash
distil seed-dataset download-metadata -d synthgen/smoke-1 "$seed_dataset_id"
# edit synthgen/smoke-1/config.yaml
dataset_id=$(distil training-dataset create-from-seed-dataset --output json \
  --config synthgen/smoke-1/config.yaml "$seed_dataset_id" | jq -r .id)
```

Two failure shapes, both worth knowing before you spend anything:

- A config naming one field under `base` is refused, because `base.task` has no default:

  ```
  Invalid config: 1 validation error for DistilConfiguration
  base.task
    Field required [type=missing, input_value={'student_model_name': 'Qwen3-0.6B'}, input_type=dict]
  ```

- A config carrying a complete `base` but no `synthgen` or `tuning` section is **accepted**, and
  those sections revert silently. Submitted against a parent that set `generation_target: 512`,
  `output_is_json: true`, `per_device_train_batch_size: 8` and a list of `mutation_topics`, a
  base-only override ran with `generation_target: 10000`, `output_is_json: false`,
  `per_device_train_batch_size: 1` and no mutation topics. Nothing errored.

Because the read-back config is complete, checking what a submission actually ran with is a
diff rather than an audit:

```bash
distil training-dataset download-metadata -d check "$dataset_id"
diff synthgen/smoke-1/config.yaml check/config.yaml
```

An override reaches the config and the job description and nothing else. **A change to the data
is a new SeedDataset** — which is why test-set expansion inverts train and test into a fresh one
rather than overriding the existing one. `distil seed-dataset download -d <dir> <id>` writes a
directory `distil seed-dataset create --data <dir>` accepts unchanged, so making one is an edit
between two commands.

Since the parent never changes, an iteration is the same parent submitted again: `smoke-1`,
`smoke-2` and `full-1` all name one SeedDataset. Record each child's UUID in the iteration's
`run.md`.

### Teacher evaluation

```bash
te_id=$(distil teacher-evaluation create-from-seed-dataset --output json "$seed_dataset_id" \
  | jq -r .id)
dl_poll teacher-evaluation "$te_id" $((60 * 30))
```

A different teacher model is a config change; sharper judge instructions are a job-description
change. Both are overrides on the same SeedDataset, and neither needs a new one:

```bash
distil seed-dataset download-metadata -d te/iter-2 "$seed_dataset_id"
# te/iter-2/config.yaml           → base.teacher_model_name
# te/iter-2/job_description.json  → llm_as_a_judge_instructions
retry_id=$(distil teacher-evaluation create-from-seed-dataset --output json \
  --config te/iter-2/config.yaml \
  --job-description te/iter-2/job_description.json "$seed_dataset_id" | jq -r .id)
dl_poll teacher-evaluation "$retry_id" $((60 * 30))
```

### Synthetic data generation

```bash
dataset_id=$(distil training-dataset create-from-seed-dataset --output json \
  "$seed_dataset_id" | jq -r .id)
dl_poll training-dataset "$dataset_id" $((60 * 90))
```

A smoke run is a config override on the same SeedDataset:

```bash
distil seed-dataset download-metadata -d synthgen/smoke-1 "$seed_dataset_id"
# in synthgen/smoke-1/config.yaml, under synthgen:
#   generation_target: 64
#   generation_iteration_size: 16
smoke_dataset_id=$(distil training-dataset create-from-seed-dataset --output json \
  --config synthgen/smoke-1/config.yaml "$seed_dataset_id" | jq -r .id)
dl_poll training-dataset "$smoke_dataset_id" $((60 * 90))
```

The full run is the same command with the intended `generation_target` and
`generation_iteration_size`, or with no `--config` at all if the SeedDataset already carries
them.

### Model training

```bash
slm_id=$(distil slm create-from-training-dataset --output json "$dataset_id" | jq -r .id)
dl_poll slm "$slm_id" $((60 * 90))
```

Training reads its baseline config from the TrainingDataset, which inherited it from the
SeedDataset synthgen ran over. Setting sensible `tuning` values before synthgen means the common
case needs no override.

A sweep is N submissions against the same dataset, one per student. Read the dataset's config
once, then write one file per student, because the CLI sends each file whole:

```bash
distil training-dataset download-metadata -d sweep "$dataset_id"
# copy sweep/config.yaml once per student, changing base.student_model_name in each

for config in sweep/student-*.yaml; do
  distil slm create-from-training-dataset --output json --config "$config" "$dataset_id" \
    | jq -r .id
done
```

`per_device_train_batch_size`, `memory_optimized_training` and `use_qlora` vary the same way —
read, edit, resend — which is how OOM is handled without touching the data. Submissions run
concurrently, so start them all and poll them after.

## Monitor

Poll `distil <group> status --output json <id>` every 20 seconds; `dl_poll` does this. The
status values and how to run a poller: `../platform.md` § Job status.

```bash
dl_poll training-dataset "$dataset_id" $((60 * 90))
```

`status` answers `{"status": "JOB_RUNNING"}`. Deployments answer
`{"deployment_status": …, "endpoint_status": …}` and have no `status` field at all, so they need
`dl_poll deployment "$id" $((60 * 40)) deployment_status`.

`distil <group> logs --output json <id>` returns `{"logs": "…"}` — the whole job log as one
string, cause at the end. It fills while the job runs, so it is also how a long job is watched;
`dl_poll` prints its tail before returning non-zero.

`distil <group> list --output json` returns one object per entity, newest first, carrying `id`,
`created_at`, the parent's id under its own key (`seed_dataset_id`, `training_dataset_id`,
`slm_id`, …) and, where an entity can come from more than one kind of parent, a `source` naming
which. It is how a lost id is recovered and how a child is traced back to its parent. **It
carries no status** — that is one `status` call per id.

## Output layout

The CLI fetches nothing by object-storage path. Metrics arrive as JSON on stdout; a download
command writes files to disk. Which outputs each stage produces, and the conditions on them:
`../platform.md` § What each stage produces.

| Entity | Metrics | Data files | Config + job description | Predictions |
|---|---|---|---|---|
| PreparedTraces | — | `traces download` | `traces download-metadata` | — |
| SeedDataset | `seed-dataset metrics` (trace-derived only) | `seed-dataset download` | `seed-dataset download-metadata` | `seed-dataset download-traces-predictions` |
| TeacherEvaluation | `teacher-evaluation metrics` | — | `teacher-evaluation download-metadata` | `teacher-evaluation download-predictions` |
| TrainingDataset | `training-dataset metrics` (byte sizes) | `training-dataset download` (metered), `training-dataset sample` (free) | `training-dataset download-metadata` | — |
| SLM | `slm metrics` | `slm download` | `slm download-metadata` | `slm download-predictions` |
| Deployment | — | — | — | `deployment endpoint` |

The `Config + job description` column is the free read every override starts from. Each
`download-metadata` writes exactly `config.yaml` and `job_description.json`, on every entity,
and nothing else.

Every command that writes a directory takes `--destination <dir>` (`-d`) and otherwise names one
after the entity: `<id>-traces`, `<id>-data`, `<id>-slm`, `<id>-metadata`. The `*-predictions`
commands write a single file, so they take `--file-name` instead and default to
`<id>-<kind>-predictions.jsonl` in the current directory. Downloads overwrite what is already
there.

Metrics and data downloads answer only once that entity's own job reaches `JOB_SUCCESS`. Until
then a metrics field is `null` and the matching download reports the job's state — `SLM is
still training` — rather than writing an empty file. Check the file appeared rather than
trusting the exit code alone.

`download-metadata` is the partial exception: config and job description are settled at
submission, so a TeacherEvaluation or SLM answers within seconds of the create and an override
can be read back from a job still running. A TrainingDataset does not answer until its job
finishes, and a failed job never answers at all — so record what you submitted rather than
planning to read it back.

Two entries carry conditions:

- **`seed-dataset metrics` is empty for a directly created SeedDataset.** Both fields are
  `null`: the command reports the trace-processing job and a direct SeedDataset runs none. On a
  trace-derived one they hold the score of **the model that produced the traces** on your test
  set — the production model being distilled from, not the untuned student. The student cannot
  be scored before it is trained.
- **`slm metrics` gives two scores but only the tuned model's predictions.** The base-vs-tuned
  gap is reportable as numbers; a base-model failure case is not.

## Fetch metrics

Aggregated metrics arrive as the `*_performance` object: a flat metric-name-to-score dict for
every task except classification. The matching download command writes the per-example detail.

```bash
distil teacher-evaluation metrics --output json "$te_id" | jq .teacher_performance
# {"rouge": 1, "binary": 0.82, "llm-as-a-judge": 1, "llm-as-a-judge-reference-free": 1}

distil slm metrics --output json "$slm_id" \
  | jq '{base: .base_model_performance, tuned: .tuned_model_performance}'

distil teacher-evaluation download-predictions "$te_id"
```

A metric the run did not compute is `null` rather than absent. Filter those before averaging.

Three things about the predictions file:

- It is **JSONL**, one test example per line, carrying `prompt`, `completion`, `prediction` and
  that example's own score under each metric name. Read one row and work from what is there.
- `prompt` is the full prompt as a JSON-encoded message list, not the user text alone, and
  `completion` and `prediction` are JSON-encoded assistant messages. Parse them; do not compare
  them as raw strings.
- For classification the performance object is not flat: alongside `accuracy` it carries **one
  key per class label**, each holding `{precision, recall, f1-score, support}`. There is no
  `confusion_matrix` and no `classification_report`. Iterate by key rather than assuming numeric
  values — `accuracy` is a float and every other entry is a dict.

`metrics --output json` also carries the `*_download_url` the download command uses. It is
presigned and expires after an hour; take it directly only to hand the data to another program.

### Reading a TrainingDataset

`training-dataset sample` is free and answers `{"rows": [{"messages": […]}, …]}` — at most 128
train rows, drawn deterministically from the first 384, never test rows. `download` costs a
credit on `training_datasets_download_get`.

```bash
distil training-dataset sample --output json "$dataset_id" > sample.json
jq '.rows | length' sample.json
distil training-dataset metrics --output json "$dataset_id" | jq .train_data_size_bytes
```

`metrics` reports bytes, not rows. At smoke scale the dataset is smaller than the cap, so the
sample is the whole thing and its row count is exact. For a full run, divide the byte count by
the mean row size in the sample and treat the result as an estimate.

## Fetch model artifacts

```bash
distil slm download --destination model "$slm_id"
```

This writes `model.tar` and `config.yaml`. The tarball expands to the layout `../deployment.md`
§ Artifacts describes; the config is not inside it, and both files are needed to serve the model.
The tarball is gigabytes — about 1.2 GB for a Qwen3-0.6B run — and the command checks free disk
space before it starts writing.

### The inference client on its own

`slm download-metadata` writes the model's client alongside its config and job description, a
few kilobytes instead of the gigabytes of the tarball:

```bash
distil slm download-metadata --destination model "$slm_id"
# model/config.yaml, model/job_description.json, model/model_client.py
```

What the client is for and how to call it: `../deployment.md`.

## Reuse synthetic data for training-only runs

Retrain the same dataset under a new config. Nothing is downloaded, copied or reassembled:

```bash
distil training-dataset download-metadata -d retrain "$dataset_id"
# in retrain/config.yaml, under tuning:
#   num_train_epochs: 6
retrained_id=$(distil slm create-from-training-dataset --output json \
  --config retrain/config.yaml "$dataset_id" | jq -r .id)
dl_poll slm "$retrained_id" $((60 * 90))
```

This is the sweep command from § Submitting jobs with one field changed instead of the student.

## Deploy (hosted)

```bash
deployment_id=$(distil deployment create-from-slm --output json "$slm_id" | jq -r .id)
dl_poll deployment "$deployment_id" $((60 * 40)) deployment_status

eval "$(distil deployment endpoint --output json "$deployment_id" \
  | jq -r '@sh "url=\(.url) api_key=\(.api_key)"')"
```

`endpoint` carries the `url` and the `api_key`. Query the deployment through the model's own
client rather than a hand-built request: `../deployment.md` § Serving hosted has the call and
says why. The client takes an OpenAI-style base URL, which is the endpoint URL with its trailing
slash dropped and `/v1` appended — `"${url%/}/v1"` in shell:

```bash
distil slm download-metadata -d model "$slm_id"      # § The inference client on its own
uv run model/model_client.py --base-url "${url%/}/v1" --api-key "$api_key" \
  --conversation '[{"role": "user", "content": "…"}]'
```

```bash
distil deployment delete "$deployment_id"
```

The deployment serves the model with vLLM, and the job does not return until vLLM answers, so
`JOB_SUCCESS` means serving rather than merely scheduled. Before that, `endpoint` answers
`{"url": null, "api_key": null}` in JSON and `This deployment is not serving an endpoint.` for a
human — **exiting 0 either way**, so poll the status rather than probing the endpoint. The API
key protects the endpoint; the tunnel has no authentication of its own and the URL is open to
all.

**A deployment is a session, not a permanent endpoint.** It stops after six hours, or after one
hour with no traffic. It cannot be restarted, and a new deployment carries a new URL and a new
key.

CAUTION: delete the deployment when finished. **A running deployment bills until its idle
timeout.** After the delete `deployment_status` stays `JOB_SUCCESS`; `endpoint_status` going to
`stopped` is what confirms it is down.

To serve the model locally, download `model.tar` as above, then read `../deployment.md`
§ Serving locally.
