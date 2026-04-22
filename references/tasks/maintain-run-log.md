# Maintain Run Log

Keep a single human-readable log of every step taken while building a model. The log lets a user (or Claude, in a later session) reconstruct what was tried, what changed, and why — without re-reading a pile of iteration-suffixed analysis reports.

## File

- **Path:** project root (the same directory as `job_description.json`, `config.yaml`, `train.csv`, `iteration-N/`).
- **Name:** `model-building-log-<name>.md` where `<name>` is a descriptive slug for the project (human-readable, not the model ID) — typically the same slug the user will pass to `distil model create`.
- **Format:** markdown. Reverse-chronological — newest entry at the top. One `##` heading per entry.

If the file already exists at session start, read it to reconstruct context before proposing next steps. Do not rewrite earlier entries — only prepend new ones.

## When to Append

Append a new entry at each of these points. The workflows and `improving-a-model.md` call this file at the right moments; you do not need to remember the list, just write an entry whenever a workflow step says to.

- Model creation (`distil model create`)
- Each data upload (`upload-data`, `upload-traces`, `reprocess-traces`)
- Analyze Uploads verdict (traces workflow, optional step)
- Test-set approval (traces workflow)
- Teacher evaluation analysis (verdict + iteration)
- Training analysis (verdict + iteration)
- Retune (new student or new tuning parameters)
- Deployment
- Start and end of each iteration (see `workflows/improving-a-model.md` — start entry names the lever being tested; end entry records the verdict delta)

## Entry Format

```markdown
## YYYY-MM-DD HH:MM — <step> — iter <N>

**What changed:** <one sentence on the lever pulled, or "n/a" for initial steps>
**Artifact:** [iteration-N/README.md](iteration-N/README.md), [iteration-N/teacher-eval-analysis.md](iteration-N/teacher-eval-analysis.md)
**Verdict:** <PROCEED | ITERATE | RETHINK | DEPLOY | RETUNE | ESCALATE | INFORMATIONAL | n/a>
**Headline:** <primary metric and delta vs. prior iteration, or "n/a">
**Why:** <one sentence on the reasoning for the lever or decision>
```

Omit fields that don't apply (e.g., `Artifact` on the model-creation entry). Keep each entry to ~5 lines. Longer prose belongs in the iteration's `README.md` or the analysis report.

## Worked Example

```markdown
# Model Building Log — support-ticket-classifier

## 2026-04-21 14:32 — training-analysis — iter 2

**What changed:** retuned with `num_train_epochs: 6` (was 4).
**Artifact:** [iteration-2/training-analysis.md](iteration-2/training-analysis.md)
**Verdict:** DEPLOY
**Headline:** LLM-as-a-Judge 0.87 (+0.06 vs. iter 1, within 0.03 of teacher).
**Why:** iter 1 showed underfit pattern; more epochs closed the gap.

## 2026-04-20 12:04 — test-set-approval — iter 1

**What changed:** user approved generated test set (240 examples, balanced across 5 classes).
**Verdict:** n/a
**Headline:** original model LLM-as-a-Judge 0.71 on this test set.

## 2026-04-20 11:40 — model-create

**What changed:** `distil model create support-ticket-classifier`.
**Verdict:** n/a
```
