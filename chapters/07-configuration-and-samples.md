# Chapter 7 — Configuration and Sample Sheets

> *Separating the pipeline's logic from the data and settings it runs on.*

Welcome to the second half of the book. Chapters 1–6 taught you to build and read
a working pipeline; the chapters ahead are about the practices that turn a working
pipeline into one a lab can *trust, share, and reuse*. We begin with the most
immediately useful of them: getting the data and the parameters *out of the
Snakefile*.

Look back at the pipeline from Chapter 6 and you will spot two things baked into
the code that should not be there: the literal list `SAMPLES = ["sampleA", ...]`,
and filename patterns like `data/{sample}.fastq.gz` that assume where the raw data
lives. The moment you want to run the same analysis on a different study, you must
*edit the code* — and editing working code just to point it at new data is exactly
how working code stops working.

## Why hard-coding is a trap

The principle here is one good engineers and good scientists share: **separate the
things that change from the things that stay the same.** A pipeline's *logic* —
decompress, then count, then report — is stable; it is the same for every RNA-seq
study you will ever run. What changes from study to study is the *data* (which
samples, where their files are) and the *settings* (parameter values, thresholds,
labels). Mixing the two has three costs:

- **Maintenance.** Every new dataset means hand-editing the Snakefile, and every
  edit risks breaking logic that was fine.
- **Reuse.** You cannot hand the workflow to a labmate for *their* data without
  them rewriting parts of it.
- **Provenance.** When the settings are scattered through the code as literals,
  there is no single place that records "here is exactly what this run was
  configured to do" — which is precisely the record a reproducible analysis needs.

The fix is to move data and settings into dedicated **configuration files** that
sit beside the workflow, and have the Snakefile *read* them. The logic lives in
`workflow/`; the configuration lives in `config/` — the split the Chapter 3 layout
anticipated.

## The config file and a YAML primer

Snakemake's general-purpose settings file is written in **YAML**, a format
designed to be easy for humans to read and write. You only need a few rules of
YAML:

```yaml
# config/config.yaml — lines starting with # are comments

report_title: "RNA-seq read counts, pilot study"   # a key mapped to a string
min_reads: 1000                                     # a number
paired_end: false                                   # a boolean (true/false)

samples: config/samples.tsv                         # path to our sample sheet

adapters:                                           # a list of values
  - AGATCGGAAGAGC
  - CTGTCTCTTATA
```

The essentials: data is written as `key: value` pairs; **indentation** (spaces,
never tabs) expresses nesting; lists are lines beginning with `-`; and strings,
numbers, and booleans are written as you would expect. That is enough YAML to
configure almost any workflow.

To load this file, add one directive near the top of the Snakefile:

```python
configfile: "config/config.yaml"
```

This reads the YAML into a Python **dictionary** (Chapter 3) named `config`, which
the whole workflow can then read from. Every top-level key becomes a lookup:

```python
config["report_title"]   # → "RNA-seq read counts, pilot study"
config["min_reads"]      # → 1000
```

Now a threshold or a label is set in *one obvious place*, the config file is a
self-contained record of how a run was configured, and — recall the rerun triggers
of Chapter 6 — changing a `params` value derived from config will correctly
trigger the affected jobs to re-run.

## The sample sheet

Listing samples as a Python list was fine for three. But real studies come with a
**sample sheet**: a table, one row per sample, carrying the metadata you need —
the sample's name, the path to its data, and biological attributes like
experimental condition, tissue, or batch. Storing samples as a table (rather than
buried in code) means the same file can be shared with collaborators, opened in a
spreadsheet, and version-controlled as the authoritative list of what is in the
study.

We use a tab-separated file, `config/samples.tsv`:

```
sample    condition    fastq
sampleA   control      data/sampleA.fastq.gz
sampleB   control      data/sampleB.fastq.gz
sampleC   treated      data/sampleC.fastq.gz
```

To use it in the Snakefile, we read it with **pandas**, the standard Python
library for tabular data.

> **pandas** loads a table into a **DataFrame** — think of it as a spreadsheet
> living in memory: labeled columns, indexed rows, and convenient lookups. If we
> set the row index to the `sample` column, then `df.loc["sampleB", "fastq"]`
> retrieves sample B's file path, much as you would find a cell by its row and
> column headers. This is the one bit of pandas the book relies on.

The reading code goes near the top of the Snakefile, just after the `configfile`
line:

