# Chapter 3 — Getting Set Up

> *Just enough Python, a clean way to install things, and a place to put them.*

Chapters 1 and 2 were about ideas. This one is about getting a working setup on
your machine so that, by Chapter 4, you can run a real pipeline. There are three
things to take care of: the small amount of **Python** you need to read Snakemake
workflows, a clean way to **install** Snakemake and the tools it orchestrates,
and a sensible **place to put** a project. None of this is hard, but doing it the
conventional way now will save you a great deal of confusion later.

## Just enough Python

Recall the claim from Chapter 1 that a Snakemake workflow file *is* a Python file
with extra syntax. That is literally true, and it has a happy consequence: to
read and write workflows you need only a small slice of Python, even if you have
never used the language. This section is that slice. (Appendix B is a fuller
refresher if you want one; here we cover only what the next few chapters use.)

> **Python** is a general-purpose programming language that is the lingua franca
> of data science and bioinformatics. It is known for readable syntax and uses
> *indentation* (leading spaces) rather than braces to group code — which is why,
> in a Snakefile, the parts of a rule are indented underneath it.

Four pieces of Python syntax account for almost everything you will see:

**Strings** are text, written in quotes. Filenames in Snakemake are just strings:

```python
"aligned/sampleA.bam"
```

**Lists** are ordered collections, written in square brackets. A rule with
several inputs holds them in a list:

```python
["raw/sampleA.fastq.gz", "raw/sampleB.fastq.gz"]
```

**Dictionaries** (**dicts**) map *keys* to *values*, written in curly braces with
`key: value` pairs. Snakemake's configuration, which you will meet in Chapter 7,
arrives as a dict — you look things up by name:

```python
{"genome": "GRCh38", "threads": 8}
# config["genome"] would give "GRCh38"
```

**f-strings** are strings prefixed with `f` that let you insert a value into text
by wrapping it in braces. They are how you build filenames out of parts:

```python
sample = "sampleA"
f"aligned/{sample}.bam"     # becomes "aligned/sampleA.bam"
```

> Keep that last one in mind. Snakemake's `{...}` placeholders from Chapter 2
> look like f-string braces on purpose — both mean "substitute a value in here."
> They are not *exactly* the same mechanism, but the intuition carries over.

That is genuinely most of it. You will also see ordinary function calls like
`expand(...)` (Chapter 5) and the occasional `for` loop, but you do not need to
write sophisticated Python to be productive. When a workflow does reach for real
Python, it is usually doing something you would otherwise have done by hand, like
reading a table of samples — and that is a feature, not a burden.

## Installing software the reproducible way

You could install Snakemake the way you might install any program and move on.
But *how* you install scientific software turns out to matter for the very thing
this book is about — reproducibility — so it is worth doing right from the first
day.

The problem is that a bioinformatics project depends on many programs (an
aligner, a counter, Snakemake itself), each of a *specific version*, and
different projects need *different, conflicting* versions. Install everything into
one shared system-wide location and the projects collide: upgrading a tool for
one analysis silently breaks another. This is sometimes called "dependency hell."

The standard escape is a **package manager** paired with **virtual
environments**.

> A **package manager** is a program that installs software and, crucially, the
> other software it depends on, at versions known to work together. A **virtual
> environment** (or just "environment") is an isolated, self-contained directory
> of installed software. Different environments can hold different — even
> conflicting — versions of the same tool without interfering, because each
> project activates only its own environment.

In bioinformatics the dominant package manager is **Conda**, because its
**Bioconda**[1] channel packages thousands of biology tools (aligners, samtools,
and so on), not just Python libraries. A "channel" is simply a repository Conda
downloads packages from; you will use two: `conda-forge` (general software) and
`bioconda` (biology tools).

> **Mamba** is a drop-in, much faster reimplementation of Conda; wherever you
> would type `conda`, you can type `mamba` and get the same result sooner.
> Modern Conda installations include it. We use `mamba` below.

Here is the whole setup. First create an environment containing Snakemake:

```bash
# Create an environment named "snakemake" with Snakemake installed in it,
# pulling from the conda-forge and bioconda channels.
$ mamba create -n snakemake -c conda-forge -c bioconda snakemake
```

Then **activate** it — switch your shell into that environment — and confirm the
install worked:

```bash
$ mamba activate snakemake
$ snakemake --version
9.x.x
```

Two habits to build immediately, because they pay off forever:

