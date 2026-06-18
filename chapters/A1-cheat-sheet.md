# Appendix A — Directive & Command Cheat Sheet

> *A one-page reference to keep beside you. See the chapter in brackets for the
> full treatment.*

## Rule directives

These go inside a `rule`. Only `output` (and a recipe) is strictly required; the
rest are added as needed.

| Directive | What it declares | Chapter |
|---|---|---|
| `input` | Files the rule needs before it can run. Can be a string, a list, named entries (`reads=...`), or a *function* of wildcards. | 2, 4 |
| `output` | Files the rule produces. Wildcards in `output` define the rule's pattern. | 2, 4 |
| `shell` | The recipe as a shell command, using `{input}`/`{output}`/`{params}`/etc. placeholders. | 4 |
| `script` | The recipe as an external Python/R file; receives a `snakemake` object with `input`, `output`, `params`. | 12 |
| `params` | Named values (numbers, flags, labels) used in the recipe as `{params.name}`. Keep settings out of the shell string. | 7 |
| `wildcards` | Not written by you — the values a job's wildcards took, available in the recipe as `{wildcards.name}`. | 5 |
| `log` | A per-job file for the command's output; redirect into it with `2> {log}` or `> {log} 2>&1`. | 10 |
| `threads` | How many cores the rule uses; passed to the tool via `{threads}`. | 9 |
| `resources` | Other needs, e.g. `mem_mb=4000`, `runtime=30`. Used locally and as cluster requests. | 9 |
| `conda` | Path to a conda environment YAML; active with `--sdm conda`. | 8 |
| `container` | A container image (`"docker://..."`); active with `--sdm apptainer`. | 8 |
| `benchmark` | A file to record the job's time/memory/CPU. | 11 |
| `wrapper` | A versioned Snakemake wrapper instead of `shell`/`conda`. | 8 |

## Output markers

Wrap an `output` filename in these:

| Marker | Effect | Chapter |
|---|---|---|
| `temp("...")` | Delete the file once all consuming jobs finish (saves disk). | 10 |
| `protected("...")` | Make the file read-only after the job succeeds. | 10 |
| `directory("...")` | The output is a directory, not a file. | 12 |
| `report("...", caption=...)` | Include the file in `--report` output. | 11 |

## Top-level statements

These go in the Snakefile outside any rule:

| Statement | Purpose | Chapter |
|---|---|---|
| `configfile: "config/config.yaml"` | Load a YAML config into the `config` dict. | 7 |
| `include: "rules/qc.smk"` | Splice in another rule file. | 11 |
| `wildcard_constraints:` | Restrict what wildcards may match (regex). | 5 |
| `module` / `use rule` | Import rules from another workflow. | 11 |
| `expand("{x}.txt", x=LIST)` | Generate a list of filenames from a pattern. | 5 |

## Common command-line flags

```bash
snakemake [flags] [target ...]
```

| Flag | Meaning | Chapter |
|---|---|---|
| `--cores N` / `-c N` | Use up to N local cores. | 4, 9 |
| `--jobs N` / `-j N` | Allow N concurrent jobs (esp. on a cluster). | 9 |
| `-n` / `--dry-run` | Show the plan without running anything. | 4 |
| `-p` | Print the shell commands. | 4 |
| `-r` / `--reason` | Print why each job will run. | 4, 6 |
| `--software-deployment-method conda` (`--sdm conda`) | Build and use `conda:` environments. (Older: `--use-conda`.) | 8 |
| `--sdm apptainer` | Use `container:` images via Apptainer. | 8 |
| `--executor slurm` | Submit jobs to a SLURM cluster. | 9 |
| `--profile DIR` | Load run settings from a profile directory. | 9 |
| `--keep-going` / `-k` | Don't stop on the first failure; finish what you can. | 10 |
| `--rerun-incomplete` | Redo jobs whose outputs were left incomplete. | 10 |
| `--rerun-triggers mtime` | Restrict what counts as "out of date." | 6 |
| `--dag` / `--rulegraph` | Emit the job-level / rule-level graph in Graphviz `dot`. | 6 |
| `--report FILE.html` | Generate a self-contained HTML report. | 11 |
| `--version`, `--help` | Version; full option list. | 3 |

## The shape of a well-formed rule

```python
rule align:
    input:
        reads="results/{sample}.fastq",
        index="results/index",
    output:
        "results/{sample}.bam"
    params:
        extra=config["aligner_args"]
    log:
        "logs/align/{sample}.log"
    threads: 4
    resources:
        mem_mb=8000,
        runtime=60,
    conda:
        "../envs/aligner.yaml"
    shell:
        "aligner {params.extra} -p {threads} -x {input.index} "
        "-U {input.reads} -o {output} > {log} 2>&1"
```
