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
        about billing disputes: refunds, double charges, invoice errors
        about account cancellation: downgrades and win-back attempts
        about technical support: setup problems and integration errors
```

| Field | Required | Notes |
|---|---|---|
| `name` | yes | The dimension this mutator varies. Keep it unique across mutators: it labels the sampled value and seeds the mutator's own random stream |
| `values` | yes | At least one value, each a key with an optional description. An empty list is a config error |
| `method` | no | `uniform` (the default) is the only implemented option. Anything else is a config error, not a silent fallback |

Unknown fields are rejected, both on a mutator and on a value.

### Three ways to write `values`

```yaml
# Keys only
values: [simple, medium, complex]

# Key to description
values:
  simple: Straightforward, minimal reasoning required.
  complex: Multiple factors, nuance, or ambiguity involved.

# Explicit entries, the same thing spelled out
values:
  - key: simple
    description: Straightforward, minimal reasoning required.
  - key: complex
```

## Prompt composition

Active mutators render one line each under a shared prelude:

```
Important! Make sure to follow those guidelines:
Generated examples should be simple
Generated examples should be about billing disputes: refunds, double charges, invoice errors
```

Every line reads `Generated examples should be <key>`, with `: <description>` appended when
the value has one. Phrase keys so they finish that sentence, which is why the topics above
read `about billing disputes` rather than a bare `billing`, and keep the detail in the
description.

Each mutator draws from its own random stream, derived from `base.random_seed` and the
mutator name. A config reproduces its own mutation sequence, and two mutators with the same
number of values do not move in lockstep. Renaming a mutator changes its draws.

## How many, and in what proportions

- One mutator per independent dimension. Each call samples once from every mutator, so two
  mutators of three values each cover a 3x3 grid of scenarios without enumerating nine
  combinations.
- Use 1-2 mutators of 3-10 values each: more dilutes the signal per value, fewer limits
  diversity.
- Sampling is uniform over the listed values, so composition sets the proportions. Repeat a
  value to weight it: three entries asking for English and one for French gives roughly a
  75/25 split.
- A single-value mutator is a useful special case. It applies the same directive to EVERY
  call, which turns the mutator into a constant extra generation instruction (for example
  output length or document realism) without touching the job description.

## Recipes for the former built-in mutators

`complexity`, `length` and `specificity` used to be built in and `complexity` used to be on
by default. They are ordinary config now. Copy the one you want:

```yaml
synthgen:
  mutators:
    - name: complexity
      values:
        trivial: Obvious answer, no reasoning needed.
        simple: Straightforward, minimal reasoning required.
        medium: Some reasoning, a few factors to consider.
        complex: Multiple factors, nuance, or ambiguity involved.
        highly complex: Expert-level, many interacting factors, edge cases.
```

```yaml
synthgen:
  mutators:
    - name: length
      values:
        short and concise: 1-2 sentences, essentials only.
        medium length: 3-5 sentences, covers main points.
        detailed: Multiple paragraphs, includes context and nuance.
```

```yaml
synthgen:
  mutators:
    - name: specificity
      values:
        generic and vague: avoid specific references and concrete terms.
        somewhat specific: Use descriptive but general references and terms.
        specific: Use realistic terms and references.
        very specific: Use precise, domain-heavy terms and references.
```

Stick to one of these three at a time. Their directives conflict when stacked.

## Deprecated parameters

| Parameter | Status |
|---|---|
| `basic_mutators_to_use` | Accepted, no effect. The built-in mutators are gone; use the recipes above |
| `mutation_topics` | Accepted and translated into mutators |

A `mutation_topics` flat list becomes one mutator named `topics_1`. Nested lists become
`topics_1`, `topics_2`, one per list. Setting `mutators` and `mutation_topics` together is a
config error, so migrate a config in one move rather than half way.

The rendering changed with the translation: topics used to read
`Generated examples should focus on: billing disputes` and now read
`Generated examples should be billing disputes`. Re-read old topic phrasings when you migrate
them, and add descriptions where a bare topic name was carrying detail.

## When to touch this

- First run: set mutators upfront if the task already names patterns, domains, or length
  characteristics to cover. Otherwise start with none and add them after reading the smoke.
- Diversity gaps while iterating: add a dimension, or add values to an existing one.
- Wrong length or difficulty profile: add the `length` or `complexity` recipe.
- Near-duplicates of seed data: that is `validation_similarity_threshold`
  (`configuration.md`), not a mutator.
