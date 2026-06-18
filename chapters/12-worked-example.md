# Chapter 12 — A Worked Example, End to End

> *Everything, in one place — a real RNA-seq pipeline of the kind you will run and
> write.*

This chapter introduces no new ideas. Its job is synthesis: to assemble the
concepts and practices from the previous eleven chapters into one complete,
realistic RNA-seq quantification pipeline — the kind of project you will open on
your first week in the lab. We will build it the way we actually build pipelines:
modular, configuration-driven, with a conda environment, a log, and resources on
every rule. Then we will tour each piece, run it, and look at what comes out. Where
a feature appears, the chapter it came from is noted in parentheses, so this doubles
as a review.

## What the pipeline does

The biological goal is the everyday RNA-seq question from Chapter 1: *how strongly
is each transcript expressed in each sample?* Our pipeline answers it in four
stages:

1. **Quality control** with **FastQC** — scan each sample's reads for problems.
2. **Index** the reference transcriptome once with **Salmon**.
3. **Quantify** each sample against that index with Salmon, producing per-transcript
   counts.
4. **Aggregate** the per-sample results into a single counts matrix, and summarize
   the QC across all samples with **MultiQC**.

> We quantify with **Salmon**,[1] which uses *pseudoalignment*: rather than working
> out each read's exact position in the genome (slow), it rapidly determines which
> transcript each read is *compatible* with — enough to count expression, and fast
> enough to run many samples on modest hardware. **MultiQC**[2] gathers the
> per-sample QC outputs into one tidy report. Both are standard, current tools; this
> is a realistic pipeline, not a toy.

## The project layout

The project follows the standard layout (Chapter 3), modularized into rule files
(Chapter 11):

```
rnaseq-quant/
├── config/
│   ├── config.yaml
│   └── samples.tsv
├── workflow/
│   ├── Snakefile
│   ├── rules/
│   │   ├── common.smk
│   │   ├── qc.smk
│   │   ├── quant.smk
│   │   └── aggregate.smk
│   ├── envs/
│   │   ├── fastqc.yaml
│   │   ├── salmon.yaml
│   │   ├── multiqc.yaml
│   │   └── pandas.yaml
│   └── scripts/
│       └── aggregate_counts.py
├── resources/          # transcriptome.fa lives here (not in version control)
└── results/            # everything the pipeline produces
```

## Configuration

Nothing study-specific is hard-coded (Chapter 7). The config file holds settings:

```yaml
# config/config.yaml
samples: config/samples.tsv
transcriptome: resources/transcriptome.fa   # reference transcript sequences (FASTA)
kmer: 31                                     # Salmon index k-mer size
```

And the sample sheet lists the samples and their files, with a `condition` column
carrying the biological metadata you would use in a later differential-expression
analysis:

```
sample   condition   fastq
ctrl1    control     data/ctrl1.fastq.gz
ctrl2    control     data/ctrl2.fastq.gz
treat1   treated     data/treat1.fastq.gz
treat2   treated     data/treat2.fastq.gz
```

## The Snakefile: a table of contents

The top-level Snakefile loads the config, includes the rule files, and declares the
two final products via the `all` rule (Chapters 4, 11):

```python
# workflow/Snakefile
import pandas as pd

configfile: "config/config.yaml"

include: "rules/common.smk"
include: "rules/qc.smk"
include: "rules/quant.smk"
include: "rules/aggregate.smk"

rule all:
    input:
        "results/counts.tsv",
        "results/multiqc/multiqc_report.html",
```

## Shared setup: `common.smk`

The sample sheet is loaded into a DataFrame, the sample list is derived from it, and
a small helper looks up a sample's FASTQ path (Chapter 7):

```python
# workflow/rules/common.smk
samples = pd.read_csv(config["samples"], sep="\t").set_index("sample")
SAMPLES = list(samples.index)

def fastq_for(wildcards):
    return samples.loc[wildcards.sample, "fastq"]
```

## Quality control: `qc.smk`

One rule, applied to every sample by wildcard (Chapter 5), with its own conda
environment (Chapter 8), log (Chapter 10), and resources (Chapter 9):

