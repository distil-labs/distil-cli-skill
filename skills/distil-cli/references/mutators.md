# Mutators

Mutators control the structure and diversity of synthetic data: each generation call samples
one value per configured mutator and appends the directives to the generation prompt.
Configured in `synthgen.mutators` in `config.yaml`.

There are no built-in mutators and no default ones. A config that does not list any generates
every example from the same prompt, so the only variation comes from the teacher temperature
and the sampled in-context exemplars.

## The shape

`synthgen.mutators` is a list, one entry per dimension to vary:

```yaml
synthgen:
  mutators:
    - name: complexity
      values: [simple, medium, complex]
    - name: topic
      values:
        - "about billing disputes: refunds, double charges, invoice errors"
        - "about account cancellation: downgrades and win-back attempts"
        - "about technical support: setup problems and integration errors"
```

| Field | Required | Notes |
|---|---|---|
| `name` | yes | The dimension this mutator varies. Keep it unique across mutators: it labels the sampled value and seeds the mutator's own random stream |
| `values` | yes | At least one value, each a plain string. An empty list is a config error |
| `method` | no | `fixed` (the default) samples from the target distribution (constant sampling weights). `adaptive` classifies what survived validation and steers sampling towards values that fall behind (§ Adaptive mutators) |
| `target_distribution` | no | `uniform` (the default), `match_seed`, or one weight per value, e.g. `[5, 3, 2]`. Weights are normalised, so proportions and relative counts both work. `match_seed` copies the proportions the teacher finds in the seed data (§ Matching the seed distribution) |
| `description` | no | What the dimension means, e.g. `how many reasoning steps the answer needs`. Read only by the classifier: matters with `adaptive` and `match_seed`, ignored otherwise |
| `beta` | no | ADVANCED, `adaptive` only. How far the sampler may drift from the target, 0 to 1. Default `0.9`. Leave it unless a value is rejected constantly (§ Adaptive mutators) |

Unknown fields on a mutator are rejected. So is `method: uniform`, the old name of the default:
configs written before adaptive mutators that set it explicitly must drop the field or change
it to `fixed`. An explicit weight list needs exactly one weight per value.

A value is one string, nothing more. The `description` field describes the whole dimension, not
one value: write the detail of a value into the same string, conventionally after a colon, and
quote the whole thing so YAML reads it as a string rather than a mapping.
`- "simple: minimal reasoning required"` is a value; `- simple: minimal reasoning required` is a
parse error waiting to happen.

## Prompt composition

Active mutators render one line each under a shared prelude:

```
Important! Make sure to follow those guidelines:
Generated examples should be simple
Generated examples should be about billing disputes: refunds, double charges, invoice errors
```

Every line is `Generated examples should be ` followed by the value verbatim, so phrase values
to finish that sentence. That is why the topics above read `about billing disputes` rather
than a bare `billing`.

Each mutator draws from its own random stream, derived from `base.random_seed` and the
mutator name. A config reproduces its own mutation sequence, and two mutators with the same
number of values do not move in lockstep. Renaming a mutator changes its draws.

## How many, and in what proportions

- One mutator per independent dimension. Each call samples once from every mutator, so two
  mutators of three values each cover a 3x3 grid of scenarios without enumerating nine
  combinations.
- Use 1-2 mutators of 3-10 values each: more dilutes the signal per value, fewer limits
  diversity.
- Sampling follows `target_distribution`, uniform unless set. To skew it, give one weight per
  value: `values: [English, French]` with `target_distribution: [3, 1]` asks for English in
  roughly three calls out of four. Repeating a value works too, but weights are easier to read
  and to change.
- A single-value mutator is a useful special case. It applies the same directive to EVERY
  call, which turns the mutator into a constant extra generation instruction (for example
  output length or document realism) without touching the job description.

## Adaptive mutators

A fixed mutator asks for each value in the target proportions and never checks what came back.
The teacher can ignore instructions, and validation drops examples unevenly, so the dataset can
end up short on exactly the values the mutator was added for. An adaptive mutator closes the
loop.

Choose this method only if the dataset is inbalanced after synthetic data generation.

```yaml
synthgen:
  mutator_update_frequency: 5
  mutators:
    - name: topic
      method: adaptive
      description: which part of the product the customer is asking about
      target_distribution: [0.5, 0.3, 0.2]
      values:
        - "about billing disputes: refunds, double charges, invoice errors"
        - "about account cancellation: downgrades and win-back attempts"
        - "about technical support: setup problems and integration errors"
```

Mechanics:

- Every `synthgen.mutator_update_frequency` batches (a batch is one `generation_iteration_size`
  round of generation plus validation), the teacher labels each surviving example with the
  value it actually landed on, or "other" if it doesn't match any value.
