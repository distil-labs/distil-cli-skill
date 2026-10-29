# Workflow: Autoresearch

The agent improves a model on its own: it runs `../stages/model-iteration.md` again and again,
one change per iteration, until the model reaches a target agreed with the user or the budget
is spent.

## Workflow Map

Print this map to the user when starting the workflow, before the first step:

```
 0 Agree ─── the primary metric, the budget, the iteration cap and the judge
   ▼
 1 Starting model ─── build-a-model.md Steps 1-8 through training, unless one exists;
   │                  then the target and the noise go into the ledger
   ▼
 2 Iterate ─── model-iteration.md Steps 1-5, without asking between iterations
   ▼  stop? (Step 2's conditions) ──► 3
   └──────────► otherwise back to 2
 3 Report ─── the best model, what each iteration changed, why the loop stopped
```

## Step 0: Agree the Limits

Before anything runs, agree with the user and write into the ledger's header
(`../stages/model-iteration.md` § The Ledger; create it now):

- **The primary metric**: the default for the task (`../references/evaluation-metrics.md`
  § Primary metric per task) unless the user names another.
- **The budget**: an amount in dollars for everything from Step 1 on, the starting model
  included, within the balance (`../references/platform.md` § Billing), and a maximum number of
  iterations. Every iteration trains at least one model, so propose the iteration count from the
  price of a training run.
- **The judge**: set once in the config (`build-a-model.md` Step 4) and kept for every run.
- **The target**: the teacher's score, unless the user sets another
  (`../references/evaluation-metrics.md` § Verdicts). It is filled in once Step 1 has measured
  the teacher.

Within this agreement the agent iterates without asking (`../SKILL.md` § Stage Protocol). It
stops and asks before:

- changing `task_description`;
- changing the test set (`../stages/model-iteration.md` § Changing the Test Set);
- changing the judge model or the judge instructions;
- changing the teacher that writes the train set;
- a submission that would take the spending over the budget;
- moving production traffic to a model. Deploying a model only to try it is allowed within the
  budget.

## Step 1: The Starting Model

A trained model on a locked test set, with its teacher evaluations in `test-set.md`, is the
ledger's row 1. When there is none, run `build-a-model.md` Steps 1-8 with their own gates: the
test set and the teacher are the user's decisions, and every iteration is read against them. At
Step 6, submit the chosen teacher twice; the spread between the two runs is the noise. At Step 8,
the first model becomes row 1 instead of a decision.

Fill in the header: the teacher's score, the base score, the noise, the target and the spending
so far.

## Step 2: Iterate

Run `../stages/model-iteration.md` Steps 1-5, again and again. Within this workflow:

- Step 1's verdict does not end the loop; the stop conditions below do.
- Smoke a run whose settings are new to the project; take the fast path for a re-run of a
  proven setup.
- Add each submission's price to the spending in the header before submitting. Before a train
  set iteration, also count the price of the last training run, because training is priced only
  once its data exists.

After each ledger row, stop and go to Step 3 when:

- **the target is reached**: the best kept row's primary metric, lowered by the noise, clears
  it;
- **the budget is spent**: the next iteration would take the spending over the budget, or the
  iteration cap is reached;
- **it plateaus**: `vs best` stays within the noise two iterations in a row;
- **the diagnosis needs the user**: the next change is one of the decisions in Step 0.

## Step 3: Report

At every stop, and whenever the user asks, report from the ledger: the target, the best
iteration and its scores, what each iteration changed and what it did, the money spent, and why
the loop stopped. When it stopped for a decision, state the failure mode and the decision
needed. A deploy candidate goes to `build-a-model.md` Step 9 with the user.