```python
# workflow/rules/qc.smk
rule fastqc:
    input:
        fastq_for
    output:
        html="results/qc/{sample}_fastqc.html",
        zip="results/qc/{sample}_fastqc.zip",
    log:
        "logs/fastqc/{sample}.log"
    threads: 1
    resources:
        mem_mb=1000,
        runtime=15,
    conda:
        "../envs/fastqc.yaml"
    shell:
        "fastqc {input} --outdir results/qc/ > {log} 2>&1"
```

Note `input: fastq_for` — the bare function name is the input function from Chapter
7, now living tidily in `common.smk` instead of an inline `lambda`.

## Quantification: `quant.smk`

Two rules. The first builds the Salmon index once from the transcriptome; the second
quantifies each sample against it. The index is a *directory*, so its output is
wrapped in `directory()`:

```python
# workflow/rules/quant.smk
rule salmon_index:
    input:
        config["transcriptome"]
    output:
        directory("results/salmon_index")
    log:
        "logs/salmon/index.log"
    threads: 4
    resources:
        mem_mb=8000,
        runtime=30,
    conda:
        "../envs/salmon.yaml"
    shell:
        "salmon index -t {input} -i {output} -k {config[kmer]} -p {threads} > {log} 2>&1"

rule salmon_quant:
    input:
        index="results/salmon_index",
        reads=fastq_for,
    output:
        "results/quant/{sample}/quant.sf"
    log:
        "logs/salmon/{sample}.log"
    threads: 4
    resources:
        mem_mb=4000,
        runtime=30,
    conda:
        "../envs/salmon.yaml"
    shell:
        "salmon quant -i {input.index} -l A -r {input.reads} "
        "-p {threads} -o results/quant/{wildcards.sample} > {log} 2>&1"
```

This is the chapter's clearest illustration of rule chaining (Chapter 6): every
`salmon_quant` job depends on *both* the single `salmon_index` output (shared across
all samples) and that sample's own reads. The DAG therefore has one index node
fanning out to every quant node — exactly the build-once-reuse-many shape from the
Chapter 2 diagram.

## Aggregation: `aggregate.smk`

Finally, gather the per-sample results. The counts matrix is built by a Python
**script** rather than a shell command, because reshaping tables is what pandas is
for (Chapter 7). The QC summary is produced by MultiQC:

```python
# workflow/rules/aggregate.smk
rule aggregate_counts:
    input:
        expand("results/quant/{sample}/quant.sf", sample=SAMPLES)
    output:
        "results/counts.tsv"
    params:
        samples=SAMPLES
    log:
        "logs/aggregate.log"
    conda:
        "../envs/pandas.yaml"
    script:
        "../scripts/aggregate_counts.py"

rule multiqc:
    input:
        expand("results/qc/{sample}_fastqc.zip", sample=SAMPLES)
    output:
        "results/multiqc/multiqc_report.html"
    log:
        "logs/multiqc.log"
    conda:
        "../envs/multiqc.yaml"
    shell:
        "multiqc results/qc/ -o results/multiqc/ > {log} 2>&1"
```

The `script` directive points at a Python file instead of a `shell` command. Inside
that script, Snakemake makes a `snakemake` object available, carrying the rule's
`input`, `output`, and `params` — so the script needs no argument parsing:

```python
# workflow/scripts/aggregate_counts.py
import pandas as pd

# One column per sample, indexed by transcript name, holding read counts.
columns = {}
for sample, path in zip(snakemake.params.samples, snakemake.input):
    quant = pd.read_csv(path, sep="\t").set_index("Name")
    columns[sample] = quant["NumReads"]

matrix = pd.DataFrame(columns)
matrix.index.name = "transcript"
matrix.to_csv(snakemake.output[0], sep="\t")
```

Each Salmon `quant.sf` file has a `Name` column (the transcript) and a `NumReads`
column (its count). The script reads each sample's counts, lines them up by
transcript, and writes a single matrix — transcripts down the rows, samples across
the columns — which is the table a differential-expression analysis consumes:

```
transcript   ctrl1   ctrl2   treat1   treat2
ENST0001     100     94      60       58
ENST0002     42      39      88       91
...
```

## The environment files

Each rule's `conda` directive points at a pinned environment (Chapter 8). All four
are tiny:

```yaml
# workflow/envs/salmon.yaml
channels: [conda-forge, bioconda]
dependencies:
  - salmon =1.10.2
```

