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

Copy this pattern verbatim. Change only the subcommand and model ID.

```bash
while true; do
    status=$(distil model teacher-evaluation <model-id> --output json | jq -r '.status')
    echo "$(date +%H:%M:%S) status=$status"
    case "$status" in
        JOB_SUCCESS|JOB_FAILURE|JOB_STOPPED) break ;;
    esac
    sleep 60
done
echo "Final status: $status"
```

Swap the status command depending on which job is being polled:

| Polling target | Status command |
|----------------|----------------|
| Upload / trace processing (by upload ID) | `distil upload status <upload-id> --output json` |
| Teacher evaluation | `distil model teacher-evaluation <model-id> --output json` |
| Teacher evaluation (by teacher evaluation ID) | `distil teacher-evaluation status <teacher-evaluation-id> --output json` |
| Synthetic data generation (by training dataset ID) | `distil training-dataset status <training-dataset-id> --output json` |
| Training | `distil model training <model-id> --output json` |
| Training (by SLM ID) | `distil slm status <slm-id> --output json` |

### Sleep interval

- **Minutes-scale jobs** (upload, trace processing, teacher evaluation, synthetic data generation): `sleep 60`.
- **Hours-scale jobs** (training, whether polled by model ID or SLM ID): `sleep 600`.

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
distil model training <model-id> | grep -q "COMPLETED\|FAILED"

# Wrong: leading sleep is blocked / unreliable
sleep 300 && distil model training <model-id>
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
