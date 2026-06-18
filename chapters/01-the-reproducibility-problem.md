# Chapter 1 — The Reproducibility Problem

> *Why workflow managers exist, and why our group uses one.*

Before you write a single line of Snakemake, it is worth understanding the
problem it was built to solve. Snakemake is not a programming language, an
aligner, or a statistics package; it does no biology of its own. It is a tool
for *organizing* the other tools you will use — and the case for spending a week
learning it only makes sense once you have felt the pain of working without it.
This chapter is about that pain.

## What a pipeline is

Almost nothing in bioinformatics is done in a single step. Consider a task you
will see constantly in our group: measuring **gene expression** with **RNA-seq**.

> **RNA-seq** (RNA sequencing) is a technique for measuring which genes are
> active in a sample of cells, and how strongly. Recall from molecular biology
> that a gene is *transcribed* into RNA before it is *translated* into protein;
> the more RNA copies (transcripts) of a gene a cell makes, the more "expressed"
> that gene is said to be. RNA-seq works by converting the RNA in a sample into
> DNA, sequencing millions of short fragments of it, and counting how many
> fragments came from each gene.[1,2] The sequencing machine does not hand you
> gene counts, though — it hands you a file of millions of short text strings
> called **reads**, each a few dozen to a few hundred letters of A, C, G, and T.
> Turning reads into a biologically meaningful answer is a computational job.

That computational job is a sequence of distinct programs, each consuming the
output of the last:

1. **Quality control** — inspect the raw reads for problems (low-quality
   bases, leftover laboratory adapter sequences) and trim them away.
2. **Alignment** (or **pseudoalignment**) — figure out which gene or location in
   the genome each read most likely came from.
3. **Quantification** — count, per gene, how many reads were assigned to it.
4. **Summarization** — collect the per-sample counts into a single table and a
   report you can actually look at.

This chain of steps is a **pipeline** (also called a **workflow**): an ordered
series of data transformations that turns raw input into a finished result. Each
step is usually a separate, specialized program — written by a different group,
with its own options and quirks — and each step produces **intermediate files**
that exist only to feed the next step. A real RNA-seq study does not run this
pipeline once. It runs it for every **sample** in the experiment — and a study
might have dozens or hundreds of samples (a sample is one sequenced biological
specimen, say "liver tissue from mouse #7").

> A note on scale, because it shapes everything that follows. A single RNA-seq
> sample's raw read file is routinely several gigabytes; alignment of one sample
> can take tens of minutes to hours of computer time and many gigabytes of
> memory. A pipeline is therefore not something you can simply re-run from
> scratch whenever you are unsure of its state — doing so might cost you a day
> and a chunk of a shared computer that your labmates also need.

## The pile of shell scripts

The natural first instinct — and the way nearly everyone starts — is to run each
program by hand at the **command line**, or to paste the commands into a **shell
script** so you can run them as a batch.

> The **command line** (or **shell**) is the text interface where you type
> commands to run programs, e.g. `trim_galore reads.fastq`. A **shell script**
> is just a text file containing a list of such commands that you can execute
> all at once. **Bash** is the most common shell language on the Linux systems
> used for bioinformatics.

Here is what that looks like for our RNA-seq example, written as a Bash script.
Read it not for the specific tools (you do not need to know them yet) but for
its *shape*:

```bash
#!/bin/bash
# run_rnaseq.sh — process one sample, the "obvious" way

# 1. Trim adapters and low-quality bases
trim_galore --output_dir trimmed/ raw/sampleA.fastq.gz

# 2. Align the trimmed reads to the reference genome
hisat2 -x genome/index -U trimmed/sampleA_trimmed.fq.gz -S aligned/sampleA.sam

# 3. Convert to a compressed, sorted format
samtools sort -o aligned/sampleA.sorted.bam aligned/sampleA.sam

# 4. Count reads per gene
featureCounts -a genome/annotation.gtf -o counts/sampleA.txt aligned/sampleA.sorted.bam
```

