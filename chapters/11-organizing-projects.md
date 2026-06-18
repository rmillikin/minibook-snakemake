# Chapter 11 — Organizing Larger Projects

> *From one Snakefile to a maintainable, shareable codebase.*

Every pipeline in this book has lived in a single Snakefile, which is fine while it
fits on a screen. Real projects do not. A production RNA-seq workflow has QC,
trimming, alignment, quantification, and differential-expression steps, each with
several rules — dozens in total, plus helper code. Cram them into one file and it
becomes a wall of text nobody can navigate. This chapter is about the practices
that keep a growing project legible and reusable: splitting it into files, reusing
components, generating reports and provenance, and the version-control hygiene that
makes a pipeline safe to share. None of this changes how rules *work* — it changes
how a *project* is structured around them.

## Splitting rules across files: `include`

The first move is simply to stop putting everything in one file. The **`include`**
directive pulls the contents of another file into the Snakefile as if it had been
typed there. By convention, rule files use the `.smk` extension and live in
`workflow/rules/`, grouped by the *stage* of the pipeline they implement.

The main `workflow/Snakefile` shrinks to a table of contents: load configuration,
include the rule files, and declare the final target.

```python
import pandas as pd

configfile: "config/config.yaml"

include: "rules/common.smk"     # shared setup and helper functions
include: "rules/qc.smk"         # quality-control rules
include: "rules/quant.smk"      # quantification rules

rule all:
    input:
        "results/report.txt"
```

A natural home for the sample-sheet loading and small helper functions is a
`common.smk` included first, so everything after it can use them:

```python
# workflow/rules/common.smk
samples = pd.read_csv(config["samples"], sep="\t").set_index("sample")
SAMPLES = list(samples.index)

def fastq_for(wildcards):
    return samples.loc[wildcards.sample, "fastq"]
```

Now a rule's input function reads cleanly as `input: fastq_for` instead of an inline
`lambda`, and each stage's rules sit in their own file. The payoff is navigational:
to find the alignment rules you open `rules/align.smk`, not line 340 of a monolith.
The DAG and behavior are identical to one big file — `include` is purely
organizational — but a newcomer (again, often *you* in six months) can find their
way around in seconds.

## Reusing whole workflows: modules

`include` organizes rules *within* a project. Sometimes you want to reuse rules
*across* projects — a standard QC sub-workflow, say, that several pipelines share.
The **`module`** system lets one workflow import another's rules wholesale:

```python
module qc_workflow:
    snakefile: "https://github.com/our-lab/qc-workflow/raw/v1.2.0/workflow/Snakefile"
    config: config

use rule * from qc_workflow        # bring in all of its rules
```

The `use rule` statement imports rules from the module; you can take them all
(`*`), take specific ones, or import a rule and override parts of it (a different
input, more threads). Because the module can be pinned to a version (`v1.2.0`
above), reuse stays reproducible — you get exactly that release's rules, not
whatever the source happens to look like today. Modules are how a lab builds a
library of trusted, shared workflow components instead of copy-pasting rules
between projects. You will not need them on day one, but recognize the pattern when
you see it in our shared pipelines.

## Reports and provenance

A finished run leaves its outputs scattered across `results/`. Snakemake can gather
them, along with a complete record of *how* they were produced, into a single
self-contained HTML **report**:

```bash
$ snakemake --report results/report.html
```

The report bundles the results you flagged for inclusion, the workflow's DAG, the
runtime of every job, the configuration used, and the exact code of each rule —
into one file you can hand to a collaborator or attach to a paper's supplement. To
mark which outputs appear in it, wrap them in `report()` with a caption:

```python
output:
    report("results/report.txt", caption="report/counts.rst", category="Counts")
```

This is provenance made tangible: the report *is* the answer to "how did you make
this figure?" — the question Chapter 1 warned you would eventually be asked. Getting
it for free, as a byproduct of having written the analysis as a workflow, is much of
the reason we use Snakemake at all.

