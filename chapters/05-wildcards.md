# Chapter 5 — Wildcards: One Rule for Many Samples

> *The idea that lets a single rule serve a whole study.*

The pipeline at the end of Chapter 4 had a fatal flaw for real work: every
filename in it was hard-wired to `sampleA`. A real RNA-seq study has dozens or
hundreds of samples, and Chapter 1 warned us where copy-pasting a rule per sample
leads — sixty near-identical rules, sixty chances to mistype a filename and count
the wrong reads. This chapter introduces the feature that dissolves the problem:
**wildcards**. By the end you will write the `count` rule *once* and have it apply
to every sample in the study.

## The problem, concretely

Suppose your study has three samples (a real one might have a hundred; three is
enough to see the pattern). Their raw reads live in:

```
data/sampleA.fastq
data/sampleB.fastq
data/sampleC.fastq
```

> A **sample** is one sequenced biological specimen — "liver tissue from mouse
> #7," "patient 12 at baseline." Each sample gets its own FASTQ file and is
> processed independently through the early steps of the pipeline; the samples
> only come together at a summary step. Keeping samples straight — never letting
> sample B's reads end up labeled as sample A's — is a correctness concern, not
> just bookkeeping, and is exactly the kind of error wildcards are designed to
> prevent.

With the Chapter 4 approach you would need a `count` rule for `sampleA`, another
for `sampleB`, another for `sampleC` — identical but for the name. That is the
anti-pattern. We want one rule that says, in effect, "to count *any* sample, do
this."

## A wildcard is a pattern variable

A **wildcard** is a named placeholder in an input or output path that can stand
for any text. You write it in braces. Here is the Chapter 4 `count` rule with
`sampleA` replaced by a wildcard named `sample`:

```python
rule count:
    input:
        "data/{sample}.fastq"
    output:
        "results/{sample}.count.txt"
    shell:
        "echo '{wildcards.sample}: '$(( $(wc -l < {input}) / 4 ))' reads' > {output}"
```

The `{sample}` in the paths is no longer a fixed name — it is a variable. The
single rule now describes how to count *any* sample whose FASTQ follows the
pattern `data/<something>.fastq`, producing `results/<that same something>.count.txt`.
One rule, any number of samples.

