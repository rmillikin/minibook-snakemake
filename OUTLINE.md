# Snakemake: A Minibook for New Bioinformaticians

*A concept-first orientation to reproducible workflows for incoming graduate students.*

## About this minibook

This minibook introduces **Snakemake**, the workflow management system our group
uses to build analysis pipelines. It is written for new graduate students who
have a semester or two of undergraduate molecular/cell biology and chemistry and
roughly a year of undergraduate computer science (so: comfortable with the idea
of a programming language, a function, a file, and a command line, but not
assumed to know Python deeply, the shell intimately, or anything about workflow
managers).

The book is **concept-first**. Each chapter leads with *why* a feature exists and
the problem it solves, then uses short code snippets as instructional tools
rather than as a copy-paste cookbook. Jargon — both computational and
biological — is defined at first use. By the end, a reader should be able to
read, run, modify, and reason about the pipelines they will encounter in the
lab, and to write new ones.

### How to read it

- **Chapters 1–6** are the core. Read them in order; they build a single mental
  model and a single running example.
- **Chapters 7–11** are the "second semester" — the practices that separate a
  toy script from a pipeline that a lab can trust and share.
- **Chapter 12** is a fully worked, realistic example that ties everything
  together.
- **Appendices** are reference material to return to, not to read straight
  through.

### Conventions

- Code blocks beginning with `$` are shell commands; everything else in a code
  block is the contents of a file (usually a `Snakefile` or a config file).
- **Bold** marks a term being defined for the first time.
- Margin-style asides (blockquotes) flag common mistakes and "in our lab we
  do X" notes.

---

## Structure

Chapters live in `chapters/` as individual Markdown files, numbered so they sort
in reading order (e.g. `01-the-reproducibility-problem.md`).

---

## Chapter outline

### Chapter 1 — The Reproducibility Problem (Why Workflow Managers Exist)
*File: `chapters/01-the-reproducibility-problem.md`*

The motivating chapter. No Snakemake yet — first establish the pain it removes.

- What a bioinformatics **analysis pipeline** is: a sequence of programs that
  transform raw data (e.g. sequencing reads) into results (e.g. a list of
  variants or a table of gene expression).
- The "pile of shell scripts" failure mode: manual step ordering, half-finished
  runs, re-running everything after one input changes, "it worked on my
  machine."
- **Reproducibility** and **provenance** as scientific requirements, not just
  software niceties; brief connection to the replication conversation in the
  life sciences.
- What a **workflow management system** is and the landscape (Make, Nextflow,
  WDL/Cromwell, CWL) — why this book is about Snakemake, including that it is a
  Python-based tool, which fits a group that already works heavily in Python.
- *Concepts introduced:* pipeline, reproducibility, provenance, dependency,
  intermediate file.
- *Code:* a deliberately fragile `bash` script (an RNA-seq quantification run)
  as a "before" picture.
- *References:* reproducibility in computational biology; the Snakemake paper;
  RNA-seq best-practices literature.

### Chapter 2 — How Snakemake Thinks: Files, Rules, and the DAG
*File: `chapters/02-how-snakemake-thinks.md`*

The single most important conceptual chapter: Snakemake's mental model.

- Snakemake is **declarative** and **file-centric**: you describe *what files
  should exist and how to make them*, not the order to run things in.
- The **rule** as the atomic unit: `input`, `output`, and a `shell`/`script`
  recipe.
- Reasoning *backwards from outputs*: you ask for a target file, Snakemake
  figures out what must run.
- The **dependency graph** (a **DAG** — directed acyclic graph): how outputs of
  one rule become inputs of the next, and why "acyclic" matters.
- Why this buys you: automatic step ordering, only-rebuild-what-changed,
  parallelism for free.
- *Concepts introduced:* declarative vs. imperative, rule, DAG, target,
  build-on-change.
- *Code:* a two-rule Snakefile shown only to illustrate the model (full
  walkthrough comes in Ch. 4).

### Chapter 3 — Getting Set Up
*File: `chapters/03-getting-set-up.md`*

Practical environment setup so the reader can follow along.

- Snakemake is a **Python** package; the minimum Python you need to read this
  book (just enough syntax — strings, lists, dicts, f-strings).
