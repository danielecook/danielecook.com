---
title: Parallelize by iterating over chromosomal ranges
author: Daniel Cook
date: 2020-01-29T01:15:53Z
tags:
  - BAM
  - VCF
  - nim
  - seq-collection
---

I have added a new utility to `seq-collection` called `iter` which generates chromosomal ranges. Lists of genomic ranges can be easily plugged into utilities such as `xargs` or [gnu-parallel](https://www.gnu.org/software/parallel/) to parallelize commands.

For example:

```bash
sc iter test.bam 100,000 # Iterate on bins of 100k base pairs

# Outputs
> I:0-999999
> I:1000000-1999999
> I:2000000-2999999
> I:3000000-3999999
> I:4000000-4999999
```

<small><strong>Note:</strong> BAMs use a 0-based coordinate system; VCFs are 1-based</small>

This list of genomic ranges can be used to process a BAM or VCF in parallel:

```bash

function process_chunk {
  # Code to process chunk
  vcf=$1
  region=$2
  # e.g. bcftools call -m --region 
  echo bcftools call --region $region $vcf # ...
}

# Export the function to make it available to GNU parallel
export -f process_chunk

parallel --verbose process_chunk ::: test.bam ::: $(sc iter test.bam)

```

You can also set the `[width]` option to 0 to generate a list of chromosomes.

See [Using GNU-Parallel for Bioinformatics](/using-gnu-parallel-for-bioinformatics/) for a comprehensive guide on using Parallel for bioinformatics.

[seq-collection](https://github.com/danielecook/seq-collection) (__sc__) is a set of tools written in [nim](https://nim-lang.org/) and using the fantastic [hts-nim](https://github.com/brentp/hts-nim) package. 