- The running counts are compared against the target for the whole run. Sampling shifts towards the values furthest
  behind.
- Batches between two classifier runs are not labelled; their counts are extrapolated from the
  last labelled batch. Each labelled example is one teacher call, so a higher frequency is
  cheaper and coarser. The default of 5 labels one batch in five.
- `beta` (ADVANCED): the weights are `(1 - beta) * target + beta * correction`. With the default
  0.9 every value keeps at least a tenth of its target share, so a value the teacher never
  manages to produce cannot take over the run. Lower it when one value keeps getting rejected (increasing the generation time)
  and keeping the mix matters more than chasing it. Do not touch it otherwise.

Gotchas:

- A smoke run may never adapt. `generation_target: 64` with `generation_iteration_size: 16` is
  four batches, one short of the first classifier run at the default frequency. Set
  `mutator_update_frequency: 1` in the smoke config to see it work, and restore the default for
  the full run.
- Always write the `description` for an adaptive mutator. The classifier decides from the
  values and the description alone, and a bare `complexity` is ambiguous without one.
- The classifier sees validated examples only, so a value that validation rejects reads as
  under-generated and gets asked for more. If a value is rejected because it is genuinly hard to generate correctly (often rejected in validation e.g. due to wrong format, length,...), fix the mutator value; adapting only burns
  teacher calls on it.

## Matching the seed distribution

`target_distribution: match_seed` has the teacher classify the seed training data along the
dimension once, at the start of the run, and uses the proportions it finds as the target. It
works with either `method` and costs one teacher call per seed example.

- Values absent from the seed data get zero weight and are never asked for (warning in the
  logs). If the user wants a value the seed data lacks, give explicit weights instead.
- With no seed data, or none that lands on any of the values, the mutator falls back to
  uniform, with a warning.
- Unlike `match_generated_distribution_to_seed` (classification and tool calling only, matches
  the class or tool distribution), `match_seed` works per mutator on any task type and on any
  dimension the values describe.

## Recipes for the former built-in mutators

`complexity`, `length` and `specificity` used to be built in and `complexity` used to be on
by default. They are ordinary config now. Copy the one you want:

```yaml
synthgen:
  mutators:
    - name: complexity
      values:
        - "trivial: obvious answer, no reasoning needed"
        - "simple: straightforward, minimal reasoning required"
        - "medium: some reasoning, a few factors to consider"
        - "complex: multiple factors, nuance, or ambiguity involved"
        - "highly complex: expert-level, many interacting factors, edge cases"
```

```yaml
synthgen:
  mutators:
    - name: length
      values:
        - "short and concise: 1-2 sentences, essentials only"
        - "medium length: 3-5 sentences, covers main points"
        - "detailed: multiple paragraphs, includes context and nuance"
```

```yaml
synthgen:
  mutators:
    - name: specificity
      values:
        - "generic and vague: avoid specific references and concrete terms"
        - "somewhat specific: use descriptive but general references and terms"
        - "specific: use realistic terms and references"
        - "very specific: use precise, domain-heavy terms and references"
```

Stick to one of these three at a time. Their directives conflict when stacked.

## Deprecated parameters

| Parameter | Status |
|---|---|
| `basic_mutators_to_use` | Accepted, no effect. The built-in mutators are gone; use the recipes above |
| `mutation_topics` | Accepted and translated into mutators |

A `mutation_topics` flat list becomes one mutator named `topics_1`. Nested lists become
`topics_1`, `topics_2`, one per list. The translation clears `mutation_topics` itself, so a
config read back after a run shows the mutators and an empty topic list. Setting `mutators`
and `mutation_topics` together is a config error, so migrate a config in one move rather than
half way.

Two things to re-read when migrating old topics. The rendering changed: topics used to read
`Generated examples should focus on: billing disputes` and now read
`Generated examples should be billing disputes`, so old phrasings may need rewording. And a
value is a string only: the `{key: description}` mapping form and `{key: ..., description: ...}`
entries are rejected, not flattened, so fold any such detail into one string.

## When to touch this

- First run: set mutators upfront if the task already names patterns, domains, or length
  characteristics to cover. Otherwise start with none and add them after reading the smoke.
- Diversity gaps while iterating: add a dimension, or add values to an existing one.
- A value that keeps coming up short in the output despite being listed: switch that mutator
  to `method: adaptive` with a `description`, or give it a heavier weight in
  `target_distribution`.
- Generated proportions should mirror the seed data along some dimension:
  `target_distribution: match_seed`.
- Wrong length or difficulty profile: add the `length` or `complexity` recipe.
- Near-duplicates of seed data: that is `validation_similarity_threshold`
  (`configuration.md`), not a mutator.