- **Conda**/**mamba** as package managers; creating an isolated **environment**
  and why isolation matters.
- Installing Snakemake; verifying the install; the `--version` and `--help`
  habit.
- A recommended **project directory layout** (`workflow/`, `config/`,
  `results/`, `resources/`) and why convention reduces cognitive load.
- *Concepts introduced:* package manager, virtual environment, project scaffold.
- *Code:* `conda`/`mamba` install commands; `tree` of a starter project.
- *Aside:* our lab's shared cluster modules / how we actually install it here.

### Chapter 4 — Your First Workflow
*File: `chapters/04-your-first-workflow.md`*

Hands-on. Build and run the smallest meaningful pipeline end to end.

- Anatomy of a `Snakefile`: writing a single rule, then a second that consumes
  the first's output.
- Running it: `snakemake --cores 1`, reading the log, finding the outputs.
- The **default target** (first rule) and the `all` rule convention.
- A **dry run** (`-n`) and printing reasons (`-p`, `-r`): seeing the plan before
  doing the work.
- What happens on a second run (nothing!) and what happens when you `touch` an
  input.
- *Concepts introduced:* Snakefile, default/target rule, `all` rule, dry run,
  timestamp-based rebuilds.
- *Code:* a complete, runnable toy pipeline (e.g. download/transform a small
  text or FASTA file) carried forward in later chapters.

### Chapter 5 — Wildcards: One Rule for Many Samples
*File: `chapters/05-wildcards.md`*

The conceptual leap that makes Snakemake scale to real data.

- The problem: a study has dozens of **samples**; you do not want dozens of
  near-identical rules.
- **Wildcards** as pattern variables in input/output paths; how Snakemake
  *infers* wildcard values by matching a requested output filename.
- The `{sample}` pattern; accessing wildcards in the shell command via
  `{wildcards.sample}`.
- Ambiguity and constraints (`wildcard_constraints`) — why a wildcard can match
  "too much" and how to rein it in.
- `expand()` to generate lists of target files from sample lists.
- *Concepts introduced:* sample, wildcard, pattern matching, `expand()`,
  wildcard constraints.
- *Code:* generalizing the Ch. 4 rule to process an arbitrary set of samples.

### Chapter 6 — Building the Graph: Dependencies, Targets, and Inspection
*File: `chapters/06-the-dag.md`*

Deepen the DAG idea now that wildcards make it nontrivial.

- How Snakemake **chains rules** by matching one rule's output pattern to
  another's input.
- Defining the final targets you actually want (the `all` rule revisited) and
  letting the graph fill itself in.
- **Visualizing** the workflow: `--dag` and `--rulegraph` piped to Graphviz;
  reading these diagrams.
- Re-run logic: what triggers a rule to re-execute (changed input, changed code,
  changed params) and **rerun triggers**.
- *Concepts introduced:* rule chaining, DAG visualization, rerun triggers.
- *Code:* generating and interpreting a DAG image for the running example.

### Chapter 7 — Configuration and Sample Sheets
*File: `chapters/07-configuration-and-samples.md`*

Separating the *code* of a pipeline from the *data and parameters* it runs on.

- Why hard-coding paths and parameters is a maintenance and reproducibility
  trap.
- The **config file** (`config.yaml`): a quick primer on **YAML** syntax and the
  `config` dictionary in Snakemake.
- The **sample sheet** (a tab/comma-separated table of samples and metadata),
  read with **pandas**; a one-paragraph intro to a pandas DataFrame for readers
  who have not seen it.
- Passing parameters to rules with the `params` directive; keeping numbers and
  flags out of the shell string.
- *Concepts introduced:* config file, YAML, sample sheet, `params`, DataFrame.
- *Code:* refactoring the running example to be driven entirely by
  `config.yaml` + `samples.tsv`.

### Chapter 8 — Reproducible Software Environments
*File: `chapters/08-software-environments.md`*

Making "it worked on my machine" go away — the other half of reproducibility.

- The problem: results can depend on the *version* of a tool (aligner, caller,
  library).
- Per-rule **conda environments** (`conda:` directive + an environment YAML);
  pinning versions.
- **Containers** (Docker/Apptainer/Singularity) and the `container:` directive;
  when containers beat conda (and vice versa) on a shared cluster.
- A note on **wrappers** (the Snakemake Wrapper Repository) as vetted, reusable
  rule bodies for common bioinformatics tools.
- *Concepts introduced:* version pinning, conda env file, container/image,
  wrapper.
- *Code:* attaching an environment file to a rule and running with
  `--use-conda` / `--software-deployment-method`.

### Chapter 9 — Scaling Up: Threads, Resources, Clusters, and the Cloud
*File: `chapters/09-scaling-up.md`*

Going from a laptop to the resources real genomics demands.

- **Parallelism**: `--cores`/`--jobs` and how Snakemake schedules independent
  jobs across the DAG.
- Per-rule `threads` and how tools are told to use them.
- The `resources` directive: declaring memory, time, and other limits.
- **HPC clusters** and the **job scheduler** (e.g. SLURM): the **executor**
  concept and profiles so the *same* workflow runs locally or on a cluster
  without edits.
- A short note on cloud execution as the same idea with a different backend.
- *Concepts introduced:* core/thread/job, resources, scheduler, executor,
  profile.
- *Code:* a rule declaring `threads`/`resources`; a minimal cluster profile.

### Chapter 10 — Robustness, Logging, and Debugging
*File: `chapters/10-debugging.md`*

What to do when (not if) a pipeline breaks.

- Reading Snakemake's error output: which job failed, the log file, the exact
  shell command it ran.
- The `log:` directive and why every nontrivial rule should have one.
- **Atomic** outputs: `temp()`, `protected()`, and why partially-written files
  on failure are dangerous (and how Snakemake guards against them).
- Common failure modes: wildcard ambiguity, missing inputs, "AmbiguousRule",
  out-of-memory, non-zero exit codes.
- Resuming after failure; `--rerun-incomplete`; targeted re-runs.
- *Concepts introduced:* log directive, `temp()`/`protected()`, atomic write,
  incomplete files.
- *Code:* adding logging to the running example and walking through a staged
  failure.

### Chapter 11 — Organizing Larger Projects
*File: `chapters/11-organizing-projects.md`*

From one Snakefile to a maintainable, shareable codebase.

- Splitting rules across files with `include`; organizing `workflow/rules/`.
- **Modules** and reuse across projects (`module` / `use rule`).
- Reproducibility extras: **reports** (`--report`), **benchmarking**
  (`benchmark:`), and recording software/provenance.
- Lab conventions and a checklist: naming, where results go, what to commit to
  **version control** (and what not to — large data).
- Where to find help: docs, the wrapper repo, the community.
- *Concepts introduced:* include, module, report, benchmark, version control
  hygiene for pipelines.
- *Code:* a modularized layout of the running example.

### Chapter 12 — A Worked Example, End to End
*File: `chapters/12-worked-example.md`*

Synthesis. Build a small but realistic pipeline using everything so far.

- A compact, recognizable **RNA-seq** workflow (quality control → alignment or
  pseudoalignment → quantification → a summary table/report), at a scale a
  reader can actually run.
- Driven by a config file and sample sheet, with per-rule conda environments,
  logging, and a generated report — i.e. "how we actually write pipelines."
- A guided tour of each rule and how the DAG connects them, then running it and
  inspecting outputs.
- Suggested extensions as exercises.
- *No new concepts* — this chapter consolidates Chapters 1–11.

---

## Appendices

### Appendix A — Directive & Command Cheat Sheet
*File: `chapters/A1-cheat-sheet.md`*

One-page references: common rule directives (`input`, `output`, `params`,
`threads`, `resources`, `log`, `conda`, `container`, `shell`, `script`,
`benchmark`) and common command-line flags (`-n`, `-p`, `-r`, `--cores`,
`--use-conda`, `--dag`, `--rerun-incomplete`, `--report`).

### Appendix B — A Minimal Python & Shell Refresher
*File: `chapters/A2-python-shell-refresher.md`*

The small amount of Python (lists, dicts, f-strings, functions) and shell
(paths, pipes, redirection, exit codes) the book assumes, for readers who want a
refresher in one place.

### Appendix C — Glossary
*File: `chapters/A3-glossary.md`*

Alphabetical definitions of every bolded term, both computational (DAG, rule,
wildcard, executor, container) and biological (sample, FASTA/FASTQ, read,
alignment) used in the book.

### Appendix D — Further Reading & References
*File: `chapters/A4-references.md`*

Consolidated, numbered reference list (Snakemake papers and docs, reproducibility
literature, tool documentation) plus pointers to the official tutorial and the
Snakemake Wrapper Repository.

---

## Design notes (for authors of this book)

- **One running example** — a small **RNA-seq** quantification pipeline —
  threads through Chapters 4–12 so concepts accrete rather than reset each
  chapter.
- **Define jargon at first use** and again in the glossary; assume bio basics
  but not sequencing-specific vocabulary, and assume CS basics but not Python
  fluency.
- **Code is illustrative, not exhaustive** — snippets should be minimal and
  runnable, with prose carrying the conceptual weight.
- **Reference where it helps the reader**, especially for claims about
  reproducibility and for pointing to authoritative docs; do not over-cite
  routine statements.
