---
title: Using GNU-Parallel for bioinformatics
author: Daniel Cook
date: 2019-09-27T01:30:53Z
tags:
  - bioinformatics
  - tutorial
  - bash
---

[GNU Parallel](https://www.gnu.org/software/parallel/) is an indispensible tool for speeding up bioinformatics. It allows you to easily parallelize commands. Below, I detail some of the basics regarding how it is used and how it can be applied to bioinformatics.

Many HPC clusters will have GNU-Parallel pre-installed or available as a module. You can also install it using [homebrew](brew.sh) or other package managers.

# Basic Usage

Lets start with a basic example:

```bash
seq 1 5 | parallel -j 4 echo
```

Here we are (1) Printing a sequence of numbers from 1 to 5, and (2) piping this data into `parallel`. We have provided the command `echo` which will be parallelized across `-j=4` jobs. We can see what this looks like by using the `--dry-run` flag which prints the commands to be run.

```bash
seq 1 5 | parallel --dry-run -j 4 echo
```
```
echo 3
echo 4
echo 5
echo 2
echo 1
```

The results are out of order! This is due to the nature of parallelization. Not all "jobs" initiate or take the same amount of time so it is common to observe outputs in a different order. We can enforce a "first in first out" result set by using the `-k` flag. Lets see what the __output__ looks like by removing the `--dry-run` flag:

```bash
seq 1 5 | parallel -j 4 -k echo
```
```
1
2
3
4
5
```

Like any other command, you can send this output to a file:

```bash
seq 1 5 | parallel -j 4 -k echo > out.txt
```

## `-j`

In order for GNU Parllel to work, you want to have a multi-core CPU. Parallelizing across more cores than you have available can actually make performance __worse__, so it is important to tune the `-j` parameter to the number of cores available.

Luckily, parallel allows you to specify `-j` using a percentage of cores or as a number relative to the total number of cores. For example:

```bash
parallel -j 100% # Uses 100% of cores.
parallel -j -1 # Uses 1 less than the total number of cores.
parallel -j +1 # Parallelize across the number of cores + 1
```


## Use `:::` for args

Use `:::` to specify arguments derived from commands or lists.

```bash
parallel -j 4 -k echo ::: `seq 1 5`
```

__Note__ - There are limits to the number of arguments you can provide with a process substitution as shown above. In these instances, it may be better to pipe arguments or use a file (below) rather than supply them with a process substitution:

```bash
seq 1 5 | parallel -j 4 -k echo
```

## Use `::::` for args within files

For large argument lists you can specify a file with a list of arguments. Specify a file of arguments (one per line) using `::::`.

```bash
parallel -j 4 -k echo :::: my_args.txt
```


## Use `{}` to specify argument variables

By default, `parallel` assumes the arguments are placed at the end of the input command, but you can explicitly define where arguments are substituted using `{}`:

```bash
parallel --dry-run -j 4 -k echo \"{} "<-- a number"\" ::: `seq 1 5`
```
```
echo "1 <-- a number"
echo "2 <-- a number"
echo "3 <-- a number"
echo "4 <-- a number"
echo "5 <-- a number"
```

Notice that we are having to escape quotes - there are ways around this.

## Combinatorials

You can keep adding `:::` and `::::` to add additional arguments, and these will be combined to generate all possible combinations. This is extremely useful for testing commands with different combinations of input parameters. 

```bash
parallel --dry-run -k -j 4 Rscript run_analysis.R {1} {2} ::: `seq 1 2` ::: A B C
```
```csv
Rscript run_analysis.R 1 A
Rscript run_analysis.R 1 B
Rscript run_analysis.R 1 C
Rscript run_analysis.R 2 A
Rscript run_analysis.R 2 B
Rscript run_analysis.R 2 C
```

## Parallelize Functions

In some cases, you want to perform a series of commands. For example, the code below compute the number of ATCGs of the complement of a DNA sequence. 

```bash
echo "ATTA" |  tr ATCG TAGC | \
    python -c "import sys; o=sys.stdin.read().strip(); print(o, o.count('T'), o.count('G'), o.count('C'), o.count('A'))"
```

This command has two operations. While is it possible to incorporate this into a 'one-liner', it is far easier to create a bash function, export it, and use that as input.

```shell 
function count_nts {
    # $1 is the first argument passed to the function
    echo $1 | tr ATCG TAGC | \
    python -c "import sys; o=sys.stdin.read().strip(); print(o, o.count('T'), o.count('G'), o.count('C'), o.count('A'))"
}

# Use the `-f` flag to export functions
export -f count_nts

parallel -j 4 count_nts ::: TAAT TTT AAAAT GCGCAT | tr ' ' '\t'

```

With the basics down, lets see how we can use parallel to speed up bioinformatics.

# GNU Parallel for Variant Calling

When working with BAMs or VCFs you can parallelize across chromosomes. Most variant callers or annotation tools allow you to operate on a single chromosome at a time by specifying a region. This allows us to apply a `split-apply-combine` strategy by splitting by chromosome, operating on each chromosome, and combining the results at the end.

## __Split__ chromosomes from a BAM

```bash

chrom_list=`samtools idxstats in.bam | cut -f 1 | grep -v '*'`

# For c. elegans you can would see the following 7
# I
# II
# III
# IV
# V
# X
# MtDNA
```

We can create a function so this operation is easier going forward:

```bash
function bam_chromosomes {
    samtools idxstats $1 | cut -f 1 | grep -v '*'
}
```

## __Apply__ an operation to each chromosome

Here is where GNU parallel comes into play: Parallelized variant calling by chromosome:

```bash
#!/bin/bash

genome=path/to/genome.fa
export genome # This is critical!

function parallel_call {
    bcftools mpileup \
        --fasta-ref ${genome} \
        --regions $2 \
        --output-type u \
        $1 | \
    bcftools call --multiallelic-caller \
                  --variants-only \
                  --output-type u - > ${1/.bam/}.$2.bcf
}

export -f parallel_call

chrom_set=`bam_chromosomes test.bam`
parallel --verbose -j 4 parallel_call sample_A.bam ::: ${chrom_set}

```

A few important notes regarding this step:

* You must export any variables you use within a parallelized function. That is what I am doing here with the reference `genome` variable.
* `bcftools mpileup` outputs an uncompressed pileup (`--output-type=u`). This is done for efficiency sake - there is no reason to pipe a compressed form of data for it to need to be uncompressed by the next tool.
* Similarly, I also output an uncompressed set of variant calls `${1/.bam/}.$2.bcf` because these are temporary files that we will remove later.
* Finally, I use a variable substitution to remove the extension from the bam and to generate a `<sample>.<chromsome>.bcf` filename: `${1/.bam/}.$2.bcf` → `sample_A.I.bam`, `sample_A.II.bam`, etc. This prevents filename collisions if we are calling many samples simultaneously.

## __Combine__ the variant calls.

Once we have completed variant calling we need to combine everything back in the right order. We can use a bash array to add a prefix and suffix to the list of chromosomes to reconstruct the output filenames and concatenate them into a single file.

```bash
# Generate an array of the resulting files
# to be concatenated.
sample_name="sample_A"
set -- `echo $chrom_set | tr "\n" " "`
set -- "${@/#/${sample_name}.}" && set -- "${@/%/.bcf}"
# This will generate a list of the output files:
# sample_A.I.bcf sample_B.II.bcf etc.

set -- "${@/#/test.}" && set -- "${@/%/.bcf}"

# Output compressed result
bcftools concat $@ --output-type b > $sample_name.bcf

# Remove intermediate files
rm $@
```

To ensure the intermediate files are removed even when errors occur you should use a [bash trap](http://redsymbol.net/articles/bash-exit-traps/).

# Summary

GNU Parallel can greatly speed up simple parallelization scenerios. Additional code is often required to handle the "splitting" and "combining" steps, but this can allow for tremendous efficiency gains.

