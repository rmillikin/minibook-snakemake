# Appendix D — Further Reading & References

> *The consolidated reference list, plus pointers to where to go next.*

## Official Snakemake resources

- **Snakemake documentation** — https://snakemake.readthedocs.io/
  Comprehensive reference and a hands-on tutorial that pairs well with this book.
- **Snakemake Wrapper Repository** — https://snakemake-wrappers.readthedocs.io/
  Vetted, versioned, reusable rule bodies for common bioinformatics tools (Ch. 8).
- **Snakemake Workflow Catalog** — https://snakemake.github.io/snakemake-workflow-catalog/
  Published, ready-to-use workflows worth reading as examples.
- **Executor plugin catalog** — https://snakemake.github.io/snakemake-plugin-catalog/
  Backends for clusters and clouds (SLURM, Kubernetes, cloud providers; Ch. 9).

## Getting help

- The Snakemake tag on **Stack Overflow**, and the project's **GitHub Discussions**.
- Most usefully: the group's own existing pipelines and the labmates who wrote them.

## Cited references

Numbered per chapter of first appearance; some are cited in more than one chapter.

### Workflow management & reproducibility

1. Köster J, Rahmann S. Snakemake — a scalable bioinformatics workflow engine.
   *Bioinformatics*. 2012;28(19):2520–2522.
2. Mölder F, Jablonski KP, Letcher B, et al. Sustainable data analysis with
   Snakemake. *F1000Research*. 2021;10:33.
3. Wratten L, Wilm A, Göke J. Reproducible, scalable, and shareable analysis
   pipelines with bioinformatics workflow managers. *Nature Methods*.
   2021;18(10):1161–1168.
4. Di Tommaso P, Chatzou M, Floden EW, et al. Nextflow enables reproducible
   computational workflows. *Nature Biotechnology*. 2017;35(4):316–319.
5. Feldman SI. Make — a program for maintaining computer programs. *Software:
   Practice and Experience*. 1979;9(4):255–265.
6. Baker M. 1,500 scientists lift the lid on reproducibility. *Nature*.
   2016;533(7604):452–454.
7. Sandve GK, Nekrutenko A, Taylor J, Hovig E. Ten simple rules for reproducible
   computational research. *PLoS Computational Biology*. 2013;9(10):e1003285.

### Software environments & infrastructure

8. Grüning B, Dale R, Sjödin A, et al. Bioconda: sustainable and comprehensive
   software distribution for the life sciences. *Nature Methods*.
   2018;15(7):475–476.
9. McKinney W. Data structures for statistical computing in Python. *Proceedings of
   the 9th Python in Science Conference (SciPy)*. 2010:56–61.
10. Yoo AB, Jette MA, Grondona M. SLURM: Simple Linux Utility for Resource
    Management. In: *Job Scheduling Strategies for Parallel Processing (JSSPP
    2003)*. Lecture Notes in Computer Science, vol 2862. Springer; 2003:44–60.

### RNA-seq biology & tools

11. Wang Z, Gerstein M, Snyder M. RNA-Seq: a revolutionary tool for transcriptomics.
    *Nature Reviews Genetics*. 2009;10(1):57–63.
12. Conesa A, Madrigal P, Tarazona S, et al. A survey of best practices for RNA-seq
    data analysis. *Genome Biology*. 2016;17:13.
13. Cock PJA, Fields CJ, Goto N, Heuer ML, Rice PM. The Sanger FASTQ file format for
    sequences with quality scores, and the Solexa/Illumina FASTQ variants. *Nucleic
    Acids Research*. 2010;38(6):1767–1771.
14. Andrews S. FastQC: a quality control tool for high throughput sequence data.
    2010. Babraham Bioinformatics.
    https://www.bioinformatics.babraham.ac.uk/projects/fastqc/
15. Patro R, Duggal G, Love MI, Irizarry RA, Kingsford C. Salmon provides fast and
    bias-aware quantification of transcript expression. *Nature Methods*.
    2017;14(4):417–419.
16. Ewels P, Magnusson M, Lundin S, Käller M. MultiQC: summarize analysis results
    for multiple tools and samples in a single report. *Bioinformatics*.
    2016;32(19):3047–3048.

## Where to go after this book

- Work through the **official Snakemake tutorial** end to end; it reinforces this
  book with a different example.
- Read one of the group's **production pipelines** in full, mapping each rule back
  to the relevant chapter.
- Complete the extension exercises in Chapter 12 (read trimming, paired-end support,
  reference download, differential expression).
- For the RNA-seq science beyond the mechanics, references 11 and 12 are excellent
  starting points.
