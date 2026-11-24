# Phase 2: Plan the Build

| | |
|---|---|
| **Input** | the scope and the draft job description (Phase 1); the user's test set, if they provided one |
| **Output** | the agreement in the run log header; the agreed job description; the root Dataset that carries it |
| **Gate** | the user agrees the plan and the job description |
| **Uses** | `references/model-catalog.md`, `references/evaluation-metrics.md`, `references/platform.md` § Billing, `references/variable-system-prompts.md`, `operations/teacher-evaluation.md`, `operations/create-dataset.md` |

## Goal

Agree how the model will be built, so that Phase 4 can run without asking. Then agree the job
description with the user, after an optional sanity check with the teacher. Present each item with
your recommendation and its reason, and get the user's answer.

## The agreement

| Item | Recommendation |
|---|---|
| **The teacher** | the catalog's default large teacher (`references/model-catalog.md` § Defaults), or the one the user names. Optionally a stronger **test-row teacher** for test rows only, because test labels decide every verdict |
| **The student, or a range of students** | the catalog's default student, with thinking on or off (`references/reasoning-models.md`). With a range, Phase 4 trains each student in iteration 1 and continues with the best |
| **The budget and the time** | dollars for everything from here on, an iteration cap, and when the user needs the model. Price the plan with dry runs on the first Dataset and the draft job description (`references/platform.md` § Billing). Training cost and time grow with the train set, so estimate the later iterations too, and give the wall-clock time of one iteration (generation plus training) next to its price (`operations/train-model.md` says how to estimate training time) |
| **The metric** | when the task has a deterministic measure the user trusts (a label, spans, a tool call checked against a schema), use it, compute it locally from the prediction files with the user's scorer, and keep the judge as a second signal. Otherwise use `llm-as-a-judge-reference-free` with a strong judge model and the rubric from Phase 1 (`references/evaluation-metrics.md` § Primary metric per task). The metric, the judge model and the rubric are fixed from here on, because a change makes every earlier score incomparable |
| **The production baseline** | the production model, if there is one. It is scored in Phase 3 and compared in Phase 5, and a student below it is not deployed |
| **The review thresholds and run sizes** | suggested starting values below; agree the actual ones |

| Threshold or size | Suggested |
|---|---|
| clear label errors in a blind review | ≤ 2% |
| undecidable rows in a blind review | ≤ 2% |
| sample reviewed after a full test-set run | 200 rows |
| smoke run | 100–250 rows |
| first train set | 5,000 generated rows (plus relabelled traces, if any) |
| rows added per iteration (top-up) | 500–1,000, aimed at one failure mode |

The first train set has to cover the whole scope, so it is large. A top-up only has to teach the
one failure mode the last error analysis found, and a few hundred well-aimed rows do that. Small
top-ups keep each iteration short, so more iterations fit in the time available.

**Parallelism.** `base.llm_num_parallel_requests` makes jobs faster without changing what they
return. In the skill's test runs, 64 generated 1.4–3× more rows per minute than 16, depending on the
task.

## Sanity check of the job description (optional)

When the user has provided a test set (labelled rows they trust), run one teacher evaluation of the
draft job description on it (`operations/teacher-evaluation.md`). It is a sanity check that the
teacher reads the description the way the user means it, not a tuning step. Phase 3 builds the
real test set next, so these rows may not be final.

- Show the user the score and a few of the teacher's errors.
- If they are fine with it, move on.
- If an error shows that the description says something the user did not mean, or leaves out
  something they consider basic, the user corrects the description and you run the check once
  more.

Do not rewrite the description row by row to raise this score. A description edited until the
teacher matches these particular rows fits those rows, not the task, and every later step inherits
that.

Skip the check when there is no user test set, or when each request carries its own system prompt
(`base.variable_system_prompt`). In the second case the task description is not part of the prompt
in teacher evaluation (`references/variable-system-prompts.md`), so a teacher score says nothing
about it.

**A fixed production system prompt.** When production sends a system prompt that the user cannot
change, the model must be trained with that prompt, so `task_description` is that prompt word for
word. Rules the user adds or clarifies go into the generation instructions and the LLM validator
instead (Phases 3 and 4).

## Exit

The user agrees the plan and the job description. Once agreed, the job description is not changed,
and neither are the judge rubric in it and the judge model. Create the **root Dataset**: the same
data with the agreed job description and the agreed config (`operations/create-dataset.md`; for
Entry A, from the same traces object again). Every later Dataset descends from it.

Record the agreement in the run log header, the job description's version history (each version's
change and where its rules came from), the sanity-check score if one was run, the agreed file and
the root Dataset id. Mark
Phase 2 done. From now on, a change to the job description is the user's decision, because it
changes what every later score measures.