For a single sample, run once, on the machine where you wrote it, this works. The
trouble begins the moment reality intrudes — and it always does. Consider what
happens in each of these entirely ordinary situations:

- **One step fails halfway.** The alignment runs out of memory and dies after
  writing a partial `sampleA.sam` file. You fix the memory problem and re-run the
  script. But the script has no idea step 2 failed last time — it dutifully
  re-runs the trimming (step 1) from scratch, wasting ten minutes, and then, if
  you are unlucky, the downstream steps happily consume the *truncated* SAM file
  from the failed run and produce silently wrong counts.

- **You add more samples.** The experiment grows from one sample to sixty. Do you
  copy-paste the four commands sixty times, hand-editing `sampleA` to `sampleB`,
  `sampleC`, and so on? Every copy is a chance to fat-finger a filename and, say,
  align sample B's reads but count sample A's — an error that produces a
  plausible-looking number and no warning at all.

- **One input changes.** A labmate regenerates the genome annotation file with a
  newer version. Which results are now stale and must be recomputed? With a
  script, the honest answer is "all of them," because the script has no model of
  what depends on what. So you re-run everything, for every sample, and lose a
  day.

- **You want to use more than one CPU.** The sixty samples are independent and
  your machine has thirty-two cores, so in principle you could process many
  samples at once. A linear shell script runs them strictly one after another.
  Parallelizing it by hand means juggling background jobs and waiting on them —
  fiddly, error-prone plumbing that has nothing to do with biology.

- **Someone else tries to run it** — including *you*, six months from now. The
  script assumes `hisat2` is installed, and a specific version of it; it assumes
  the genome index lives at `genome/index`; it assumes a dozen things about the
  machine it was born on. On a different computer it fails in a cascade of
  confusing errors, or — worse — it runs to completion using a *different
  version* of a tool and produces subtly different numbers.

None of these are exotic. They are the daily texture of computational research.
The shell-script approach does not so much solve the problem as defer it, and the
deferred cost is paid in lost days, silent errors, and results nobody is quite
sure they can trust.

## Reproducibility is a scientific requirement

That last phrase — "results nobody is quite sure they can trust" — is the heart
of the matter, and it is not merely an engineering annoyance. It is a scientific
problem.

A result is **reproducible** if someone else (or you, later) can take the same
data and the same described methods and obtain the same answer. This is a
foundational expectation of science, and the life sciences have spent the last
decade reckoning with how often it fails: in one widely cited survey, more than
70% of researchers reported having tried and failed to reproduce another
scientist's experiments.[3] For *computational* work specifically, the obstacle
is rarely the unavailability of a microscope or a reagent — it is that the exact
sequence of programs, versions, parameters, and files that produced a figure was
never written down precisely enough to repeat.[4]

Closely related is **provenance**: a complete, trustworthy record of *how* a
given result was produced — which inputs, which tools and versions, which
parameters, in which order. A shell script is a partial provenance record at
best, and it goes out of date the instant someone runs a command by hand that
they forget to add back to the file.

The point worth internalizing now is this: **reproducibility and provenance are
not optional polish you add at the end of a project.** They are properties your
analysis either has or does not have from the start, and they are far cheaper to
build in than to reconstruct months later when a reviewer asks how, exactly, you
made Figure 3. A good workflow manager gives them to you almost for free, as a
side effect of how it works.

## What a workflow manager does

A **workflow management system** is a tool whose entire job is to run pipelines
correctly. Rather than executing a fixed list of commands top to bottom, you
*describe* the steps and how they connect — and the system works out the rest.
The good ones share a set of capabilities that map directly onto the failures we
just catalogued:

- **It tracks dependencies.** A **dependency** is simply the relationship "step B
  needs the output of step A before it can run." Given these relationships, the
  system computes the correct order to run things — you never specify it by hand.

