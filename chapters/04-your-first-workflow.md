# Chapter 4 — Your First Workflow

> *From the abstract model to files you can watch appear on disk.*

This is the chapter where Snakemake stops being a diagram and becomes something
you run. We will write the smallest complete pipeline that is still recognizably
RNA-seq, run it, and then poke at it to watch the Chapter 2 model — rules, the
DAG, build-only-what-changed — behave exactly as promised. Everything here uses
only ordinary command-line utilities, so you can run it on a laptop with nothing
installed but Snakemake. The real aligners and counters arrive later; right now
the goal is the *mechanics*, not the biology.

## The data: a word on FASTQ

The raw output of a sequencing machine, and the input to our pipeline, is a
**FASTQ** file. You will handle thousands of these, so meet the format now.

> A **FASTQ** file stores sequencing **reads** along with a quality score for each
> base. It uses exactly **four lines per read**:
>
> ```
> @read1                 ← line 1: an identifier, starting with "@"
> GATCACAGGT             ← line 2: the sequence of bases (the read itself)
> +                      ← line 3: a separator, just "+"
> IIIIIHHHHH             ← line 4: a quality score for each base, as characters
> ```
>
> The four-lines-per-read rule is the key fact: it means the number of reads in a
> file is simply its line count divided by four. We will use exactly that to count
> reads.

Make a working directory with a tiny FASTQ file in it so you can follow along.
Create `data/sampleA.fastq` with two reads (eight lines):

```
@read1
GATCACAGGT
+
IIIIIHHHHH
@read2
TTACGGATCC
+
IIIIIIIIII
```

## Anatomy of a Snakefile

The file Snakemake reads is called, by convention, the **Snakefile**. Recall from
Chapter 3 that Snakemake looks for `workflow/Snakefile` automatically — but for a
first run a plain `Snakefile` in your current directory works just as well, so
create one there.

We will build a two-step pipeline: **count** the reads in each sample, then write
a small **report** summarizing them. Here is the first rule:

```python
rule count:
    input:
        "data/sampleA.fastq"
    output:
        "results/sampleA.count.txt"
    shell:
        "echo $(( $(wc -l < {input}) / 4 )) > {output}"
```

Read this against the anatomy from Chapter 2. The rule is named `count`. Its
`input` is the FASTQ file; its `output` is a file that will hold the read count;
its recipe is a `shell` command. The command counts the file's lines
(`wc -l`), divides by four, and writes the result to the output. The
`{input}` and `{output}` placeholders are filled in by Snakemake with the real
filenames before the command runs — so you write the filename once, in the
`input`/`output` block, and never repeat it in the recipe.

Now the second rule, which **consumes the first rule's output**:

```python
rule report:
    input:
        "results/sampleA.count.txt"
    output:
        "results/report.txt"
    shell:
        "echo 'sampleA: '$(cat {input})' reads' > {output}"
```

This is the entire mechanism behind the DAG, sitting in plain sight: the `input`
of `report` is exactly the `output` of `count`. Neither rule mentions the other by
name, and nothing states that counting comes before reporting. But because the
files match, Snakemake can chain them — precisely the backward-chaining reasoning
from Chapter 2.

## The default target and the `all` rule

Put both rules in your `Snakefile`, `count` first, and run Snakemake, telling it
it may use one CPU core:

```bash
$ snakemake --cores 1
```

Snakemake must decide *what to build*. Unless you tell it otherwise, it builds the
output of the **first rule in the file** — this is the **default target**. Here
the first rule is `count`, so this command builds only `results/sampleA.count.txt`
and stops. The `report` rule never runs, because nothing asked for its output.

That is a clumsy default to rely on, and it leads to the single most important
convention in Snakemake. Almost every real workflow opens with a rule named
**`all`** that has no recipe and no output — only `input`s listing the *final*
files you want. Because it sits first, it becomes the default target, and asking
for its inputs pulls the whole pipeline through the DAG. Add this rule to the very
top of your Snakefile:

```python
rule all:
    input:
        "results/report.txt"
```

Now `rule all` is the default target. Its input is the pipeline's final product,
`results/report.txt`. To produce that, Snakemake needs the `report` rule; to run
`report` it needs `results/sampleA.count.txt`; to get that it runs `count`. The
`all` rule is not a real processing step — it is a *to-do list of final
outputs* that gives Snakemake a single, stable thing to aim at.