- **`--version`** tells you exactly which version you are running. When you report
  results or ask for help, this number is the first thing anyone needs.
- **`--help`** prints the full list of command-line options. Snakemake has many;
  no one memorizes them. `snakemake --help` (or skimming the online docs) is the
  normal way to find the flag you need.

> The environment that holds Snakemake itself is separate from the environments
> that hold the *tools each rule runs* — Chapter 8 shows how Snakemake can create
> and manage a dedicated environment per rule, so that every step of a pipeline
> records and pins its own software versions. For now, one environment with
> Snakemake in it is all you need.

## A note on our cluster

Most real RNA-seq work does not happen on your laptop — the files are too large
and the jobs too heavy (recall the scale note in Chapter 1). It happens on a
shared **high-performance computing (HPC) cluster**: a large multi-user machine,
or set of machines, that you log into remotely and submit jobs to. We cover
running Snakemake on the cluster properly in Chapter 9.

For setup purposes, the one thing to know now is that on shared clusters software
is often provided through an **environment module** system (you may see commands
like `module load miniconda`) rather than installed by you from scratch. The
specifics are site-dependent, so rather than reproduce them here — where they
would quickly go stale — consult our group's internal onboarding notes for the
exact `module` commands and where to create your environments on the cluster
filesystem. The Conda/Mamba workflow above is the same once a base Conda is
available; only the first step of *obtaining* Conda differs.

## A place to put a project: the standard layout

A Snakemake project is a *directory* of files, and where you put each kind of file
is governed by a widely used convention.[2] You could, in principle, dump
everything in one folder. You should not — and not for tidiness' sake. A
consistent layout means anyone in the group (including future you) can open any
project and know instantly where the workflow logic lives, where the
configuration lives, and where results will land. Convention reduces the mental
effort of navigating a project to nearly zero.

Here is the layout we use, shown as a directory tree:

```
my-rnaseq-project/
├── config/
│   ├── config.yaml          # parameters and settings for a run
│   └── samples.tsv          # the table of samples to process
├── workflow/
│   ├── Snakefile            # the entry point Snakemake reads first
│   ├── rules/               # rule definitions, split across files
│   ├── envs/                # per-rule conda environment files
│   └── scripts/             # Python/R scripts that rules call
├── resources/               # inputs you download/reuse (genome, annotation)
└── results/                 # everything the pipeline produces
```

The guiding principle behind this split is one you will see again and again:
**separate the things that change from the things that do not.**

- **`workflow/`** holds the *logic* — the rules that define how to transform data.
  This is the code; it is what you commit to version control and share. By
  convention Snakemake looks for `workflow/Snakefile` automatically, so you can
  run `snakemake` from the project root with no arguments pointing at the file.
- **`config/`** holds the *settings* — which samples to run, which genome,
  parameter values. Changing what an analysis does, without changing how it
  works, means editing files here and nothing else. (This separation is the whole
  subject of Chapter 7.)
- **`resources/`** holds large, reusable *inputs* you bring in from outside — a
  reference genome, a gene annotation. These are typically too big to put under
  version control.
- **`results/`** holds *outputs* — everything the pipeline generates. A useful
  consequence of keeping all outputs here is that you can delete the entire
  `results/` directory to start completely fresh, and Snakemake will rebuild it.

> Notice how this layout physically encodes the Chapter 2 model: `resources/` and
> `config/` are the raw inputs at the bottom of the DAG, `workflow/` is the set of
> rules, and `results/` is everything the DAG produces. You are not just
> organizing files; you are laying out the pipeline's anatomy on disk.

You do not need to memorize this or build it by hand right now — Snakemake can
generate a starter scaffold for you, and we will create the directories as we
need them. The point is to recognize the shape when you see it, because every
well-kept project in the group looks like this.

## Where we are going

You now have a workable Python vocabulary, Snakemake installed in a clean
environment, and a mental template for how a project is laid out. That is the
entire toolkit needed to write something real. In Chapter 4 we open an empty
`workflow/Snakefile` and build the smallest complete RNA-seq pipeline that
actually runs — turning the abstract rules and DAG of Chapter 2 into files you can
watch appear on disk.

---

## References

1. Grüning B, Dale R, Sjödin A, et al. Bioconda: sustainable and comprehensive
   software distribution for the life sciences. *Nature Methods*.
   2018;15(7):475–476.
2. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33. (See the "Distribution and
   reproducibility" section on recommended project structure.)
