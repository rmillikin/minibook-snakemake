# Appendix B — A Minimal Python & Shell Refresher

> *The small amount of Python and shell this book assumes, gathered in one place.*

You do not need to be fluent in either Python or the shell to use Snakemake, but a
handful of constructs come up constantly. Chapter 3 introduced the essentials in
passing; this appendix collects them for reference.

## Python

Snakemake files *are* Python, so the language's basics apply directly.

### Indentation

Python groups code by **indentation** (leading spaces), not braces. In a Snakefile,
the parts of a rule are indented under it, and the recipe under its directive. Be
consistent — mixing tabs and spaces is a common, confusing error.

### Strings

Text in quotes. Filenames are strings:

```python
"results/sampleA.bam"
'single quotes work too'
```

### Lists

Ordered collections in square brackets:

```python
samples = ["ctrl1", "ctrl2", "treat1"]
samples[0]        # → "ctrl1"  (indexing starts at 0)
len(samples)      # → 3
```

### Dictionaries

Key→value maps in curly braces. Snakemake's `config` is a dict:

```python
settings = {"genome": "GRCh38", "threads": 8}
settings["genome"]     # → "GRCh38"
```

### f-strings

A string prefixed with `f` that substitutes values inside `{...}`:

```python
sample = "ctrl1"
f"results/{sample}.bam"        # → "results/ctrl1.bam"
```

> This is the same brace-substitution intuition as Snakemake's `{input}`/`{output}`
> placeholders — though those are filled in by Snakemake, not by Python f-string
> evaluation.

### Functions and `lambda`

A named function with `def`, or a small unnamed one with `lambda`:

```python
def fastq_for(wildcards):
    return samples.loc[wildcards.sample, "fastq"]

# the same thing, inline:
lambda wildcards: samples.loc[wildcards.sample, "fastq"]
```

Input functions (Chapter 7) are exactly this: a function that takes `wildcards` and
returns filename(s).

### `for` loops

Occasionally useful for building inputs:

```python
for s in samples:
    print(f"results/{s}.bam")
```

### pandas (one operation)

The book uses pandas only to read a table and look up cells by label:

```python
import pandas as pd
df = pd.read_csv("config/samples.tsv", sep="\t").set_index("sample")
df.loc["ctrl1", "fastq"]    # value in row "ctrl1", column "fastq"
list(df.index)              # the list of row labels (sample names)
```

## The shell

Recipes in `shell:` are commands run by **Bash**. The pieces you will see:

### Paths

Files are named by path. `/` separates directories; `.` is the current directory,
`..` the parent. `results/qc/ctrl1.html` is a relative path from where you ran
Snakemake.

### Pipes

A pipe `|` sends one command's output straight into the next, forming a chain
without temporary files:

```bash
hisat2 -x index -U reads.fq | samtools sort -o out.bam
```

### Redirection

`>` sends a command's output to a file; `<` reads input from one. The output
streams have numbers: **standard output** is `1`, **standard error** (diagnostics)
is `2`.

```bash
command > out.txt          # stdout to a file (overwrite)
command >> out.txt         # stdout appended to a file
command 2> errors.log      # stderr to a file
command > all.log 2>&1     # both stdout and stderr to one file
wc -l < input.txt          # feed a file in as stdin
```

The `2> {log}` and `> {log} 2>&1` patterns in Chapter 10's rules are exactly these.

### Exit codes

Every command returns a numeric **exit code**: `0` means success, anything else
means failure. This is how Snakemake knows a job failed — a non-zero exit code from
the recipe aborts the job (Chapter 10). You rarely write exit codes yourself, but it
helps to know that "exited with non-zero exit code" in an error message simply means
"the program reported failure."

### Command substitution

`$(...)` runs a command and substitutes its output into the line. Chapter 4 used it
to count reads:

```bash
echo $(( $(wc -l < reads.fastq) / 4 ))   # $(...) runs wc; $(( )) does arithmetic
```

That is the whole vocabulary the book relies on. For anything deeper, the Python
documentation (docs.python.org) and any introductory Bash guide go far beyond what
Snakemake requires of you.
