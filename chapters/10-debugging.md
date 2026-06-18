# Chapter 10 — Robustness, Logging, and Debugging

> *What to do when — not if — a pipeline breaks.*

At the toy scale of earlier chapters, things just worked. At real scale they will
not, and that is normal: with a hundred samples and a dozen tools, some job will
eventually run out of memory, hit a malformed input, or trip over a tool's bug. A
pipeline you can trust is not one that never fails — it is one that **fails
safely** (without corrupting your results) and **fails legibly** (telling you
exactly what went wrong) so you can fix it and resume without redoing the work that
already succeeded. This chapter is about those properties and the directives that
give them to you.

## Reading a failure

The first skill is simply reading Snakemake's error output, because it is more
helpful than it first looks. When a job's command exits with a non-zero status —
the universal Unix signal for "this program failed" — Snakemake stops and prints a
block like this:

```
Error in rule count:
    jobid: 4
    input: results/sampleB.fastq
    output: results/sampleB.count.txt
    log: logs/count/sampleB.log (check log file(s) for error details)
    shell:
        (echo 'sampleB: '$(( $(wc -l < results/sampleB.fastq) / 4 )) ...) 2> logs/count/sampleB.log
        (exited with non-zero exit code)
```

Read it top to bottom and it answers the questions you need: *which rule* failed
(`count`), *for which sample* (the wildcards are visible in the input/output), the
*exact command* Snakemake ran (you can copy-paste and run it by hand to reproduce
the failure), and *where the log is*. Almost every debugging session starts here.
The single most common beginner mistake is to scroll past this block in a panic;
slow down and read it, and it usually points straight at the problem.

> By default, a single failure does not abort jobs already running — Snakemake lets
> in-flight jobs finish, then stops launching new ones and exits non-zero. If you
> would rather it push on and complete every job it *can*, collecting failures at
> the end, add `--keep-going`. On a long cluster run this is often what you want:
> one bad sample should not idle the other ninety-nine.

## Logging: the `log` directive

Notice the error pointed at a *log file* rather than dumping the tool's output to
your screen. That is the **`log`** directive at work, and adopting it everywhere is
the highest-value habit in this chapter.

When you run a hundred jobs — especially across a cluster, where there is no console
to watch — you cannot have every tool's chatter interleaved on one terminal. Instead
each job should write its own output to its own file. The `log` directive declares
that file (with wildcards, so each job gets a distinct one), and you redirect the
command's output into it:

```python
rule count:
    input:
        "results/{sample}.fastq"
    output:
        "results/{sample}.count.txt"
    log:
        "logs/count/{sample}.log"
    shell:
        "(echo '{wildcards.sample}: '$(( $(wc -l < {input}) / 4 ))' reads' > {output}) 2> {log}"
```

The `2> {log}` redirects the command's **standard error** (the stream where Unix
programs write diagnostics) into the log file; `> {log} 2>&1` would capture both
normal output and errors. Now when `count` fails for `sampleB`, its complaint is
sitting in `logs/count/sampleB.log`, isolated from every other job, named for the
exact sample that failed.

> Two rules of thumb. First, **give every nontrivial rule a `log`** — the five
> seconds it costs you while writing the rule saves an hour when that rule fails at
> 2 a.m. on sample 73. Second, a `log` file is *not* an `output`: Snakemake does not
> treat it as a product to be built, and — usefully — it does not delete the log when
> the job fails, which is precisely when you need to read it.

## Failing safely: incomplete and atomic outputs

Here is a subtle danger. Suppose `count` is killed *halfway* through writing
`results/sampleB.count.txt` — the node dies, the cluster preempts the job. The
output file now exists but is truncated and wrong. If a later run saw that file and
assumed "the output exists, so the job is done," it would feed corrupt data
downstream and you would never know.

Snakemake guards against this automatically. When a job fails, **Snakemake deletes
the outputs of that job**, on the principle that a half-made file is worse than no
file — better to rebuild than to trust a corpse. This is what people mean by
treating outputs as **atomic**: from the pipeline's point of view a file either
exists complete or does not exist at all, never half-done.

