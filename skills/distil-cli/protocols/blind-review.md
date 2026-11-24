# Protocol: Blind Review

Verifies rows that the platform added (relabelled or generated) before they are used. The
teacher's labels are only as good as the teacher, and a scan of the rows by eye misses most label
errors, so independent reviewers label the rows again without seeing the row's answer, and every
disagreement is examined.

It needs a coding agent that can start **subagents**: independent agents with their own context
that run in parallel. You (the main agent) never label the rows yourself, because you have seen
the generated answers and the settings, and that biases the review.

Called by `make-rows.md` after every smoke run, and on a sample after a full test-set run.

## Inputs

- the run directory (it holds `output/` with the downloaded Dataset);
- the parent's split file, to tell the new rows apart from the old ones;
- the sample size (all new rows for a smoke; the agreed sample for a full test-set run);
- the thresholds from the run log header: maximum clear label errors and maximum undecidable rows.

## Step 1: Prepare the batches

Write a short script for this task's row format:

1. **Find the new rows.** Take the run's split from the downloaded Dataset and drop every row
   that is also in the parent's split. Match on the row's input, the last user message with
   whitespace normalised; for multi-turn rows, the whole conversation before the answer. Number
   the new rows.
2. **Sample.** For a full test-set run, draw the agreed sample size at random, with a fixed seed
   so it can be reproduced. For a smoke, take every new row.
3. **Write two files.** `<run-dir>/new_rows.json` holds each row's number, input and answer, and
   stays with you. `<run-dir>/review/blind_batch_<k>.json` holds the number and the input only,
   with no answers, split into about 3 batches of equal size (one per reviewer).

## Step 2: Run the reviewers

Start one subagent per batch, all at once, with the prompt below. Fill in the angle brackets for
the task. Step 2 of the prompt matters most: list the facts in this task that decide the answer
and must be computed explicitly, such as time windows, counts, thresholds or priority rules.
Reviewers that compute these facts catch the errors the teacher makes; reviewers that skim agree
with the teacher.

```text
You are a careful annotator. Answer inputs for <task>, following its task description exactly.

Read the full task description: the `task_description` field of <path to the agreed
job_description.json>. Read all of it, including every rule and policy.

Then answer every row in <run-dir>/review/blind_batch_<k>.json (a JSON list of {row, text}).

For each row:
1. List everything the input asks to be acted on or answered (not background or venting), and
   apply the task description's rules for choosing between them.
2. Work out every fact that decides the answer explicitly, with numbers: <the task's deciding
   facts, e.g. hours since delivery against the 48-hour window; business days against the refund
   window; the priority tier of each issue>.
3. Give exactly one answer, in the format the task description requires.
Also rate realism: "ok"; "unrealistic" (reads like a constructed test case, has contradictory
facts, translates itself, or states the conclusion instead of the facts); or "undecidable" (a
fact needed to decide the answer is missing or contradictory).

Write <run-dir>/review/labels_batch_<k>.json: a JSON list with one object per row:
{"row": <int>, "label": "<your answer>", "confidence": "high"|"medium"|"low",
 "realism": "ok"|"unrealistic"|"undecidable",
 "note": "<one or two sentences: what decided the answer, with the arithmetic>"}.
Every row must appear. Do not open any other file. Reply with the number of rows written and
how many are low confidence.
```

For free-text or tool-call answers, step 3 of the prompt becomes "write the answer you would
give"; Step 3 below says how to compare such answers.

If a reviewer fails (an error, a refusal, a missing file), start it again once. If it fails
again, review the remaining batches and record the gap. Unreviewed rows are unknown, not failed:
decide on the rows that were reviewed, and report the gap without waiting for the user. When the
gap is large enough to change the decision, or covers one kind of row entirely, review those rows
another way (smaller batches, another reviewer model) before deciding.

## Step 3: Compare

Join the reviewers' labels with the rows' answers on the row number, and decide for each row
whether they agree. For a label, that is the same label after whitespace normalisation. For
structured answers (spans, fields, tool calls), it is the task's own measure with a threshold you
state, for example span F1 of at least 0.9. For free text, read each pair and decide with the
judge rubric whether both answers are acceptable.

Write `<run-dir>/review/comparison.txt` with:

- the agreement rate;
- the counts of realism and confidence ratings;
- every row to adjudicate: each disagreement, and each row a reviewer did not rate "ok", with the
  reviewer's note and the full input.

## Step 4: Adjudicate

Read every listed row against the task description and sort it:

| Category | Meaning | Counts against |
|---|---|---|
| clear error | the row's answer is wrong under the task description | label errors |
| debatable | two answers are defensible; the task description does not settle it | neither (record it) |
| undecidable | a fact needed to decide is missing or contradictory | undecidable |
| reviewer mistake | the row is right; the reviewer misread it | neither |
| unrealistic but correct | the answer is right, the input reads constructed | neither (record it; a pattern is worth fixing) |

## Step 5: Decide

The review **passes** when clear errors and undecidable rows are both at or below the thresholds
in the run log header, as a share of the reviewed rows.

Whether it passes or not, find what caused each pattern of errors: a mutator value that asks for
something contradictory, an instruction the teacher misreads, a missing check in the LLM
validator, or a teacher lapse on a rule that is already clear. Two or more rows with the same
cause are a pattern. Fix the cause in the settings first, so the next run does not repeat it; when
a setting cannot fix it, filter or correct the affected rows locally (`SKILL.md` § Rules). A
teacher lapse with no pattern is noise.

## Step 6: Record

Write `<run-dir>/review/summary.md`: the counts (reviewed, agreed, clear errors, debatable,
undecidable, unrealistic), each adjudicated row in one line, the causes found, the decision
(pass or fail) and the fixes for the next smoke. Add the counts and the decision to the run's
`run.md` and to the run log.
