# Operation: Evaluate a Deployment

Deploys a trained SLM behind a new inference endpoint and runs the rows of a split through it with
the inference settings production will use: the system prompt, temperature, token limit and
thinking setting. It writes one predictions file per split and its scores, then checks from the
endpoint's records that every answer came from the model and not from the fallback. This is also
the only way to score an existing SLM on rows it was not evaluated on: no platform command does
that (`../references/inference-endpoints.md` § Lifetime).

| | |
|---|---|
| **Called by** | the deploy phase (the best model with production settings), and the test-set phase when an extended test set is scored without retraining |
| **Price** | one deployment, priced by `--dry-run` (Step 2), charged at creation and not refunded when the deployment fails. The endpoint, the key and the records are free (`../references/platform.md` § Billing) |

## Settings that control the output

| Setting | In the request | What it changes |
|---|---|---|
| The system prompt | the first message | The prompt sent with every row: the production system prompt, or each row's own system message |
| Temperature | `temperature` | Sampling. `0` gives repeatable answers; production may use another value |
| Token limit | `max_tokens` | The longest answer; a cut-off answer scores as wrong |
| Thinking | `extra_body={"chat_template_kwargs": {"enable_thinking": …}}` | A model trained without thinking must run with it off (`../references/reasoning-models.md` § Deployment) |
| JSON mode | `response_format={"type": "json_object"}` | For every task with JSON answers (`synthgen.output_is_json`), as `model_client.py` and the platform's evaluation do; without it the model sometimes writes malformed JSON |
| Tools | `tools`, `tool_choice` | For `chat-completion` with tools, as in `model_client.py` |
| Concurrency | requests in flight at once | Speed. Keep it at 1–2: one deployment returned corrupted answers under 16 concurrent requests (accuracy 0.905 against 0.960 at 1) |
| The fallback | `--fallback-model` at endpoint creation | An OpenRouter model, required by every endpoint. It answers when the deployment fails, which is why Step 5 checks the records |

