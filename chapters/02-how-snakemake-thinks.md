# Chapter 2 — How Snakemake Thinks

> *Files, rules, and the dependency graph — the one mental model the rest of the
> book builds on.*

Chapter 1 argued that a pipeline run as a shell script is fragile because the
script has no *model* of the work: it is just a list of commands executed top to
bottom. Snakemake's power comes from having such a model. Before we write
anything runnable (that is Chapter 4), this chapter installs the model in your
head. Almost everything else in the book is a detail hanging off the ideas here,
so it is worth going slowly.

## Two ways to describe work

There are two fundamentally different ways to tell a computer how to carry out a
multi-step task.

The first is **imperative**: you spell out the steps, in order, and the computer
does exactly that, in exactly that sequence. The Bash script from Chapter 1 is
imperative — "trim, then align, then sort, then count." This is how most
programming you have done so far works, and it is intuitive. Its weakness is that
*the order is your responsibility*, and the computer understands nothing about
*why* the steps are in that order.

The second is **declarative**: instead of listing steps in sequence, you describe
the *relationships* between things — what depends on what — and let the system
work out the order itself. You declare the goal and the rules of the game; the
system plans the moves.

> A homely analogy: an imperative recipe says "preheat the oven, then mix the
> dry ingredients, then mix the wet, then combine, then bake." A declarative
> description says "the cake requires batter and a hot oven; batter requires
> combined wet and dry mixtures." From the second description a sufficiently
> clever cook can *derive* the order — and can notice that heating the oven and
> mixing the batter can happen at the same time. Snakemake is that clever cook.

Snakemake is declarative, and the specific flavor of declarative it uses is
**file-centric**. This is the first big idea:

> **In Snakemake, the things that exist are files, and the way you describe work
> is by saying which files can be produced from which other files.** You do not
> say "run the aligner." You say "this aligned file can be made from that trimmed
> file, like so." The aligner runs as a *consequence* of your having asked for the
> aligned file.

This file-centric, declarative model is inherited from **Make**, a venerable tool
from the world of compiling software, where the same logic governs turning source
code files into program files.[1] Snakemake adapts it for data analysis and adds
the Python on top.[2]

## The rule: Snakemake's atomic unit

The unit you describe work with is the **rule**. A rule is a recipe for producing
output file(s) from input file(s). At minimum, a rule has three parts:

- **`input`** — the file(s) the rule needs before it can run.
- **`output`** — the file(s) the rule produces.
- a **recipe** — the command(s) that turn input into output, written either as a
  `shell` command or as a `script` (Python, R, …).

Here is a single rule for the alignment step of our RNA-seq pipeline. Read it for
its anatomy, not its syntax details — we dissect syntax in Chapter 4:

```python
rule align:
    input:
        reads="trimmed/sampleA.fq.gz",
        index="genome/index",
    output:
        "aligned/sampleA.bam",
    shell:
        "hisat2 -x {input.index} -U {input.reads} | samtools sort -o {output}"
```

In English: *"There is a rule called `align`. To make the file
`aligned/sampleA.bam`, I need the files `trimmed/sampleA.fq.gz` and
`genome/index`. Once I have them, produce the output by running this command."*

Notice what is **absent**. The rule says nothing about *when* it should run, or
whether trimming has happened yet, or what happens after alignment. It describes
only a local fact: *this output, from these inputs, via this command.* A rule is a
self-contained claim about how one kind of file comes to exist. The genius of the
model is that a complete pipeline emerges from a pile of such local claims,
without anyone ever writing down the global order.

> The `{input.index}`, `{input.reads}`, and `{output}` in the command are
> **placeholders**: Snakemake substitutes the actual filenames in before running
> the command. This keeps the recipe from repeating the filenames and, as we will
> see in Chapter 5, lets a single rule serve many files. For now just read them
> as "the input" and "the output."

## Reasoning backwards from a target

If rules only describe local input→output facts, how does a whole pipeline run?
Through a beautifully simple piece of reasoning that runs **backwards**.

You start by naming a file you *want* — a **target**. Say you ask Snakemake for
`counts/sampleA.txt`, the final per-gene count table for sample A. Snakemake then
reasons like this:

1. *Which rule produces `counts/sampleA.txt`?* The `count` rule does. To run it,
   I need its input, `aligned/sampleA.bam`. Does that file exist yet?
2. *No. Which rule produces `aligned/sampleA.bam`?* The `align` rule. Its input
   is `trimmed/sampleA.fq.gz`. Does *that* exist?
3. *No. Which rule produces `trimmed/sampleA.fq.gz`?* The `trim` rule. Its input
   is `raw/sampleA.fastq.gz`. Does that exist?
4. *Yes* — that is the raw data you started with. So the chain bottoms out.

Having traced the chain backwards from the goal to the raw inputs, Snakemake now
knows it must run the rules **forwards**: trim, then align, then count. You never
told it that order. You told it a target and a handful of local input→output
facts, and it *derived* the order by matching each rule's output to the next
rule's input.