## Benchmarking

To run a pipeline efficiently you need to know what each step actually costs — which
feeds directly back into the `resources` declarations of Chapter 9. The
**`benchmark`** directive records a job's wall-clock time, memory, and CPU usage to a
file:

```python
rule fastqc:
    input:
        "results/{sample}.fastq"
    output:
        "results/qc/{sample}_fastqc.html"
    benchmark:
        "benchmarks/{sample}.fastqc.txt"
    conda:
        "envs/fastqc.yaml"
    shell:
        "fastqc {input} --outdir results/qc/"
```

After a run, `benchmarks/sampleA.fastqc.txt` holds the measured time and peak memory
for that job. Benchmark a few representative samples, read off the real numbers, and
set `resources: mem_mb` and `runtime` from evidence rather than guesswork — the
difference between jobs that sail through the scheduler and jobs that get killed or
wait forever in the queue.

## Version control hygiene

A workflow is code, and code belongs in **version control** (Git) — but a
bioinformatics *project* also contains huge data files that emphatically do not. The
guiding question is: *is this file a source you authored, or a product you can
regenerate?*

**Commit** (it is small, authored, and defines the analysis):

- `workflow/` — the Snakefile, rule files, scripts.
- `config/` — `config.yaml` and the sample sheet.
- `workflow/envs/` — the conda environment files.

**Do not commit** (it is large, or regenerable, or both); list it in `.gitignore`:

- `results/` — everything the pipeline produces. It can be rebuilt from the inputs
  and the code, which is the whole point.
- `resources/` and raw data — reference genomes and FASTQ files are gigabytes and do
  not belong in Git. Track *where they came from* (a download rule, an accession
  number, a path) instead of the bytes themselves.

A `.gitignore` enforcing this is short:

```
results/
resources/
.snakemake/
```

The litmus test: someone should be able to clone your repository — small enough to
download in seconds — and reproduce every result by running `snakemake`, pulling the
large inputs from their recorded source. If reproducing your work requires you to
email someone a 50 GB folder, the project is not really reproducible.

## A starting checklist

When you set up a new project, aim for this shape — it is what every well-kept
pipeline in the group looks like, and it operationalizes the whole book:

- Standard layout (`config/`, `workflow/`, `resources/`, `results/`) — Chapter 3.
- Samples and settings in `config/`, never hard-coded — Chapter 7.
- Rules split by stage into `workflow/rules/*.smk` — this chapter.
- A `conda` environment per non-trivial rule, versions pinned — Chapter 8.
- `threads`/`resources` declared, set from benchmarks — Chapters 9 and this one.
- A `log` for every non-trivial rule — Chapter 10.
- A `.gitignore` excluding `results/`, `resources/`, and `.snakemake/`.
- A `README` saying how to run it and where the input data lives.

## Where to find help

You are not expected to memorize all of this. The resources you will actually lean
on:

- **The official documentation** (snakemake.readthedocs.io) — thorough, with a
  hands-on tutorial that complements this book.
- **The Snakemake Wrapper Repository** (Chapter 8) — before writing a rule for a
  common tool, check whether a vetted wrapper already exists.
- **The community** — the Snakemake tag on Stack Overflow and the project's GitHub
  discussions; most errors you hit have been hit before.
- **Our group** — and, most importantly, our existing pipelines and the labmates who
  wrote them. Reading a working pipeline you can ask questions about is the fastest
  way to learn.

## Where we are going

You now have the full toolkit: the concepts (Chapters 1–6), and the practices that
make a pipeline reproducible (7–8), scalable (9), robust (10), and maintainable
(this chapter). Chapter 12 puts it all together in one place — a complete, realistic
RNA-seq pipeline that uses every one of these pieces, so you can see how they
combine in the kind of workflow you will actually run and write in the lab.

---

## References

1. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33. (Modularization, reports, and
   benchmarking.)