The model was trained and scored on the platform with the `task_description` as its system prompt
(or each row's own, with `base.variable_system_prompt` on) and the request settings in its
`model_client.py`: sampling, thinking, and JSON mode for JSON tasks
(`../references/deployment.md` § Why model_client.py instead of raw requests). Read that file
before this run: it is a standard OpenAI client and a reference for how to call the model, not a
fixed tool. Take its request settings (thinking, JSON mode, tools) into the run, and change only
what the evaluation needs to differ (for example temperature 0 or a token limit). Production
settings that differ from the client's are what this run measures.

## Working directory

```
evaluate-deployment/<name>/
├── system_prompt.txt     # the production system prompt, exactly as the application sends it
├── rows/                 # the split files run through the endpoint
├── preds/                # one predictions file per split
├── scores/               # scores per split and group, records.jsonl from the endpoint
├── run_split.py          # the runner you wrote from model_client.py
└── run.md                # SLM id, deployment id, endpoint name, key file name, settings, price, scores
```

`distil api-keys create` writes the key file into the current directory, so run it from the
directory that should hold it, outside any version control. Never print the file.

## Step 1: Prepare the input

- **The SLM id**, from the run log. A model trained with a `tuning.lora_r` outside 8, 16, 32, 64,
  128, 256, 320, 512 cannot be deployed.
- **The rows.** Download the Dataset whose split is evaluated and copy the split into `rows/`
  (`../references/execution/cli.md` § Reading a Dataset). Keep the row-group files the run log
  lists, so scores can be read per group.
- **The settings**, from the calling phase: the system prompt into `system_prompt.txt`, the
  temperature, the token limit, and the thinking setting. When the rows' own system messages are
  to be sent instead, check that every row has one.
- **The fallback**: the production model's OpenRouter slug in `owner/model` form. Without a
  production model, pick a cheap open model, and tell the user that the rows it answers reach its
  provider.
- **The key**: an existing key file, or a new key in Step 3. An account holds at most 5 keys.

Prepare all of it before the deployment exists: a deployment stops after 6 hours, or after 1 hour
without calls, and cannot be restarted (`../references/inference-endpoints.md` § Lifetime). Run
every split in one session.

## Step 2: Price it

```bash
distil deployment create-from-slm --dry-run <slm-id>
```

Record the price the dry run returns in the run log's spending and in `run.md`. A failed deployment
is not refunded, and every new deployment is charged again.

## Step 3: Submit

Deploy, wait until it is ready, create the endpoint and link a key
(`../references/execution/cli.md` § Deploy an SLM, § Create an endpoint, § Keys):

```bash
distil deployment create-from-slm --output json <slm-id> | jq -r .id
bash poll.sh deployment <deployment-id>        # ready at deployment_status JOB_SUCCESS
distil inference-endpoint create-from-deployment --output json --name <name> \
  --fallback-model "<owner/model>" <deployment-id> | jq -r .unique_endpoint_name
distil api-keys create <key-name> > /dev/null  # only when no key exists; writes ./<key-name>.json
chmod 600 <key-name>.json
distil inference-endpoint link-api-key <unique-endpoint-name> <key-name>
```

`distil deployment status --output json <deployment-id>` returns
`{"deployment_status": …, "endpoint_status": …}`; it reads `JOB_RUNNING` while the deployment
starts. `create-from-deployment` refuses with `Deployment is not ready` before `JOB_SUCCESS`. The
key file holds the secret in its `secret` field: never print it, and name the file in `run.md`.
Record the deployment id and the unique endpoint name in the run log's lineage.

A newly linked key can be rejected for about a minute. Wait a minute, then run Step 4's runner on
one row as a check before the full run.

**Check who answers every call.** A deployment that stopped or failed does not make the calls
fail: the fallback answers them, and the run looks normal. Each response says who served it, the
student or the fallback. On the check row, confirm that the response and the endpoint's record
(Step 5) agree. Then check every response in the run: when the fallback answered, send the
request again. When retries keep coming from the fallback, the deployment has stopped: check
`distil deployment status`, stop the run, and find the cause.

## Step 4: Wait and pull

Write a runner from the model's `model_client.py`: a standard OpenAI client that calls
`https://inference.distillabs.ai/v1` with `model` set to the unique endpoint name and the key's
secret as the API key. Keep the client's request settings (thinking, JSON mode, tools), and set
the ones this evaluation needs (the system prompt, temperature, the token limit) from the table
above. For each row of a split, send the conversation without its final answer and save the
input, the reference and the raw answer, one JSON object per row. Retry a failed call a few times,
and count the empty answers. Run one or two requests at a time.

Then download the endpoint's records for the run. Pass the start of the run and `--all`: the
default window is the last day and the default cap 1000 records
(`../references/execution/cli.md` § Download the traces):

```bash
distil inference-endpoint download-traces --from-start-time <run start, ISO 8601> --all \
  --file-name evaluate-deployment/<name>/scores/records.jsonl <unique-endpoint-name>
```

When all splits are done, delete the deployment unless it is about to serve traffic:
`distil deployment delete <deployment-id>`. The endpoint stays and answers from the fallback.

## Step 5: Read the output

**Every answer came from the model.** Count the records by source:

```bash
jq -r '.metadata.source' evaluate-deployment/<name>/scores/records.jsonl | sort | uniq -c
```

Every record must read `primary`, and the count should match the rows sent. A `fallback` record
is the production model's answer, so the scores are not the model's: the deployment failed or
stopped (`distil deployment status`). Failed calls are recorded too; `metadata.status` gives the
HTTP status.

**Scores by group.** Score each split with the project's metric: overall, and for each row group
the run log lists (`protocols/error-analysis.md` Step 1 says how). Write them to `scores/`.

**Empty answers.** Read the rows with an empty answer. Check the token limit first, especially
with thinking on, where the reasoning counts against the same limit.

**Parity with the platform.** Compare the endpoint's score on the test set with the platform's
score for the same SLM on the same metric (`distil slm metrics`, or your scorer on the platform's
predictions file). They must agree within the noise recorded in the run log. Also count the rows
where the endpoint's answer differs from the platform's prediction: at temperature 0 they should
match on nearly every row. A larger gap means the prompt or the settings differ from the
platform's: compare the system prompt sent with the system message in the `prompt` field of the
SLM's predictions file, and the settings with the model's `model_client.py`. When production
settings differ on purpose, the gap is the effect of those settings; to separate the two, run the
test split once more with the trained system prompt and the client's settings.

**Judge-scored tasks.** A judge score cannot be computed locally. Check parity row by row
instead: when the endpoint's answers match the platform's predictions on nearly every row, the
platform's judge scores stand for the deployed model. Report the share of rows that differ, and
read them: a difference there is a prompt or settings difference, not noise.

Record the scores, the source counts and the parity check in `run.md` and the run log.
