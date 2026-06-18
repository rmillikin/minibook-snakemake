# Chapter 6 — Building the Graph: Dependencies, Targets, and Inspection

> *Now that the graph branches and merges, learn to see it and to predict when it
> re-runs.*

Chapter 2 introduced the DAG as an idea; Chapter 5 made it real by letting one
rule spawn many jobs. With wildcards in hand, the graph for even a small study
already branches across samples and merges at a summary — too much to hold
reliably in your head. This chapter is about *inspecting* that graph: how rules
chain into it, how to draw a picture of it, and how to predict exactly which jobs
will re-run when something changes. These are the day-to-day skills for working
with a pipeline you did not write from scratch — which, as a new student, will be
most of them.

## A slightly longer chain

To have something worth inspecting, let us grow the running example by one step.
Real FASTQ files almost always arrive **compressed** (gzipped, with a `.gz`
suffix) to save space — a single uncompressed RNA-seq file can be many gigabytes.
So a realistic pipeline begins by decompressing. Assume the raw inputs are now:

```
data/sampleA.fastq.gz
data/sampleB.fastq.gz
data/sampleC.fastq.gz
```

We add a `decompress` rule ahead of `count`. The full pipeline becomes a
three-stage chain per sample, merging at the report:

```python
SAMPLES = ["sampleA", "sampleB", "sampleC"]

rule all:
    input:
        "results/report.txt"

rule decompress:
    input:
        "data/{sample}.fastq.gz"
    output:
        "results/{sample}.fastq"
    shell:
        "gunzip -c {input} > {output}"

rule count:
    input:
        "results/{sample}.fastq"
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

## How rules chain into a graph

Look closely at where `decompress` and `count` meet. The `output` of `decompress`
is the pattern `results/{sample}.fastq`. The `input` of `count` is the *same*
pattern, `results/{sample}.fastq`. That coincidence is the entire chaining
mechanism: **Snakemake connects two rules when one rule's output pattern can
produce a file matching another rule's input.**

Trace it backwards, as Snakemake does, for `sampleB`:

- `report` needs `results/sampleB.count.txt` (among others).
- That filename matches `count`'s output pattern, binding `sample="sampleB"`, so
  `count` needs `results/sampleB.fastq`.
- *That* filename matches `decompress`'s output pattern, again binding
  `sample="sampleB"`, so `decompress` needs `data/sampleB.fastq.gz`.
- That raw file exists. The chain bottoms out.

Notice the wildcard value `sampleB` **propagates along the whole chain**: the same
sample threads from raw input to final count without you ever wiring the rules
together by name. You defined three independent, local rules; the graph assembled
itself from the filename patterns. This is worth dwelling on because it is the
opposite of how the Chapter 1 shell script worked, where the order was written out
explicitly and by hand.

## Targets revisited: the graph fills itself in

The only place the pipeline states a *goal* is the `all` rule, whose input is
`results/report.txt`. Everything else — three decompress jobs, three count jobs,
one report job, in the right order — is *derived* from that single target by
backward-chaining. You can also ask for a different target directly on the command
line, which is invaluable when developing or debugging:

```bash
$ snakemake --cores 1 results/sampleB.count.txt
```

This asks for *just* one sample's count. Snakemake builds only the sub-graph
needed to produce it — `decompress` then `count` for `sampleB` — and ignores the
rest. The lesson: **the target you request determines how much of the graph comes
to life.** The `all` rule is just the convenient "everything" target; you are free
to aim at any file the pipeline knows how to make.

## Drawing the graph

For anything bigger than a toy, you will want to *see* the graph. Snakemake can
emit a description of it in the **Graphviz** `dot` language — a small standard
language for describing diagrams — which the `dot` program renders to an image.
There are two views, and the difference between them matters.

**The rule graph** (`--rulegraph`) shows one node per *rule*: the pipeline's
structure, independent of how many samples there are.

```bash
$ snakemake --rulegraph | dot -Tpng > rulegraph.png
```

For our pipeline it is a simple chain of three boxes:

```
   decompress  →  count  →  report
```

**The DAG** (`--dag`) shows one node per *job* — every rule instance with its
wildcards filled in. This is the concrete graph Snakemake will actually execute,
and for three samples it expands to seven jobs:

```bash
$ snakemake --dag | dot -Tpng > dag.png
```

```
 decompress(A)   decompress(B)   decompress(C)
      │               │               │
   count(A)        count(B)        count(C)
      │               │               │
      └───────────────┼───────────────┘
                      │
                   report
```

Hold the two pictures side by side and the relationship is clear: the **rule
graph is the template** (three rules), and the **DAG is the template stamped out
across your data** (seven jobs). When you want to understand *what a pipeline
does*, read the rule graph. When you want to understand *what a particular run
will execute* — including how much parallelism is available — read the DAG.

> A practical habit: `snakemake --rulegraph | dot -Tpng > rulegraph.png` is one of
> the first things to run when you inherit an unfamiliar pipeline. In thirty
> seconds you have a map of every step and how data flows between them — far
> faster than reading the rules top to bottom.

## When does a job re-run?

You saw the basics in Chapter 4: if an output is missing, or older than its
input, Snakemake rebuilds it. But "older than its input" is only one of several
**rerun triggers** — the conditions under which Snakemake decides a job's result
is stale and must be recomputed. Modern Snakemake tracks several, and understanding
them prevents two opposite frustrations: results that are quietly out of date, and
needless hours-long rebuilds.

The triggers are:

- **Input file timestamps (`mtime`)** — an input is newer than the output. The
  classic Make behavior from Chapter 4.
- **Code** — the rule's recipe (its `shell` command or script) changed. If you fix
  a bug in a command, Snakemake knows the old outputs were made by the old code
  and rebuilds them.
- **Parameters** — a `params` value (Chapter 7) the rule uses changed.
- **Input set** — the *list* of input files changed (e.g. you added a sample),
  even if the existing files did not.
- **Software environment** — the rule's declared software environment (Chapter 8)
  changed, e.g. you bumped a tool's version.

That second one is a genuine advance over plain Make, and it matters for
reproducibility: with Make, editing a command does *not* by itself trigger a
rebuild, so your outputs can silently disagree with the code that supposedly
produced them. Snakemake closes that gap.

You can inspect *why* a job will run with the `-r` flag from Chapter 4, and you can
control which triggers are active with `--rerun-triggers`. For example, if you
have only edited a comment and are certain the results are still valid, you might
restrict Snakemake to timestamps alone to avoid a large recompute:

```bash
$ snakemake -n -r                              # dry run, with reasons for each job
$ snakemake --cores 1 --rerun-triggers mtime   # consider only timestamps
```

> Use trigger overrides sparingly and deliberately. The defaults are conservative
> on purpose: they err toward recomputing when in doubt, which is the safe choice
> for trustworthy results. Reach for `--rerun-triggers mtime` as an occasional,
> conscious shortcut — never as a habit to silence rebuilds you do not understand.

## Where we are going

You can now read a pipeline as a graph, draw both the rule-level and job-level
views, request any target you like, and predict which jobs a change will trigger.
That is enough to confidently operate workflows others have written. The next step
is to stop hard-coding `SAMPLES = [...]` and scattered filenames in the Snakefile
and instead drive the whole pipeline from external **configuration** — a config
file and a sample sheet — so that running the same workflow on a new dataset means
editing data, not code. That is Chapter 7.
