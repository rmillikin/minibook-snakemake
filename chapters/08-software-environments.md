# Chapter 8 — Reproducible Software Environments

> *Making "it worked on my machine" go away — the other half of reproducibility.*

Chapter 7 made *which data* and *which settings* a run uses explicit and portable.
This chapter handles the part of reproducibility that has quietly haunted us since
Chapter 1: *which software, at which version*. Up to now our recipes have leaned
on whatever happened to be installed — `gunzip`, `wc` — and assumed it behaves the
same everywhere. For coreutils that is a safe bet. For the scientific tools a real
RNA-seq pipeline depends on, it is not, and the consequences land directly on your
results.

## The problem: results depend on versions

Bioinformatics tools change. An aligner's default parameters get retuned between
versions; a quantifier fixes a bug that shifts counts; a library updates a
statistical method. This is normal and healthy — but it means the *version* of a
tool is part of your method, every bit as much as the parameters you pass it. Two
runs of "the same" pipeline with different tool versions can produce different
numbers, and if nobody recorded which version was used, the discrepancy is
unexplainable and the work is not reproducible.[1]

The naive fix — "just install the tools" — recreates the dependency hell from
Chapter 3: every project wants different, often conflicting, versions, and a
system-wide install can serve only one at a time. What we want is for **each rule
to carry its own software, pinned to an exact version**, independent of whatever
else is on the machine. Snakemake offers two complementary ways to do this:
per-rule conda environments, and containers.

## Per-rule conda environments

Chapter 3 used Conda to install Snakemake itself into one environment. The same
mechanism can supply software *per rule*: you describe the environment a rule needs
in a small YAML file, and Snakemake creates that environment on demand and runs the
rule inside it.

To make this concrete, let us add a real RNA-seq step to the running pipeline:
**quality control** with **FastQC**, a standard tool that scans reads for problems
(low-quality stretches, leftover adapter sequence, unusual base composition) and
emits a report.[2] FastQC is not a coreutil — it must be installed — which makes it
the perfect candidate for a per-rule environment.

First, describe the environment in `workflow/envs/fastqc.yaml`:

```yaml
channels:
  - conda-forge
  - bioconda
dependencies:
  - fastqc =0.12.1
```

This says: from the `conda-forge` and `bioconda` channels, install FastQC, **pinned
to version 0.12.1**. That `=0.12.1` is the crucial part. **Version pinning** —
recording the exact version rather than "whatever is latest" — is what makes the
environment reproducible: anyone, anytime, gets the same FastQC, so they get the
same QC results.

Then attach the environment to a rule with the **`conda`** directive:

```python
rule fastqc:
    input:
        "results/{sample}.fastq"
    output:
        "results/qc/{sample}_fastqc.html"
    conda:
        "envs/fastqc.yaml"
    shell:
        "fastqc {input} --outdir results/qc/"
```

The `conda:` line points at the environment file (relative to the Snakefile). When
you run the workflow and ask Snakemake to honor these declarations, it creates the
environment (once, then caches it) and runs the `fastqc` command *inside* it:

```bash
$ snakemake --cores 1 --software-deployment-method conda
```

> The flag `--software-deployment-method conda` (often abbreviated `--sdm conda`)
> tells Snakemake to actually build and use the declared environments. Older
> workflows and tutorials use the now-deprecated spelling `--use-conda`; you will
> see both, and they mean the same thing. Without such a flag, the `conda:`
> directive is ignored and the rule runs with whatever is on your `PATH` — so
> remember to pass it.

The payoff is large and almost free. Each rule now declares its own software, those
declarations live in version control beside the code, and the environment files are
a precise, machine-readable record of every tool and version the pipeline used.
Different rules can even depend on conflicting versions of the same tool without any
trouble, because each runs in its own isolated environment.

## Containers

A conda environment pins the *packages* but still runs on top of the host machine's
operating system and system libraries. For most work that is enough. When you need
to pin *everything* — the OS, system libraries, and all — or when a tool is awkward
to package with conda, the heavier-duty option is a **container**.