> This backward-chaining is exactly why adding a step in the middle of a pipeline
> is painless in Snakemake and treacherous in a shell script. You write one new
> rule whose output is some existing rule's input, and the chain re-routes
> through it automatically. Nothing about ordering needs to be edited, because you
> never wrote the ordering down in the first place.

## The dependency graph (the DAG)

The "chain" above is the simplest possible shape: a straight line. Real pipelines
branch and merge — quality-control reports for every sample, a genome index built
once and reused by every alignment, a final summary that gathers counts from all
samples at once. The general structure formed by rules connected through their
files is a graph, and it has a precise name: a **directed acyclic graph**, or
**DAG**.

Let us unpack that term, because it is used constantly and rewards understanding:

- **Graph** — a collection of *nodes* connected by *edges*. Here, each node is a
  job (a rule applied to particular files) and each edge is a dependency (file X
  is produced here and consumed there).
- **Directed** — every edge has a direction: data flows from the rule that
  *produces* a file to the rule that *consumes* it. Trimming feeds alignment, not
  the other way around.
- **Acyclic** — there are no cycles; you can never follow the arrows and arrive
  back where you started.

That last property is not a technicality — it is what makes the whole scheme
work. Suppose alignment depended on counting *and* counting depended on
alignment. Then to run either you would first need to have run the other: an
impossible chicken-and-egg loop. Snakemake would have nothing to start from. A
DAG forbids exactly this. Because the graph is acyclic, there is always at least
one node with no unmet dependencies (some raw input you already have), so there is
always somewhere to begin, and the work is guaranteed to finish.

Here is the DAG for processing three samples through our four-step pipeline. Read
top to bottom as the direction of data flow:

```
   raw/A.fastq      raw/B.fastq      raw/C.fastq
       │                │                │
     trim             trim             trim          genome/index
       │                │                │            ╱   │   ╲
     align ◄────────────┼────────────────┼───────────╯    │    ╲
       │              align ◄────────────┼─────────────────╯     ╲
       │                │              align ◄────────────────────╯
     count            count            count
       │                │                │
       └────────────────┼────────────────┘
                        │
                    summarize
                        │
                  results/report.html
```

A few things this picture makes visible. The genome index is built once and feeds
*all three* alignments — that is a node with several outgoing edges. The three
samples' branches are entirely independent of one another until they converge at
`summarize` — that is the branch-and-merge shape real pipelines have. And every
arrow points downward: the graph is directed and acyclic.

> When people say a pipeline "is a DAG," this is what they mean, and it is true of
> *every* workflow manager, not just Snakemake — Nextflow, WDL, and CWL all build
> a DAG too. The DAG is the universal idea; the syntax for expressing it is what
> differs between tools. Learn to see the DAG and you have learned the portable
> skill.

## What the model buys you

The payoff of describing your pipeline as a DAG of file-producing rules is that
Snakemake can now *reason* about your work, and several of the headaches from
Chapter 1 simply evaporate as automatic consequences:

- **Correct ordering, for free.** The order of execution is computed from the
  graph, not written by you, so it cannot be wrong and never needs editing when
  you add or rearrange steps.

- **Only necessary work gets done.** Snakemake compares the timestamp of each
  output against its inputs. If an output already exists and is newer than
  everything it depends on, that node is *already satisfied* and is skipped.
  Change a single input file and only the nodes downstream of it — the ones whose
  inputs are now newer than their outputs — are recomputed. This "build only what
  is out of date" logic is, again, inherited from Make.[1] (Snakemake actually
  tracks more than timestamps — changed code and parameters can trigger re-runs
  too — but timestamps are the core idea; the details are Chapter 6.)

- **Parallelism, for free.** Two nodes with no path between them in the DAG are
  independent and may run at the same time. Snakemake sees this directly in the
  graph: in the figure above, all three `trim` jobs are mutually independent, so
  given three free CPUs Snakemake runs them simultaneously, with no concurrency
  code from you. You will see this in action in Chapter 9.

- **Clean recovery and provenance.** Because the system knows the intended output
  of every job, an interrupted job's half-written file can be recognized and
  discarded rather than mistaken for a finished result, and the graph itself is a
  precise record of how every file was made. We return to both in Chapters 10
  and 11.

Every one of these is the same underlying move: *because you described
relationships instead of dictating steps, the system understands your pipeline
well enough to manage it for you.* That is the entire reason workflow managers
exist, and now you can see exactly how Snakemake delivers it.

## Where we are going

You now hold the core mental model: **files** are the currency, a **rule** is a
local recipe mapping input files to output files, and the rules together form a
**DAG** that Snakemake traverses backwards from a target to plan the work and
forwards to do it. Hold onto this picture — when a pipeline later does something
surprising, the explanation almost always lives in the DAG.

In Chapter 3 we get your environment set up so you can run Snakemake, and in
Chapter 4 we turn the fragments above into a real, executable Snakefile and watch
this model spring to life on actual files.

---

## References

1. Feldman SI. Make — a program for maintaining computer programs. *Software:
   Practice and Experience*. 1979;9(4):255–265.
2. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33.
