# Phase 5: Deploy and Evaluate

| | |
|---|---|
| **Input** | the best model (Phase 4); the test set; the production inference settings |
| **Output** | the best model's scores with production settings, against the other setups; a serving endpoint or a local deployment, if the user decides to serve |
| **Gate** | the user decides whether to serve |
| **Uses** | `operations/evaluate-deployment.md`, `operations/inference-endpoint.md`, `operations/local-deployment.md` |

The platform scores a model under its own settings. Production calls it with the application's
system prompt, sampling and token limit, so the score that matters is measured the same way.

## Step 1: Evaluate with production settings

Start from the model's `model_client.py`: it is a standard OpenAI client and the reference for
how to call the model (system prompt, thinking, JSON mode for JSON tasks, tools). Then ask the
user how the application will call it: the system prompt it sends, temperature, the maximum
number of new tokens. Keep the client's request settings unless the user's setup needs otherwise;
adapting the client is fine. Deploy the best model and run the
test set through it with those settings (`operations/evaluate-deployment.md`). Run any other
evaluation data the user has the same way.

Check two things before reading the scores:

- the endpoint's score on the test set matches the platform's score within the noise; a larger
  gap means a difference in the prompt or settings;
- every call was answered by the model, not by the endpoint's fallback (the endpoint records say
  which).

## Step 2: Compare with the other setups

Put the scores in one table, overall and per row group: the best model, the base student, the
teacher, and the production model, with the earlier iterations for reference. Report what the
gains came from (the slices that improved, from the ledger) and what is left (the largest failure
modes of the last error analysis). Use absolute scores for every setup.

## Step 3: Serve

◆ The user decides whether to serve the model.

- **Serving endpoint** (`operations/inference-endpoint.md`): the student as primary, the
  production model (or the fallback chosen for the evaluation) as fallback. Every request must
  carry the system prompt the model was trained with. The evaluation's deployment has usually
  timed out by now, which is normal: serving starts a new deployment, at its usual price.
- **The user's GPU** (`operations/local-deployment.md`).

## The next cycle

A serving endpoint's records are new traces. To improve the model on them, create a Dataset with
the new traces, the current `test.jsonl` and the best Dataset's `train.jsonl` (upload, so the
train rows and the test set carry over), and continue in Phase 4: the first iteration relabels
the new traces. If production shows failures the test set does not have, extending the test set
is the user's decision (Phase 3, § Changing the test set later).