> An `all` rule with no `output` and no `shell` looks strange at first. The trick
> is that a rule's job is to make its *outputs* exist, but a rule can also simply
> *demand* that its inputs exist. The `all` rule exploits the second reading: it
> produces nothing itself; it just declares "these files are what 'done' means."

## Seeing the plan before you run it: the dry run

Before letting Snakemake actually do work — which, for real data, could run for
hours — you should look at *what it intends to do*. This is a **dry run**,
requested with `-n` (for "no execution"). Pair it with `-p`, which **p**rints the
shell commands Snakemake would run:

```bash
$ snakemake -n -p
```

Snakemake reads the Snakefile, builds the DAG, and prints the plan without
touching a single file: the jobs it would run, in order, and the exact commands.
A closely related flag, `-r`, adds the **r**eason for each job ("output does not
exist," "input is newer than output," and so on). The habit to build is: **dry-run
first.** It is the cheapest possible way to catch a mistake — a wrong filename, a
rule that will not chain — before it costs you compute time.

## The real run

Satisfied with the plan, run it for real:

```bash
$ snakemake --cores 1
```

Snakemake prints a job-by-job log as it goes, then leaves you with the finished
files. Look at the result:

```bash
$ cat results/report.txt
sampleA: 2 reads
```

Two things just happened that are worth pausing on. First, you never told
Snakemake to run `count` before `report`; it derived that order from the DAG.
Second, the intermediate file `results/sampleA.count.txt` was created along the
way and is still sitting in `results/` — it is a genuine file, the edge between
two nodes of the graph made concrete on disk.

## Building only what is out of date

Now run the *exact same command* again:

```bash
$ snakemake --cores 1
Nothing to be done (all requested files are present and up to date).
```

Nothing happened — and that is the headline feature from Chapter 2 working for
real. Snakemake compared each output's timestamp against its inputs, found every
output already newer than what it depends on, and concluded there was no work to
do. Re-running a finished pipeline is free.

To see the flip side, **modify an input** and run again. The `touch` command
updates a file's timestamp without changing its contents — a convenient way to
simulate "this input changed":

```bash
$ touch data/sampleA.fastq
$ snakemake --cores 1 -r
```

This time Snakemake notices that `data/sampleA.fastq` is now *newer* than
`results/sampleA.count.txt`, marks that output stale, and re-runs `count` — and
because `count`'s output then changes, it re-runs `report` too. The `-r` flag will
spell out exactly that reasoning. Crucially, it re-ran *only* what was downstream
of the changed file. In this tiny pipeline that is everything; in a sixty-sample
pipeline, changing one sample's reads would re-run that one sample's branch and
leave the other fifty-nine untouched. This is the "which results are now stale?"
problem from Chapter 1, solved automatically.

> This timestamp comparison is the core of Snakemake's rebuild logic, inherited
> from Make. Snakemake can also detect changes to a rule's *code* or *parameters*,
> not just its input files — so editing a recipe can also trigger a re-run. Those
> additional triggers are the subject of Chapter 6; for now, "newer input ⇒
> rebuild" is the mental model to hold.

## The whole thing

Here is the complete Snakefile you built, in order:

```python
rule all:
    input:
        "results/report.txt"

rule count:
    input:
        "data/sampleA.fastq"
    output:
        "results/sampleA.count.txt"
    shell:
        "echo $(( $(wc -l < {input}) / 4 )) > {output}"

rule report:
    input:
        "results/sampleA.count.txt"
    output:
        "results/report.txt"
    shell:
        "echo 'sampleA: '$(cat {input})' reads' > {output}"
```

Thirteen lines, and every idea from Chapter 2 is in there: rules with
inputs/outputs/recipes, a DAG chained through matching filenames, a target that
drives the build, and timestamp-based rebuilds. You can delete the entire
`results/` directory and reproduce it exactly with one command — the smallest
possible taste of reproducibility in action.

## Where we are going

There is one glaring limitation. This pipeline is hard-wired to `sampleA`. A real
study has dozens of samples, and you saw in Chapter 1 that copy-pasting a rule per
sample is exactly the trap we are trying to escape. The fix is the idea that makes
Snakemake scale, and it is the subject of Chapter 5: **wildcards**, which let a
single rule like `count` apply to `sampleA`, `sampleB`, and a hundred others
without being rewritten once.

---

## References

1. Cock PJA, Fields CJ, Goto N, Heuer ML, Rice PM. The Sanger FASTQ file format
   for sequences with quality scores, and the Solexa/Illumina FASTQ variants.
   *Nucleic Acids Research*. 2010;38(6):1767–1771.