- **It only does necessary work.** If a result already exists and nothing it
  depends on has changed, the system skips it. Change one input, and it recomputes
  exactly the things downstream of that input and nothing else. The "which results
  are now stale?" question is answered automatically.

- **It recovers from failure cleanly.** A step that dies does not leave a
  half-written file lying around to poison the next run; on restart, the system
  resumes from where it stopped rather than from the beginning.

- **It parallelizes for free.** Because the system knows which steps are
  independent, it can run them simultaneously across as many CPUs — or cluster
  nodes — as you give it, with no extra effort from you.

- **It records provenance and pins environments.** A good workflow manager can
  attach a specific software version to each step and produce a report of what it
  did, turning reproducibility from an aspiration into the default behavior.

## Why Snakemake

Several mature workflow managers exist in bioinformatics. **Make**, borrowed from
the world of compiling software, pioneered the file-and-dependency model that
Snakemake builds on. **Nextflow**, **WDL** (with the Cromwell engine), and
**CWL** (the Common Workflow Language) are all widely used in genomics, each with
real strengths, particularly at very large institutional scale.[5]

This book is about **Snakemake**[6,7] for two reasons, one general and one
specific to our group.

The general reason is that Snakemake hits a sweet spot for an academic lab: it is
expressive enough for production genomics yet simple enough that a new student can
read an existing workflow and understand it within a chapter or two. Its design
deliberately echoes Make's elegant file-based model, while removing Make's
notorious rough edges.[6]

The specific reason is that **Snakemake is built on Python.** A Snakemake
workflow file *is* a Python file with some extra syntax, which means everything
you already know — and everything our group already does — in Python is available
inside your pipelines: pandas for reading sample tables, ordinary functions and
loops for generating inputs, the scientific Python ecosystem for analysis steps.
Because so much of our day-to-day work is in Python, choosing a Python-native
workflow manager means there is one fewer language to context-switch into, and
the boundary between "my analysis code" and "my pipeline code" stays thin. This
was the deciding factor in choosing Snakemake over Nextflow, whose workflow
language is its own Groovy-based dialect.

> This is not a claim that Snakemake is universally "best" — Nextflow in
> particular is excellent and you will certainly encounter it. It is a claim that
> Snakemake is the right fit *for us*, and the concepts you will learn here —
> rules, dependencies, the dependency graph — transfer directly to every other
> workflow manager, because they all solve the same problem in the same shape.

## Where we are going

You now have the *why*. The rest of the book builds the *how*, starting in
Chapter 2 with the single most important idea in Snakemake: that you describe a
pipeline by declaring **what files should exist and how to make each one**, and
let Snakemake figure out the order, the re-runs, and the parallelism on your
behalf. We will return, again and again, to the same small RNA-seq example begun
above — by the end of the book it will be a real, runnable, reproducible pipeline
of the kind you will use in your research.

---

## References

1. Wang Z, Gerstein M, Snyder M. RNA-Seq: a revolutionary tool for
   transcriptomics. *Nature Reviews Genetics*. 2009;10(1):57–63.
2. Conesa A, Madrigal P, Tarazona S, et al. A survey of best practices for
   RNA-seq data analysis. *Genome Biology*. 2016;17:13.
3. Baker M. 1,500 scientists lift the lid on reproducibility. *Nature*.
   2016;533(7604):452–454.
4. Sandve GK, Nekrutenko A, Taylor J, Hovig E. Ten simple rules for reproducible
   computational research. *PLoS Computational Biology*. 2013;9(10):e1003285.
5. Wratten L, Wilm A, Göke J. Reproducible, scalable, and shareable analysis
   pipelines with bioinformatics workflow managers. *Nature Methods*.
   2021;18(10):1161–1168.
6. Köster J, Rahmann S. Snakemake — a scalable bioinformatics workflow engine.
   *Bioinformatics*. 2012;28(19):2520–2522.
7. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33.