> Inside the recipe, the wildcard's *value* is available as `{wildcards.sample}`
> — note the `wildcards.` prefix. This is how a rule can put the sample's name
> into its command or output, here labeling each count with the sample it came
> from. Do not confuse `{input}`/`{output}` (the rule's files) with
> `{wildcards.sample}` (the value a wildcard took on); they are different things
> that happen to share the brace notation.

## The crucial idea: wildcards are inferred *backwards*

Here is the point that trips up nearly everyone at first, so read it slowly. A
wildcard does *not* have a value until someone asks for a specific output file.
The value is determined by **matching the requested filename against the output
pattern** — and this happens backwards, from output to input, exactly like the
DAG reasoning in Chapter 2.

Walk through what happens when something asks Snakemake to produce
`results/sampleB.count.txt`:

1. Snakemake looks for a rule whose `output` pattern matches that filename. The
   pattern `results/{sample}.count.txt` matches, and the match forces
   `sample = "sampleB"`.
2. With `sample` now bound to `"sampleB"`, Snakemake fills in the *same* wildcard
   everywhere else in the rule. The `input` becomes `data/sampleB.fastq`. The
   `{wildcards.sample}` in the recipe becomes `sampleB`.
3. Snakemake checks whether `data/sampleB.fastq` exists (or can itself be built by
   another rule), and proceeds.

The wildcard flows **output → wildcard value → input**. This is why you never
write a loop over samples inside a rule: you write the rule for a single,
anonymous sample, and Snakemake instantiates it once per requested output, each
instance with its own wildcard value. Each such instance is a separate **job** —
a rule with its wildcards filled in — and a separate node in the DAG.

## Asking for many outputs at once: `expand()`

Wildcards explain how *one* output gets built. But our `all` rule needs to *list*
all the final outputs we want, across every sample — and we would rather not type
them out by hand. Snakemake provides a helper called **`expand()`** for exactly
this: it takes a pattern and one or more lists of values, and returns the list of
all filenames formed by filling the pattern in.

First, list the samples once, as an ordinary Python list (this is the kind of
plain Python Chapter 3 promised you would need):

```python
SAMPLES = ["sampleA", "sampleB", "sampleC"]
```

Then use `expand()` to turn that list into a list of target files:

```python
expand("results/{sample}.count.txt", sample=SAMPLES)
# → ["results/sampleA.count.txt",
#    "results/sampleB.count.txt",
#    "results/sampleC.count.txt"]
```

> Note the division of labor between `{sample}` here and the wildcard of the same
> name in the rule. In a *rule*, `{sample}` is a wildcard whose value Snakemake
> *infers* by matching a requested filename. In `expand()`, `{sample}` is a slot
> that *you* fill from a list to *generate* those requested filenames. `expand()`
> produces the requests; wildcard matching answers them. They meet in the middle.

## Putting it together

Here is the Chapter 4 pipeline, generalized to any number of samples. The `count`
rule is written once with a wildcard; the `report` rule gathers every sample's
count into one file; and `expand()` drives the targets:

```python
SAMPLES = ["sampleA", "sampleB", "sampleC"]

rule all:
    input:
        "results/report.txt"

rule count:
    input:
        "data/{sample}.fastq"
    output:
        "results/{sample}.count.txt"
    shell:
        "echo '{wildcards.sample}: '$(( $(wc -l < {input}) / 4 ))' reads' > {output}"

rule report:
    input:
        expand("results/{sample}.count.txt", sample=SAMPLES)
    output:
        "results/report.txt"
    shell:
        "cat {input} > {output}"
```

Create `data/sampleB.fastq` and `data/sampleC.fastq` alongside the `sampleA` file
from Chapter 4 (any small FASTQ contents will do), then dry-run and run it:

```bash
$ snakemake -n          # see the plan: three 'count' jobs and one 'report' job
$ snakemake --cores 1
$ cat results/report.txt
sampleA: 2 reads
sampleB: 2 reads
sampleC: 2 reads
```

Look at what the DAG became. The single `count` rule produced **three** jobs — one
per sample, each with `sample` bound to a different value — and all three feed the
one `report` job. This is precisely the branch-and-merge shape from the Chapter 2
diagram, and you wrote it without ever naming a sample inside a rule. Add a fourth
sample to the FASTQ folder and to `SAMPLES`, and a fourth `count` job appears
automatically; no rule changes. *This* is what "one rule for many samples" buys
you.

> A preview of Chapter 9: because the three `count` jobs are independent nodes in
> the DAG, running with `--cores 3` would execute all three **at the same time**.
> Wildcards do not just save you typing — they expose the parallelism in your data
> to Snakemake's scheduler for free.

## When a wildcard matches too much

Wildcards are greedy: by default a wildcard will match *as much text as it can*,
including dots and slashes. Usually this is harmless, but it can create
**ambiguity** when a filename could be split in more than one way.

Imagine you later name files with both a sample and a sequencing **replicate**,
like `results/sampleA.rep1.count.txt`, using a two-wildcard pattern:

```python
output: "results/{sample}.{replicate}.count.txt"
```

How should `sampleA.rep1` be split? Snakemake could bind `sample="sampleA"` and
`replicate="rep1"` — what you meant — or `sample="sampleA.rep1"` and an empty
`replicate`, or other splits. Faced with this, Snakemake will either guess wrong
or stop with an error rather than risk it.

The fix is a **wildcard constraint**: you restrict, with a small pattern, what
text a wildcard is allowed to match. Constraints are written as **regular
expressions** (a standard notation for describing text patterns; `[A-Za-z0-9]+`
means "one or more letters or digits").

```python
wildcard_constraints:
    sample="[A-Za-z0-9]+",     # letters and digits only — no dots
    replicate="rep[0-9]+",     # the word 'rep' followed by digits

rule count:
    input:
        "data/{sample}.{replicate}.fastq"
    output:
        "results/{sample}.{replicate}.count.txt"
    # ...
```

Now `sample` cannot contain a dot, so `sampleA.rep1` can only split the way you
intended. You will not need constraints for simple pipelines, but when filenames
get structured — and in real projects they do — knowing that this lever exists
will save you a baffling afternoon. The rule of thumb: **if Snakemake complains
about ambiguity, or a wildcard is grabbing more of a filename than you meant,
reach for `wildcard_constraints`.**

## Where we are going

Wildcards are the conceptual hinge of the whole system. With them, a rule stops
being a one-off command and becomes a *template* that Snakemake stamps out across
your data, building exactly the DAG your samples imply. We have been hand-listing
samples in a `SAMPLES = [...]` line, which is fine for three but clumsy for a real
study and disconnected from the sample metadata you actually have. Chapter 7 fixes
that by reading samples from a proper **sample sheet**. But first, Chapter 6
steps back to the DAG itself — now that wildcards make it branch and merge — and
shows how to *visualize* it and reason about exactly when each job re-runs.
