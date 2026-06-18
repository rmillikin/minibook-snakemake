# Chapter 9 — Scaling Up: Threads, Resources, Clusters, and the Cloud

> *The same pipeline, from one laptop core to a thousand-node cluster — without
> rewriting a rule.*

Everything so far has run on a single core processing toy files in milliseconds.
Real RNA-seq is not like that: a study may have a hundred multi-gigabyte samples,
and a single alignment can occupy several processors and many gigabytes of memory
for an hour. This chapter is about scale — how to tell Snakemake what each rule
*needs*, and how to dispatch the work across a laptop, a big shared computer, or
the cloud. The crucial theme, and the reason this is a short chapter rather than a
rewrite of everything: **scaling up changes how a workflow is *run*, not how it is
*written*.** The rules you have learned to write do not change.

## Cores, threads, and jobs

Three words get used loosely in conversation but mean distinct things here, and
keeping them straight makes the rest of the chapter easy.

> A **core** (or CPU core) is one physical processing unit; a modern laptop has a
> handful, a cluster node has dozens. A **thread** is one stream of computation; a
> program written to use multiple threads can do work on several cores at once. A
> **job**, as we defined it in Chapter 5, is one rule with its wildcards filled in
> — one box in the DAG. The scaling question is: *how do we map jobs and their
> threads onto the available cores?*

When you ran `snakemake --cores 1`, you handed Snakemake a budget of one core. It
is the scheduler's job to spend that budget. Give it more, and it spends more —
this is where the parallelism that has been latent in the DAG since Chapter 2
finally pays off:

```bash
$ snakemake --cores 8
```

With eight cores, Snakemake looks at the DAG, finds jobs whose inputs are ready and
which do not depend on one another, and runs as many simultaneously as the budget
allows. Recall the three independent `fastqc` jobs (one per sample): on one core
they run one after another; on eight cores Snakemake fires all three at once and
finishes in roughly a third of the time. You wrote nothing to make this happen — the
independence was visible in the graph, and the scheduler exploited it.

> `--cores` sets the budget on a single machine. Its cousin `--jobs` (or `-j`) sets
> how many jobs may run at once when Snakemake is submitting work to a *cluster*,
> where the limit is not local cores but how many jobs the system will let you queue.
> We get to that below.

## Telling a rule how many threads it needs

Some tools are themselves multi-threaded: give an aligner four threads and it
processes one sample roughly four times faster. You declare this with the
**`threads`** directive, and pass the value into the tool via the `{threads}`
placeholder — the same substitution machinery as `{input}` and `{output}`:

```python
rule fastqc:
    input:
        "results/{sample}.fastq"
    output:
        "results/qc/{sample}_fastqc.html"
    threads: 4
    conda:
        "envs/fastqc.yaml"
    shell:
        "fastqc --threads {threads} {input} --outdir results/qc/"
```

Declaring `threads: 4` does two cooperating things. It tells the *tool* to use four
threads (via `{threads}` in the command), and it tells the *scheduler* that each
such job costs four cores out of the budget. So under `--cores 8`, Snakemake will
run two `fastqc` jobs at a time (4 + 4 = 8), not three. The two numbers stay
consistent automatically — and if you run with fewer cores than a rule requests,
Snakemake quietly scales the rule down to fit rather than overcommitting the
machine. This is why you should declare threads on the rule rather than hard-coding
`-t 4` into the command: it keeps the tool and the scheduler in agreement.

## Declaring other resources: memory and time

CPU is not the only thing a job consumes. Alignment can need tens of gigabytes of
RAM; a long step can run for hours. On a shared machine, ignoring these limits is
how you crash a node or get your jobs killed. The **`resources`** directive lets a
rule declare what it needs beyond cores:

```python
rule fastqc:
    input:
        "results/{sample}.fastq"
    output:
        "results/qc/{sample}_fastqc.html"
    threads: 4
    resources:
        mem_mb=2000,      # megabytes of memory this job needs
        runtime=30,       # expected wall-clock minutes
    conda:
        "envs/fastqc.yaml"
    shell:
        "fastqc --threads {threads} {input} --outdir results/qc/"
```

`resources` entries are arbitrary named quantities; `mem_mb` (memory in megabytes)
and `runtime` (minutes) are conventional and widely understood. Locally, Snakemake
uses them to avoid over-packing the machine — it will not start a set of jobs whose
combined `mem_mb` exceeds the memory budget you give it (`--resources mem_mb=16000`).
On a cluster, as we are about to see, these same numbers become the resource request
the scheduler uses to place each job. Declaring them once, on the rule, makes the
rule behave sensibly everywhere.

