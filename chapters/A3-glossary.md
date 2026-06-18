# Appendix C — Glossary

> *Definitions of the terms bolded throughout the book, computational and
> biological. The chapter of first use is noted in brackets.*

**Acyclic** — having no cycles: you can never follow the directed edges of the
graph and return to where you started. The property that guarantees a workflow can
start somewhere and finish. [2]

**Alignment** — determining where in a reference genome (or which transcript) a
sequencing read came from. Contrast with *pseudoalignment*. [1, 12]

**`all` rule** — the conventional first rule in a Snakefile, with no output and no
recipe, whose `input` lists the workflow's final target files. Becomes the default
target. [4]

**Apptainer** (formerly **Singularity**) — a container system designed for shared
HPC clusters, usable without administrator privileges. [8]

**Atomic (output)** — the guarantee that an output file either exists complete or
not at all; Snakemake deletes a failed job's outputs so half-written files are never
mistaken for finished ones. [10]

**Bash** — the common Unix shell language in which `shell:` recipes are run. [1]

**Benchmark** — a record of a job's wall-clock time, memory, and CPU usage, produced
by the `benchmark` directive. [11]

**Bioconda** — a Conda channel packaging thousands of bioinformatics tools. [3]

**Channel** — a repository a package manager downloads from; this book uses
`conda-forge` and `bioconda`. [3]

**Conda** — a package manager widely used in bioinformatics; pairs with virtual
environments to isolate software. [3]

**Config file** — a YAML file (loaded by `configfile:`) holding a workflow's
settings, read in the Snakefile as the `config` dict. [7]

**Container** — a bundle of a program plus its entire software environment, packaged
into an *image* that runs identically on any machine with a container runtime. [8]

**Core** — one physical processing unit of a CPU. The budget set by `--cores`. [9]

**DAG (directed acyclic graph)** — the structure of a workflow: nodes are jobs,
directed edges are file dependencies, and no cycles are allowed. Snakemake reasons
over it to plan and run work. [2]

**DataFrame** — pandas' in-memory table: labeled columns and indexed rows, with
lookups like `df.loc[row, col]`. [7]

**Declarative** — describing *what* should exist and the relationships between
things, leaving the system to work out the order. Snakemake's style; contrast
*imperative*. [2]

**Default target** — what Snakemake builds when given no target on the command line:
the output of the first rule (by convention, the `all` rule). [4]

**Dependency** — the relationship "step B needs step A's output before it can run."
Edges of the DAG. [1]

**Directory()** — an output marker indicating the output is a directory. [12]

**Docker** — a widely used container system; images are often named
`docker://...`. [8]

**Dry run** — `snakemake -n`: print the plan without executing anything. [4]

**Executor** — the plug-in deciding *where and how* jobs run (local cores, SLURM,
cloud). Switching executors changes the backend, not the rules. [9]

**`expand()`** — a helper that fills a filename pattern from one or more lists,
producing the list of resulting filenames. [5]

**FASTQ** — a text format storing sequencing reads with per-base quality scores,
using four lines per read. [4]

**FastQC** — a standard tool that scans sequencing reads for quality problems. [8]

**Gene expression** — how strongly a gene is transcribed into RNA in a sample; what
RNA-seq measures. [1]

**HPC cluster** — a high-performance computing cluster: many networked machines
shared by many users, accessed via a job scheduler. [3, 9]

**Image** — a packaged container; the shareable artifact a container runs from. [8]

**Imperative** — spelling out steps in order for the computer to execute literally.
Contrast *declarative*. [2]

**Include** — the directive that splices another rule file (`.smk`) into the
Snakefile. [11]

**Incomplete (output)** — a file left in an untrusted state because Snakemake itself
was killed mid-job; redone with `--rerun-incomplete`. [10]

**Input function** — an `input` given as a function of `wildcards` (often a
`lambda`), used to look up per-sample files from a sample sheet. [7]

**Intermediate file** — a file produced only to feed a later step, not a final
result. [1]

**Job** — one rule with its wildcards filled in: a single node of the DAG. [5]

**Job scheduler** — the cluster's traffic controller (e.g. **SLURM**) that places
submitted jobs onto nodes according to their resource requests. [9]

**`lambda`** — Python's syntax for a small unnamed function. [7]

**`log` directive** — declares a per-job file capturing the recipe's output, for
diagnosis. [10]

**Make** — the classic file-and-dependency build tool whose model Snakemake
adapts. [1, 2]

**Mamba** — a fast drop-in reimplementation of Conda. [3]

**Module** — a mechanism (`module` / `use rule`) for importing another workflow's
rules wholesale, enabling cross-project reuse. [11]

