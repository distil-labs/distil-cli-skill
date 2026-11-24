# Phase 0: Start

| | |
|---|---|
| **Input** | what the user has: an LLM in production, files, or both |
| **Output** | the project directory and run log; the first Dataset id; whether traces, labelled rows and train rows exist |
| **Gate** | none |
| **Uses** | `operations/inference-endpoint.md`, `operations/create-dataset.md`, `protocols/run-log.md` |

## Step 1: Set up

Install the CLI and sign in (`references/execution/README.md` § Set up the CLI), choose the
backend, and create the project directory and `log.md` (`protocols/run-log.md`). Copy the user's
files into `raw-inputs/` unchanged.

## Step 2: Choose the entry point

Ask what the user has. Both entries end in a Dataset; the later phases do not depend on which.

| The user has | Entry |
|---|---|
| An LLM in production, called by an application through an OpenAI-compatible chat API that can be pointed at another base URL | A |
| An endpoint that already holds traffic | A, from Step A3 |
| Files: labelled rows, train rows, unlabelled inputs, logs, or any mix | B |

The production model does not have to be one OpenRouter serves. The endpoint forwards each call to
a fallback model, which is named by an OpenRouter slug: the production model itself when OpenRouter
serves it, otherwise the closest model it does serve (for example for a private fine-tune). Tell the
user that the fallback, not their own model, answers through the endpoint until a trained student is
in place.

A user with files and a production LLM can do both: upload the files now (B) and put an endpoint in
place for the next cycle.

## Entry A: inference endpoint

The production model's traffic becomes **traces**: real inputs that can be relabelled into test
and train rows.

1. **A1.** Create a collecting endpoint with the production model, or its closest OpenRouter
   equivalent, as the fallback (`operations/inference-endpoint.md`).
2. **A2.** The user moves the application onto it. Wait until there are enough traces. The test
   set, the first train set and the generation context all draw on them, and each trace is used
   once (`references/platform.md` § The expand operations): plan for a few hundred for the test
   set, a few hundred for the first train rows, and more as generation context. Agree with the
   user when to come back, and stop until then.
3. **A3.** Create a traces object from the endpoint's records, and a Dataset from it
   (`operations/create-dataset.md`), with a draft config and job description: the task type
   (`references/task-types.md`), the user's description of the task, and the defaults. Phase 1
   and Phase 2 settle the real values.

The traces hold the production model's answers, not verified labels. They become test or train
rows only through relabelling (Phase 3, Phase 4).

## Entry B: dataset upload

1. **B1.** Sort the user's files by what they are:
   - rows with answers the user vouches for → candidate test rows (`test.jsonl`), and train rows
     (`train.jsonl`) if there are many;
   - inputs without trusted answers (logs, exported production calls) → `traces.jsonl`, to be
     relabelled later;
   - the task's specification, policies or examples of correct behaviour → material for the job
     description (Phase 1).
2. **B2.** Convert them into the Dataset format (`references/data-preparation/overview.md` and the
   file for the task type). Remove fields that leak the answer, such as a rationale.
3. **B3.** Upload with a draft config and job description (`operations/create-dataset.md`).

Whichever entry, check the task type against the answer format before the first upload
(`references/task-types.md`): an answer that calls tools is `chat-completion`, never
`question-answering`. Tools written into the row text of a question-answering task are just text to
the platform, so nothing warns about it.

## Step 3: Record

Write to the run log: the entry, the first Dataset id in the lineage, and the counts of labelled
rows, train rows and distinct traces. Phase 1 needs to know what data exists, Phase 2 whether labelled rows exist for the teacher check, Phase 3
whether there is a candidate test set and traces, and Phase 4 whether there are traces for the
first iteration.
