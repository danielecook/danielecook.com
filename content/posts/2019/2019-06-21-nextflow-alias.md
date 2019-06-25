---
title: Useful Nextflow bash functions for SLURM
author: Daniel Cook
layout: post
aliases:
  -  /nextflow-bash-aliases-for-slurm/
date: 2019-06-21
tags:
  - nextflow
  - gist
draft: false
---

If you use [Nextflow](http://www.nextflow.io) on a cluster with the SLURM scheduler, than these bash functions may be useful to you and worth sticking in your `.bashrc`.

{{< highlight bash "hl_lines=5-6,linenos=table" >}}
# Shortcut for going to work directories
# Usage: gw <workdir pattern>
# Replace the work directory below as needed
# Where workdir pattern is something like "ab/afedeu"
function gw {
        path=`ls --color=none -d /path/to/work/directory/$1*`
        cd $path
}

# sq squeue alternative
# Outputs more complete information about jobs including the work directory
function sq() {
    squeue --user `whoami` --format='%.18i %50j %10u %.10C %m %20J %M %.2t %n %R %Z' | awk -v OFS='\t' '{ match($10, /([a-f0-9]{2}\/[a-f0-9]{6})/, arr); print $1, $2, $3, $4, $5, $6, $7, $8, $9, arr[1] }' 
}
{{< /highlight >}}

Now, typing `sq` will give:

|    JOBID | NAME                                               | USER   |   CPUS | MIN_MEMORY   | THREADS_PER_CORE   | TIME   | ST   | REQ_NODES   |           |
|---------:|:---------------|:-------|-------:|:-------------|:-------------------|:-------|:-----|:------------|:----------|
| 17475076 | nf-fastq | cookd  |      8 | 12G          | *                  | 0:00   | PD   | (Priority)  | 04/fbbe3e |
| 17475077 | nf-fastq | cookd  |      8 | 12G          | *                  | 0:00   | PD   | (Priority)  | c9/9176eb |
| 17475078 | nf-fastq | cookd  |      8 | 12G          | *                  | 0:00   | PD   | (Priority)  | 80/6a233a |

And you can type `gw c9/9176eb` which will take you to the work directory for that job.


[gist](https://gist.github.com/danielecook/bae19b7b9191b76fb6972bd7ef16718d)