The other three are identical in shape: `fastqc =0.12.1`, `multiqc =1.21`, and
`pandas =2.2.2` (the last from `conda-forge` only). Pinning every version means
this pipeline produces the same numbers next year as it does today.

## Running it

Always dry-run first to inspect the plan (Chapter 4), then run for real, asking
Snakemake to build the conda environments (Chapter 8) and giving it eight cores
(Chapter 9):

```bash
$ snakemake -n                                        # dry run: review the plan
$ snakemake --cores 8 --software-deployment-method conda
```

Snakemake builds the DAG, runs `salmon_index` once, fans out the four `fastqc` and
four `salmon_quant` jobs across the available cores, then converges on
`aggregate_counts` and `multiqc`. On a cluster, the *same* command becomes
`snakemake --profile profiles/slurm` and the jobs spread across nodes (Chapter 9) —
with no change to anything above.

Take a look at the shape of the run before trusting it:

```bash
$ snakemake --rulegraph | dot -Tpng > rulegraph.png    # the six-rule structure
$ snakemake --dag       | dot -Tpng > dag.png          # every job for these samples
```

When it finishes, the two products are `results/counts.tsv` (the counts matrix
above) and `results/multiqc/multiqc_report.html` (the QC summary). And because the
whole thing is a workflow, you can bundle results, provenance, runtimes, and the
config into one shareable file (Chapter 11):

```bash
$ snakemake --report results/report.html
```

## How the pieces map to the book

Step back and notice that this single pipeline exercises essentially everything:

- **Rules, inputs, outputs, the DAG** (Ch. 2, 4, 6) — six rules chained through
  matching filenames.
- **Wildcards and `expand()`** (Ch. 5) — one rule per stage, applied across samples.
- **Config and sample sheet** (Ch. 7) — nothing study-specific in the code.
- **Per-rule conda environments, pinned** (Ch. 8) — reproducible software.
- **`threads` and `resources`** (Ch. 9) — runs on a laptop or a cluster unchanged.
- **`log` on every rule** (Ch. 10) — debuggable when a sample misbehaves.
- **Modular `include` layout, `script`, report** (Ch. 7, 11) — maintainable and
  shareable.

That correspondence is the point of the book: each chapter taught one piece, and a
real pipeline is just those pieces composed.

## Exercises: extend it

The best way to cement this is to grow the pipeline. In rough order of difficulty:

1. **Add read trimming.** Insert a `trim` rule (e.g. with `fastp` or `cutadapt`)
   between the raw reads and Salmon, so quantification runs on cleaned reads. You
   will write one new rule and re-point `salmon_quant`'s `reads` input at its output
   — and nothing else needs to change, which is the lesson.
2. **Support paired-end reads.** Add a `fastq2` column to the sample sheet and an
   input function returning both files; update `salmon_quant` to use `-1`/`-2`
   instead of `-r`.
3. **Download the reference automatically.** Write a rule whose output is
   `resources/transcriptome.fa`, fetching it from a recorded URL — so even the
   reference is reproducible rather than a file you copied in by hand.
4. **Add differential expression.** Write a final `script` rule that reads
   `counts.tsv` and the `condition` column from the sample sheet and runs a
   differential-expression test (e.g. with `pydeseq2`), producing a results table.
   This turns the pipeline from "quantify" into "find the genes that changed."

Each extension reuses exactly the machinery from this book; none requires a new
concept. When you can add these comfortably, you are ready for the real pipelines in
the lab.

## The end (and the beginning)

That is the whole book. You started with a fragile shell script and the problem of
trust; you end with a reproducible, scalable, maintainable pipeline and the
understanding to read, run, debug, and extend the ones you will inherit. The
appendices that follow are reference material — a directive and command cheat sheet,
a Python and shell refresher, a glossary, and the consolidated references — to keep
within reach as you start writing workflows of your own. Welcome to the group.

---

## References

1. Patro R, Duggal G, Love MI, Irizarry RA, Kingsford C. Salmon provides fast and
   bias-aware quantification of transcript expression. *Nature Methods*.
   2017;14(4):417–419.
2. Ewels P, Magnusson M, Lundin S, Käller M. MultiQC: summarize analysis results for
   multiple tools and samples in a single report. *Bioinformatics*.
   2016;32(19):3047–3048.
