# Writing Nerine tests from a specification

Hand an LLM a specification and this reference together, and ask it to write
the Nerine scripts that check it. [`REFERENCE.md`](REFERENCE.md) is written
for that: a complete, self-contained description of the Nerine scripting
language, precise enough that the model does not need anything else to write
a correct script.

## Using it

Give the model both documents in one prompt: the specification, and
`REFERENCE.md`. Ask it to write one or more Nerine scripts covering the
specification. `REFERENCE.md` ends with a small worked example showing the
translation the model needs to make — a sentence of a specification becomes
one or more `compare` lines on the request it describes.

Run the result the way you would run any Nerine script:

```
nerine "tests/*.txt"
```

## How this differs from `spec-driven/`

[`spec-driven/`](../spec-driven) hands an agent a specification and a set of
already-written Nerine scripts, and has the agent build an application the
scripts hold it to. This is close to the opposite: there is no application
and no scripts yet, and the model's job is to produce the scripts themselves,
directly from the specification.