```python
import pandas as pd

configfile: "config/config.yaml"

samples = pd.read_csv(config["samples"], sep="\t").set_index("sample")
SAMPLES = list(samples.index)     # ["sampleA", "sampleB", "sampleC"]
```

Two things just happened. We loaded the sample sheet into a DataFrame named
`samples`, and we derived `SAMPLES` *from the table* instead of hard-coding it.
Adding a sample to the study is now a one-line edit to `samples.tsv` — no code
changes at all. Note also that the path to the sample sheet itself came from
`config["samples"]`, so even *that* is configurable.

## Looking up per-sample data: input functions

The `decompress` rule previously assumed every sample's file was at
`data/{sample}.fastq.gz`. But our sample sheet has an explicit `fastq` column —
the files could live anywhere, with any names. We want `decompress` to look up
*this* sample's path from the table.

A fixed string pattern cannot do that lookup. For cases like this Snakemake lets an
`input` be a **function** of the wildcards instead of a string. The function
receives the job's wildcards and returns the filename(s):

```python
rule decompress:
    input:
        lambda wildcards: samples.loc[wildcards.sample, "fastq"]
    output:
        "results/{sample}.fastq"
    shell:
        "gunzip -c {input} > {output}"
```

> A `lambda` is just Python's syntax for a small unnamed function;
> `lambda wildcards: ...` reads as "given the wildcards, return this." When
> Snakemake builds the job for `sampleB`, it calls the function with
> `wildcards.sample == "sampleB"`, and `samples.loc["sampleB", "fastq"]` returns
> `data/sampleB.fastq.gz`. The sample sheet now *drives the inputs*. These
> **input functions** are the bridge between your metadata table and the DAG, and
> you will use them constantly once your pipelines read from sample sheets.

## Keeping parameters out of the recipe: the `params` directive

There is one more place values sneak into rules: the recipe itself. It is tempting
to write thresholds and labels directly into a `shell` command, but a value buried
in a command string is invisible to Snakemake — it cannot be reported, and
changing it will not be recognized as a reason to re-run.

The **`params`** directive is the proper home for such values. It declares named
parameters, computed once, that the recipe then refers to as `{params.<name>}` —
mirroring how `{input}` and `{output}` work. Here is the `report` rule pulling its
title from config via `params`:

```python
rule report:
    input:
        expand("results/{sample}.count.txt", sample=SAMPLES)
    output:
        "results/report.txt"
    params:
        title=config["report_title"]
    shell:
        "echo '# {params.title}' > {output} && cat {input} >> {output}"
```

The title now comes from one configurable place; it appears in Snakemake's job
provenance; and — per the rerun triggers of Chapter 6 — editing
`report_title` in the config will correctly mark the report stale and rebuild it.
The habit to form: **numbers, flags, thresholds, and labels belong in `params`
(usually sourced from config), not hard-coded into the shell string.**

## The fully configured pipeline

Putting it all together, here is the Chapter 6 pipeline refactored to be driven
entirely by configuration. The *logic* is unchanged; what changed is that nothing
study-specific is hard-coded anymore:

```python
import pandas as pd

configfile: "config/config.yaml"

samples = pd.read_csv(config["samples"], sep="\t").set_index("sample")
SAMPLES = list(samples.index)

rule all:
    input:
        "results/report.txt"

rule decompress:
    input:
        lambda wildcards: samples.loc[wildcards.sample, "fastq"]
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
    params:
        title=config["report_title"]
    shell:
        "echo '# {params.title}' > {output} && cat {input} >> {output}"
```

To run this same workflow on a different study, you now touch *no code at all*: you
edit `config/samples.tsv` to list the new samples and their files, adjust any
settings in `config/config.yaml`, and run `snakemake`. The pipeline has become a
reusable instrument rather than a one-off script — and the two small config files
are a clean, shareable record of exactly what any given run was.

## Where we are going

Configuration handles *which data* and *what settings*. It does not yet handle the
other half of reproducibility raised back in Chapter 1: *which software, at which
version*. Our recipes still silently assume that `gunzip` — and, in a real
pipeline, an aligner and a counter — are installed and are the versions you expect.
Chapter 8 closes that gap with per-rule software environments and containers, so
that a pipeline carries its own software with it.

---

## References

1. McKinney W. Data structures for statistical computing in Python. *Proceedings
   of the 9th Python in Science Conference (SciPy)*. 2010:56–61. (The pandas
   library.)
