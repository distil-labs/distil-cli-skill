# Polling Long-Running Jobs

Distil jobs (upload processing, teacher evaluation, synthetic data generation, training) run asynchronously. This file is the canonical polling pattern. Every workflow and reference should link here rather than re-document the loop.

---

## Job Status Values

All async commands return one of these statuses:

| Status | Meaning |
|--------|---------|
| `JOB_NOT_STARTED` | Not started yet. |
| `JOB_PENDING` | Queued, waiting for a worker. |
| `JOB_RUNNING` | Currently executing. |
| `JOB_SUCCESS` | Completed successfully. |
| `JOB_FAILURE` | Errored out and stopped. |
| `JOB_STOPPED` | Manually stopped or interrupted. |

**Terminal values:** `JOB_SUCCESS`, `JOB_FAILURE`, `JOB_STOPPED`. Anything else means still in progress.

---

## Canonical Polling Loop

Copy this pattern verbatim. Change only the status command and the ID.

```bash
while true; do
    status=$(distil teacher-evaluation status <teacher-evaluation-id> --output json | jq -r '.status')
    echo "$(date +%H:%M:%S) status=$status"
    case "$status" in
        JOB_SUCCESS|JOB_FAILURE|JOB_STOPPED) break ;;
    esac
    sleep 60
done
echo "Final status: $status"
```

Every entity exposes `status`, and every one puts the job status at `.status`, so the loop body is identical across jobs:

| Polling target | Status command |
|----------------|----------------|
| Upload / trace processing | `distil upload status <upload-id> --output json` |
| Teacher evaluation | `distil teacher-evaluation status <teacher-evaluation-id> --output json` |
| Synthetic data generation | `distil training-dataset status <training-dataset-id> --output json` |
| Training | `distil slm status <slm-id> --output json` |

### Sleep interval

- **Minutes-scale jobs** (upload, trace processing, teacher evaluation, synthetic data generation): `sleep 60`.
- **Hours-scale jobs** (training): `sleep 600`.

### Deployments poll differently

`distil deployment status` is the one exception: it returns **two** fields and neither is called `status`.

```bash
while true; do
    endpoint=$(distil deployment status <deployment-id> --output json | jq -r '.endpoint_status // "none"')
    echo "$(date +%H:%M:%S) endpoint=$endpoint"
    if [ "$endpoint" = "running" ]; then break; fi
    sleep 30
done
```

`deployment_status` carries the usual `JOB_*` values and tells you whether the deploy finished. `endpoint_status` is `running`, `stopped`, or `null`, and tells you whether the endpoint can actually answer a request. Poll on `endpoint_status` when you are waiting to send traffic — a `JOB_SUCCESS` deploy whose endpoint is still `stopped` is not ready.

### Datasets with no job behind them

`distil training-dataset status` reports `JOB_SUCCESS` immediately for a dataset created by `distil training-dataset create`, because a direct upload is synchronous and has no job. The loop above terminates on the first iteration, which is correct — do not read it as a job that finished suspiciously fast.

---

## Why This Pattern

Two pitfalls caused real iteration loops in past sessions. This pattern avoids both.

1. **Do not grep human-readable output.** The printable status strings can change between CLI versions, so a `grep -q "COMPLETED\|FAILED"` loop picks the wrong status about half the time. Always extract `.status` via `--output json | jq -r '.status'`.

2. **Do not chain `sleep N && command`.** Claude Code blocks long leading sleeps that look like rate-limit workarounds, and the chained form breaks silently. A `while ... sleep N; done` loop is fine because the sleep is inside the loop body, not leading the next command.

### Wrong patterns — avoid

```bash
# Wrong: greps arbitrary words
distil slm status <slm-id> | grep -q "COMPLETED\|FAILED"

# Wrong: leading sleep is blocked / unreliable
sleep 300 && distil slm status <slm-id>
```

---

## Working Directory Gotcha

Bash heredocs and some multi-line commands can reset the working directory between invocations. If you `cd` into a project dir then run a Python heredoc, the next command may run from a different cwd. Two safe patterns:

```bash
# Pattern A: cd inline on every command
cd /path/to/project && python script.py

# Pattern B: absolute paths only
python /path/to/project/script.py
```
