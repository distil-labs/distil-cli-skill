# Maintain Run Log

Keep a single human-readable log of every step taken while building a model. The log lets a user (or Claude, in a later session) reconstruct what was tried, what changed, and why — without re-reading a pile of iteration-suffixed analysis reports.

The log records **decisions and reasoning** — which lever was pulled, what the verdict was, and why. It is not an ID ledger: the platform already links entities to their parents, so `references/cli-reference.md` (`### Tracing the Chain`) recovers any lineage you need from a single ID. Noting the ID a step produced is useful shorthand, not a system of record.

## File

- **Path:** project root (the same directory as `job_description.json`, `config.yaml`, `train.jsonl`, `iteration-N/`).
- **Name:** `model-building-log-<name>.md` where `<name>` is a descriptive slug for the project. It is local bookkeeping only — no command takes a project name.
- **Format:** markdown. Reverse-chronological — newest entry at the top. One `##` heading per entry.

If the file already exists at session start, read it to reconstruct context before proposing next steps. Do not rewrite earlier entries — only prepend new ones.

## When to Append

Append a new entry at each of these points. The workflows and `improving-a-model.md` call this file at the right moments; you do not need to remember the list, just write an entry whenever a workflow step says to.

- Each data upload (`upload create`, `traces upload`, `upload create-from-traces`)
- Analyze Uploads verdict (traces workflow, optional step)
- Test-set approval (traces workflow)
- Teacher evaluation analysis (verdict + iteration)
- Synthetic data generation (`training-dataset create-from-upload`)
- Training analysis (verdict + iteration)
- Deployment (`deployment create-from-slm`)
- Start and end of each iteration (see `workflows/improving-a-model.md` — start entry names the lever being tested; end entry records the verdict delta)

## Entry Format

```markdown
## YYYY-MM-DD HH:MM — <step> — iter <N>

**What changed:** <one sentence on the lever pulled, or "n/a" for initial steps>
**ID:** <the entity ID this step produced — handy shorthand, not a system of record>
**Artifact:** [iteration-N/README.md](iteration-N/README.md), [iteration-N/teacher-eval-analysis.md](iteration-N/teacher-eval-analysis.md)
**Verdict:** <PROCEED | ITERATE | RETHINK | DEPLOY | ESCALATE | INFORMATIONAL | n/a>
**Headline:** <primary metric and delta vs. prior iteration, or "n/a">
**Why:** <one sentence on the reasoning for the lever or decision>
```

Omit fields that don't apply (e.g., `Artifact` on an upload entry). Keep each entry to ~6 lines. Longer prose belongs in the iteration's `README.md` or the analysis report.

The `Why` line is the one that earns the file. Scores and IDs can be re-read off the platform later; the reasoning behind a lever choice cannot.

## Worked Example

```markdown
# Model Building Log — support-ticket-classifier

## 2026-04-21 14:32 — training-analysis — iter 2

**What changed:** regenerated the training dataset with `num_train_epochs: 6` (was 4) and retrained. Reused iter 1's upload and teacher evaluation — the lever was a tuning parameter, not the data.
**ID:** slm 4c88d017-2f3a-4e91-b06d-8ca7f1e45d90
**Artifact:** [iteration-2/training-analysis.md](iteration-2/training-analysis.md)
**Verdict:** DEPLOY
**Headline:** LLM-as-a-Judge 0.87 (+0.06 vs. iter 1, within 0.03 of teacher).
**Why:** iter 1 showed underfit pattern; more epochs closed the gap.

## 2026-04-20 12:04 — test-set-approval — iter 1

**What changed:** user approved generated test set (240 examples, balanced across 5 classes).
**Verdict:** n/a
**Headline:** original model LLM-as-a-Judge 0.71 on this test set.

## 2026-04-20 11:40 — upload-create — iter 1

**What changed:** initial upload of 40 labeled tickets across 5 classes.
**ID:** upload 8a21c4de-6b5f-41c2-9d8e-3f0a72bc14e6
**Verdict:** n/a
```