## Running on a cluster

A laptop runs out of cores quickly. Serious genomics runs on a **high-performance
computing (HPC) cluster**: many networked machines (nodes) shared by many users. You
do not run programs on a cluster directly — you submit them to a **job scheduler**.

> A **job scheduler** (common ones are **SLURM**, SGE, and PBS) is the cluster's
> traffic controller. You hand it a job along with a statement of what the job needs
> (so many cores, so much memory, so much time), and it finds a node with room,
> runs the job there when resources free up, and reports back. It lets hundreds of
> users share a cluster fairly without trampling each other.

Here is the elegant part. Snakemake already knows each job's requirements — you
declared them with `threads` and `resources`. So instead of running jobs itself, it
can *submit each job to the scheduler* on your behalf, translating your declarations
into the scheduler's resource request. The component that knows how to talk to a
particular backend is called an **executor**:

> An **executor** is the plug-in that decides *where and how* jobs actually run —
> on local cores, by submitting to SLURM, by launching cloud instances, and so on.
> Switching executors switches the backend; the workflow's rules are untouched. This
> is the mechanism behind the chapter's promise that scaling changes how a pipeline
> is run, not how it is written.

In modern Snakemake you select the SLURM executor and let it submit up to, say, 100
jobs at a time:

```bash
$ snakemake --executor slurm --jobs 100
```

Snakemake now submits each ready job to SLURM with its declared cores, memory, and
runtime; SLURM schedules them across the cluster; and your hundred samples process
in parallel across many nodes. The exact same Snakefile that ran on your laptop runs
here — only the command changed.

## Profiles: capturing the run configuration

That command line will grow unwieldy fast (executor, job limits, default memory,
partition names, retry policy…), and you do not want to retype it or remember it.
A **profile** bundles all of these run-time settings into a small configuration
directory you can name once.

A profile is a folder containing a `config.yaml` of default flags. A minimal SLURM
profile, say `profiles/slurm/config.yaml`:

```yaml
executor: slurm
jobs: 100
default-resources:
  mem_mb: 4000
  runtime: 60
  slurm_partition: general
```

With that in place, the whole cluster configuration collapses to:

```bash
$ snakemake --profile profiles/slurm
```

And — the point of the whole chapter — running the *same workflow* on your laptop is
just a different profile (or none at all):

```bash
$ snakemake --cores 4        # laptop: local cores
$ snakemake --profile profiles/slurm   # cluster: submit to SLURM
```

`default-resources` supplies fallback values for any rule that did not declare its
own, so even rules without explicit `resources` get sensible requests. Our group
keeps a shared, tuned profile for our cluster; as a new student, you will typically
be handed that profile and run with it rather than writing one from scratch — but now
you understand what it is doing.

## A note on the cloud

Cloud execution — running on rented machines from providers like AWS, Google Cloud,
or a Kubernetes cluster — is conceptually nothing new after the cluster discussion:
it is simply *another executor*. You point Snakemake at a cloud backend and a storage
location for inputs and outputs, and the same DAG-of-jobs gets dispatched to cloud
machines instead of cluster nodes. The rules, again, do not change. When and whether
to use the cloud is mostly a question of cost and data logistics rather than of
Snakemake itself, so we leave the specifics to our group's infrastructure notes.

## Where we are going

You can now express what each rule needs — cores via `threads`, memory and time via
`resources` — and run the identical workflow on a laptop, a SLURM cluster, or the
cloud just by choosing an executor or a profile. Scale is handled, and reproducibly
so.

The remaining gap is robustness. At scale, jobs *will* fail — a node runs out of
memory, an input is malformed, a tool returns an error. Chapter 10 is about what
happens then: reading Snakemake's failure output, logging each rule so you can
diagnose problems, protecting against half-written files, and resuming a large run
without redoing the work that already succeeded.

---

## References

1. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33. (Resource specification and cluster/cloud
   execution.)
2. Yoo AB, Jette MA, Grondona M. SLURM: Simple Linux Utility for Resource
   Management. In: *Job Scheduling Strategies for Parallel Processing (JSSPP 2003)*.
   Lecture Notes in Computer Science, vol 2862. Springer; 2003:44–60.