**MultiQC** — a tool that aggregates many per-sample QC outputs into one report. [12]

**Node / edge** — a node is a vertex of a graph (here, a job); an edge is a
connection between two (here, a file dependency). [2]

**Package manager** — software that installs programs and their dependencies at
compatible versions (e.g. Conda). [3]

**pandas** — the standard Python library for tabular data; provides the
DataFrame. [7]

**`params` directive** — declares named values used in a recipe as `{params.name}`,
keeping settings out of the shell string. [7]

**Pipeline (workflow)** — an ordered series of data transformations turning raw
input into finished results. [1]

**Profile** — a directory of default run settings (executor, job limits, resources)
selected with `--profile`, so the same workflow runs locally or on a cluster
unchanged. [9]

**`protected()`** — an output marker making a file read-only after success, guarding
expensive results. [10]

**Provenance** — a trustworthy record of how a result was produced: which inputs,
tools, versions, and parameters. [1]

**Pseudoalignment** — rapidly determining which transcript a read is compatible
with, rather than its exact position; used by Salmon for fast quantification. [12]

**Python** — the general-purpose language Snakemake is built on and written in. [3]

**Quantification** — counting, per gene or transcript, how many reads were assigned
to it. [1, 12]

**Reads** — the short sequences of bases output by a sequencing machine. [1]

**Recipe** — the commands that turn a rule's inputs into its outputs (`shell` or
`script`). [2]

**Regular expression** — a notation for describing text patterns, used in
`wildcard_constraints`. [5]

**Report** — a self-contained HTML summary (`--report`) bundling results,
provenance, runtimes, config, and rule code. [11]

**Reproducibility** — the ability for someone else (or future you) to obtain the
same result from the same data and methods. [1]

**Rerun triggers** — the conditions under which Snakemake recomputes a job: changed
input timestamps (`mtime`), code, params, input set, or software environment. [6]

**Resources** — declared job needs beyond cores (e.g. `mem_mb`, `runtime`), used
locally and as cluster requests. [9]

**RNA-seq** — RNA sequencing: measuring which genes are expressed, and how strongly,
by sequencing a sample's RNA. [1]

**Rule** — Snakemake's atomic unit: a recipe producing `output` file(s) from
`input` file(s). [2]

**Rule chaining** — Snakemake connecting two rules when one's output pattern can
produce a file matching another's input. [6]

**Rule graph** — the per-*rule* view of a workflow (`--rulegraph`): its structure,
independent of sample count. Contrast the DAG (per-*job*). [6]

**Salmon** — a fast, bias-aware transcript quantification tool using
pseudoalignment. [12]

**Sample** — one sequenced biological specimen, processed independently through the
early pipeline steps. [5]

**Sample sheet** — a table (one row per sample) of sample names, file paths, and
metadata; read with pandas. [7]

**`script` directive** — runs an external Python/R file as the recipe; the file
receives a `snakemake` object with `input`/`output`/`params`. [12]

**Shell / command line** — the text interface for running programs. [1]

**Shell script** — a file of shell commands run as a batch; the fragile "before"
picture of Chapter 1. [1]

**Snakefile** — the file Snakemake reads (by convention `workflow/Snakefile`). [4]

**Standard error / standard output** — the two output streams of a Unix program
(diagnostics vs. normal output), numbered `2` and `1`. [10]

**Target** — a file you ask Snakemake to produce; determines how much of the DAG
runs. [2]

**`temp()`** — an output marker that deletes the file once all consuming jobs
finish, to save disk. [10]

**Thread** — one stream of computation; multi-threaded tools use several cores at
once. Declared with `threads`. [9]

**Transcriptome** — the set of transcript sequences a quantifier maps reads
against. [12]

**Version control** — tracking code changes over time (Git); workflow code is
committed, large data is not. [11]

**Version pinning** — recording a tool's exact version (e.g. `salmon =1.10.2`) so an
environment is reproducible. [8]

**Virtual environment** — an isolated directory of installed software, letting
projects hold conflicting versions without interfering. [3]

**Wildcard** — a named placeholder (e.g. `{sample}`) in a path; its value is
inferred by matching a requested output filename, then propagated through the
rule. [5]

**Wildcard constraints** — restrictions (regex) on what a wildcard may match,
resolving ambiguity. [5]

**Workflow management system** — a tool that runs pipelines correctly: tracking
dependencies, ordering, rebuilds, parallelism, and provenance. Snakemake is one. [1]

**Wrapper** — a vetted, versioned, reusable rule body for a common tool, from the
Snakemake Wrapper Repository. [8]

**YAML** — a human-readable data format used for config and conda environment
files. [7]
