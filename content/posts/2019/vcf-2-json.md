---
title: Converting VCF To JSON
author: Daniel Cook
date: 2019-09-22T01:15:53Z
tags:
  - VCF
  - nim
  - seq-collection
---

Recently I started developing a set of utilities called [seq-collection](https://github.com/danielecook/seq-collection) (__sc__) written in [nim](https://nim-lang.org/) and using the fantastic [hts-nim](https://github.com/brentp/hts-nim) package.

The first utility I added was a tool to convert a VCF to JSON. This tool is useful for building out an API that reads genotype data directly from the VCF format. It is possible to read specific variants or intervals of VCF files when they are indexed, allowing for fast and efficient querying of genetic data without the need for a database. Furthermore, these queries can be made over http connections making it possible to use a VCF file as a database.

For example, as a graduate student I developed a [genome browser](https://elegansvariation.org/data/browser/) for _C. elegans_ wild isolates. Queries are made directly on specific genomic intervals of VCF files using `bcftools`. However, a large amount of python code is used to convert VCF output ot JSON format. The `sc` utility could replace this underlying python code on the server with a simple binary and a few command line arguments to accomplish the same task. Below I have a few examples illustrating how to use `sc json`.

# Installation

You can download the MAC OSX binary [here](https://github.com/danielecook/seq-collection/releases/download/0.0.1/sc_macosx).

Or build from source at [danielecook/seq-collection](http://www.github.com/danielecook/seq-collection)

# Usage

```bash
json

Convert a VCF to JSON

Usage:
  json [options] vcf [region ...]

Arguments:
  vcf              VCF to convert to JSON
  [region ...]     List of regions

Options:
  -i, --info=INFO            comma-delimited INFO fields; Use 'ALL' for everything
  -f, --format=FORMAT        comma-delimited FORMAT fields; Use 'ALL' for everything
  -s, --samples=SAMPLES      Set Samples (default: ALL)
  -p, --pretty               Prettify result
  -a, --array                Output as a JSON array instead of individual JSON lines
  -z, --zip                  Zip sample names with FORMAT fields (e.g. {'sample1': 25, 'sample2': 34})
  -n, --annotation           Parse ANN Fields
  --pass                     Only output variants where FILTER=PASS
  --debug                    Debug
  -h, --help                 Show this help
```

# Examples

## List all sites

```bash
sc json tests/data/test.vcf.gz
```

__Output__

```json
{"CHROM":"I","POS":41947,"ID":".","REF":"A","ALT":["T"],"QUAL":999.0,"FILTER":["PASS"]}
{"CHROM":"I","POS":105133,"ID":".","REF":"G","ALT":["A"],"QUAL":999.0,"FILTER":["PASS"]}
{"CHROM":"I","POS":176422,"ID":".","REF":"A","ALT":["G"],"QUAL":999.0,"FILTER":["PASS"]}
```

## pretty output

We can "prettify" this output using the `--pretty` flag:

```bash
sc json --pretty tests/data/test.vcf.gz
```

```json
{
  "CHROM": "I",
  "POS": 41947,
  "ID": ".",
  "REF": "A",
  "ALT": [
    "T"
  ],
  "QUAL": 999.0,
  "FILTER": [
    "PASS"
  ]
}
```

## Fetch Genotypes

Next we can output genotype calls by specifying the `--FORMAT` flag with a `GT` argument:

```bash
> sc json --FORMAT=GT tests/data/test.vcf.gz
```
```json
{"CHROM":"I","POS":41947,"ID":null,"REF":"A","ALT":["T"],"QUAL":999.0,"FILTER":["PASS"],"FORMAT":{"GT":[[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0]]}}
{"CHROM":"I","POS":105133,"ID":null,"REF":"G","ALT":["A"],"QUAL":999.0,"FILTER":["PASS"],"FORMAT":{"GT":[[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[1,1]]}}
{"CHROM":"I","POS":176422,"ID":null,"REF":"A","ALT":["G"],"QUAL":999.0,"FILTER":["PASS"],"FORMAT":{"GT":[[0,0],[0,0],[0,0],[0,0],[0,0],[0,0],[1,1],[0,0],[0,0],[0,0],[0,0],[1,1],[1,1],[null,null]]}}
```

The genotypes are ordered by sample, and the numbers correspond as follows as follows:

* `-1` → Missing
* `0` → Reference
* `1` → First ALT allele
* `2` → Second ALT allele
* `3` → etc.

You can also use `SGT` to outut a string representation of genotypes (e.g. "0/1"). It is also possible to use `TGT` to output the actual bases:

```bash
 sc json --format=TGT tests/data/test.vcf.gz
```
```json
{"CHROM":"I","POS":41947,"ID":null,"REF":"A","ALT":["T"],"QUAL":999.0,"FILTER":["PASS"],"FORMAT":{"TGT":["A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A","A/A"]}}
{"CHROM":"I","POS":105133,"ID":null,"REF":"G","ALT":["A"],"QUAL":999.0,"FILTER":["PASS"],"FORMAT":{"TGT":["G/G","G/G","G/G","G/G","G/G","G/G","G/G","G/G","G/G","G/G","G/G","G/G","G/G","A/A"]}}
{"CHROM":"I","POS":176422,"ID":null,"REF":"A","ALT":["G"],"QUAL":999.0,"FILTER":["PASS"],"FORMAT":{"TGT":["A/A","A/A","A/A","A/A","A/A","A/A","G/G","A/A","A/A","A/A","A/A","G/G","G/G","./."]}}
```