> A **container** bundles a program together with its entire software environment —
> libraries, dependencies, sometimes a whole minimal operating system — into a
> single shareable **image** that runs identically on any machine with a container
> runtime. **Docker** is the best-known system for building and running them.
> **Apptainer** (formerly **Singularity**) is the variant designed for shared HPC
> clusters, because — unlike Docker — it does not require administrator (root)
> privileges, which multi-user clusters do not grant.

Snakemake attaches a container to a rule with the **`container`** directive, naming
an image (commonly pulled from a public registry):

```python
rule fastqc:
    input:
        "results/{sample}.fastq"
    output:
        "results/qc/{sample}_fastqc.html"
    container:
        "docker://quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0"
    shell:
        "fastqc {input} --outdir results/qc/"
```

```bash
$ snakemake --cores 1 --software-deployment-method apptainer
```

**When to use which?** A useful rule of thumb:

- **Conda** is lighter, faster to set up, and easy to compose and tweak (edit a
  YAML line to add a package). It is the right default for most rules and most of
  our group's pipelines.
- **Containers** give the strongest reproducibility guarantee — the whole
  environment is frozen — and are the better choice when a tool has gnarly system
  dependencies, when you must reproduce results years later with certainty, or when
  a project standardizes on published images. On our cluster, containers run via
  Apptainer.

The two are not mutually exclusive; Snakemake can even run a rule's conda
environment *inside* a container for belt-and-suspenders reproducibility. For a new
student, the practical guidance is: **prefer conda environments per rule, and reach
for containers when conda cannot cleanly express what a rule needs.**

## A note on wrappers

There is a recurring frustration lurking here: for common tools like FastQC, every
lab writes nearly the same rule — same environment, same command shape, same output
handling. The **Snakemake Wrapper Repository** exists to stop that duplication. A
**wrapper** is a vetted, reusable, versioned rule body for a popular tool,
maintained by the community; it bundles the environment and the command logic so you
supply only inputs, outputs, and parameters.[3]

You use one with the **`wrapper`** directive instead of writing `shell` and `conda`
yourself:

```python
rule fastqc:
    input:
        "results/{sample}.fastq"
    output:
        html="results/qc/{sample}.html",
        zip="results/qc/{sample}_fastqc.zip",
    wrapper:
        "v3.10.2/bio/fastqc"
```

The version prefix (`v3.10.2`) pins the wrapper itself, so its behavior — including
the tool version it installs — is fixed. Wrappers are worth knowing about because
they let you assemble a pipeline from well-tested building blocks rather than
reinventing each step; browse the repository before hand-writing a rule for any
common tool. We will not lean on wrappers in this book's examples (writing rules by
hand is more instructive while you are learning), but in day-to-day work they save
real effort and reduce bugs.

## Where we are going

With Chapters 7 and 8 together, a pipeline now carries everything it needs to be
reproduced: the logic in `workflow/`, the data and settings in `config/`, and the
exact software in per-rule environment files or container images. Someone can clone
the project and reproduce your results on a different machine, which is the whole
goal we set out in Chapter 1.

What we have *not* yet addressed is scale. Running QC and quantification on three
toy samples on one core is one thing; running a real study of dozens of multi-
gigabyte samples is another. Chapter 9 turns to performance: telling rules how many
threads and how much memory they need, and dispatching the workflow's jobs across
the many cores of a cluster — without changing a line of the pipeline's logic.

---

## References

1. Di Tommaso P, Chatzou M, Floden EW, et al. Nextflow enables reproducible
   computational workflows. *Nature Biotechnology*. 2017;35(4):316–319. (On the
   role of software versioning and isolation in reproducible pipelines.)
2. Andrews S. FastQC: a quality control tool for high throughput sequence data.
   2010. Babraham Bioinformatics.
   https://www.bioinformatics.babraham.ac.uk/projects/fastqc/
3. Köster J, et al. The Snakemake Wrapper Repository.
   https://snakemake-wrappers.readthedocs.io/