There is one case it cannot clean up: if Snakemake *itself* is killed (you lose your
SSH connection, the head job is cancelled), it has no chance to tidy up, and a job's
output may be left in an **incomplete** state. On the next run Snakemake detects this
from its own bookkeeping and refuses to trust the file, telling you to re-run it
with:

```bash
$ snakemake --cores 8 --rerun-incomplete
```

which redoes exactly the jobs whose outputs were left incomplete, and nothing else.

## Managing intermediates: `temp()` and `protected()`

Two more markers help with the *lifecycle* of output files, and both matter more as
data gets large.

Our pipeline decompresses each `.fastq.gz` into a full-size `.fastq`, used only as
input to the next steps and then never again. Multiply by a hundred multi-gigabyte
samples and these intermediates can swamp your disk. Wrapping an output in **`temp()`**
tells Snakemake to **delete it automatically as soon as every job that consumes it
has finished**:

```python
rule decompress:
    input:
        lambda wildcards: samples.loc[wildcards.sample, "fastq"]
    output:
        temp("results/{sample}.fastq")
    shell:
        "gunzip -c {input} > {output}"
```

The decompressed FASTQ still gets made and used, but it is swept away the moment
`count` (and any other consumer) is done with it — keeping disk usage bounded
without you tracking which intermediates are still needed.

The mirror image is **`protected()`**, which marks an output as precious: after the
job succeeds, Snakemake makes the file read-only, so a stray command or an
accidental re-run cannot clobber a result that took hours to produce.

```python
output:
    protected("results/report.txt")
```

Use `temp()` for cheap-to-remake intermediates you do not want cluttering disk, and
`protected()` for expensive, downstream-critical results you do not want to lose.

## A field guide to common failures

A handful of errors account for most of what you will hit. Recognizing them by name
shortens debugging from an afternoon to a minute.

- **`MissingInputException`** — Snakemake needs a file but no rule produces it and it
  does not exist. Almost always a *typo in a filename or wildcard*, or a sample path
  in the sample sheet that does not exist. Check spelling and paths first.
- **`AmbiguousRuleException`** — two different rules can both produce the same output
  file, so Snakemake does not know which to use. Make the outputs distinct, or use
  the wildcard constraints from Chapter 5 to separate them.
- **Wildcard errors** — a wildcard cannot be resolved, or grabs more of a filename
  than intended. Revisit Chapter 5's `wildcard_constraints`.
- **Non-zero exit code** — the *tool itself* failed. Snakemake cannot tell you why;
  the answer is in the rule's **log file**. Open it. This is the payoff for the
  logging habit above.
- **Out-of-memory (job killed by the scheduler)** — common on clusters and often
  reported as a job that died without a tidy error. The fix is to raise the rule's
  `resources: mem_mb` (Chapter 9) and re-run.

## Resuming without redoing work

The reassuring through-line: because Snakemake tracks which outputs exist and are
up to date (Chapter 6), **recovering from a failure never means starting over.**
After you fix the cause, just run the same command again. Snakemake skips every job
that already completed successfully and re-runs only the failed job and whatever
depends on it. On a run where eighty samples finished and twenty died from a memory
limit, bumping the memory and re-running processes only those twenty.

You can also aim a run at a single target to debug in isolation — exactly the
command-line targeting from Chapter 6:

```bash
$ snakemake --cores 1 results/sampleB.count.txt   # rebuild just this one, to study it
```

The combination — read the failure block, open the log, fix, re-run, let Snakemake
resume — is the entire debugging loop. It is fast precisely because the system never
forgets what already worked.

## Where we are going

You can now make a pipeline that fails safely and legibly, and recover from failure
without wasted work. Together with reproducible inputs (Chapter 7), reproducible
software (Chapter 8), and scalable execution (Chapter 9), you have everything needed
to run real workflows responsibly. What remains is keeping a *large* project
organized as it grows past a single Snakefile — splitting rules across files,
reusing components, and generating the reports and provenance records that make work
shareable. That is Chapter 11.

---

## References

1. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33. (Logging, temporary/protected outputs,
   and error handling.)
