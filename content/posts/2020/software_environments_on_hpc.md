---
title: "Managing software environments on HPC systems"
author: Daniel Cook
date: 2020-07-08T01:15:53Z
draft: true
tags:
  - presentation
---

The ability to control software environments is essential for reproducible and robust data analysis. A number of tools exist for managing software environments that can be run on  High-performance computing (HPC) systems. Below I detail a few of them.

# Replication and Portability

The [Replication crisis](https://en.wikipedia.org/wiki/Replication_crisis) is an ongoing problem in which the results of many studies cannot be reproduced. Surprisingly, many studies that rely on computational methods also are not reproducible even though code, data, and software can be packaged and distributed for others to evaluate [^1].

A good ideal to shoot for is to make computational experiments __portable__. Being able to move code, data, and software to a new system with limited friction means you have a good, reproducible experiment. Recently developed tools make this relatively easy on HPC systems.

# Linuxbrew

Homebrew is a package manager for MacOS, and [Linuxbrew](https://docs.brew.sh/Homebrew-on-Linux) is a fork developed for Linux. You can use Linuxbrew to install software in HPC environments, but a lot of the software you install will not be usable by others. The reason for this is that Linuxbrew is designed to manage software for individual users. It stores shared libraries, binaries, and other dependencies in the home directory (`~/.linuxbrew`).

I mention Linuxbrew here though because it is still really useful in HPC environments to be able to install software for testing purposes. In fact you can use it to install utilities like [autojump](https://github.com/wting/autojump), [direnv](https://direnv.net/), git, [pyenv](https://github.com/pyenv/pyenv), and others that can be very beneficial in HPC environments.


# Executable binaries

When you run a command-line interface (CLI)  program in an HPC environment, you are often running an [executable binary](https://en.wikipedia.org/wiki/Executable). You can see the location of that binary by typing:

```bash
which bedtools # /software/bedtools/2.29.1/bin/bedtools
```

Binaries are platform-specific, meaning you need to compile them or get a version compiled for your platform in order for them to work. Fortunately, the vast majority of HPC systems run linux. However, executable binaries can also use dynamic linking which can complicate their portability.

## Dynamically linked binaries

Binaries often rely on shared libraries. Shared libraries are used by multiple programs and are said to be 'dynamically linked' because they are loaded at runtime.

On linux, shared libraries end with a `.so` extension [^2], and they are often stored in the `/lib` or `/lib/lib64`. These are generally off limits to users in HPC environments, so this can complicate the portability of your work because shared libraries differ from system to system.

One workaround is to bundle a binary with the required shared libraries, and append the directory of shared libraries to the `LD_LIBRARY_PATH` environmental varialbe. `LD_LIBRARY_PATH` is used to specify additional locations for shared libraries.

The bottom line is that dynamically linked binaries can complicate portability of your work - making it more difficult to distribute.

## Statically linked binaries

Some executable binaries are statically linked. This means that all the dependencies are bundled into the binary itself when it is compiled.

A good example of this is [somalier](https://www.github.com/brentp/somalier). __somalier__ includes all the dependent libraries. All you have to do is download it and it will run on your Linux-based HPC environment!

## Java .jar files

Java programs can be compiled to `.jar` (java archive) files. These offer an advantage over traditional binaries in that they do not need to be compiled for a specific machine. Instead, they run using the Java Virtual Machine:

```bash
java -jar gatk.jar <options>
```

The downside of Jars is that they may be dependent on the version of Java you are running. 

## The `$PATH` variable

On Unix systems, the `$PATH` variable is used to specify the location of binaries. For example, if you have downloaded `somalier` to your home folder and want it to be globally accessible, you can do so by appending the directory to your PATH variable.

```bash
~/my_software/somalier # location of somalier
export PATH=${PATH}:~/mysoftware/somalier
```

## Summary of binaries

Binaries and `.jar` files can be used to distribute a software environment on their own. A good way of doing this might be to write a script that retrieves and compiles the required software, but if many tools are being used this approach can quickly become difficult to manage.

# The `module` command

[module](https://linux.die.net/man/1/module) is a program often found on HPC systems that allows you to load software. You load programs by typing something like:

```bash
module load bcftools/1.10.1
```

You can also load the latest version of a module by leaving off the version, but this is bad practice, because results can change when software is updated and your work is no longer reproducible.

```bash
module load bcftools
```

Additional commands you should be aware of:

```bash
module avail # Lists all available software
module list  # Shows loaded modules
```

The module command is often run in analysis scripts. For example, on a SLURM cluster you might have a script that looks like this:

```bash
#!/usr/bin/bash
# Author: Daniel E. Cook
#SBATCH --job-name bcftools_analysis
#SBATCH --part=cpu
#SBATCH --time=2:00:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=4G
#SBATCH -o %j.out
#SBATCH -e %j.err
module load bcftools/1.10.1
module load bedtools/2.29.1
bcftools call ...
bedtools ...
```

## modulefiles

Modules are defined using `modulefiles`, which basically alters your shell environment to make make software accessible.

The contents of a module can be shown by typing `module show <module name>`.

```bash
>>> module show bedtools/2.29.1
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
   /software/Modules/3.2.9/modulefiles/bedtools/2.29.1:
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
whatis("Sets up bedtools/2.29.1 ")
setenv("BEDTOOLS_DIR","/software/bedtools/2.29.1/bin")
setenv("BEDTOOLS_HOME","/software/bedtools/2.29.1")
prepend_path("PATH","/software/bedtools/2.29.1/bin")
help([[This module adds bedtools/2.29.1 to various paths
]])
```

If you are lucky, your HPC administrators will manage installation of software and you will not have to write any `modulefiles` yourself. However, there are scenerios where you might choose to do so. For example, if you work in a lab and want to share a program program with other lab members, you can write a module for the group. The software and `modulefile` would be stored in your shared disk space, and everyone would have to add the directory of the `modulefile` to an environmental variable called `MODULEPATH` in order to load it.

## Should I distribute software environments using `module`?

Modules are generally not portable - meaning you cannot use the same software locally or even on another HPC cluster. Even when two clusters have the same software and versions you still might not be able to load the software correctly with the same script because modules are often named arbitrarily and are case sensitive.

For example, I have worked on two HPCs with `bedtools` installed but I have to load it differently on each one:

```
module load bedtools/2.29.1 # HPC 1
module load BEDtools/2.29.1 # HPC 2
```

It's a good idea to just use lowercase for module names to avoid confusion.

__Other issues with modules__

* Your HPC admin might remove or update modules. This can break all of your scripts.
* Loading multiple modules can unload and/or load different versions of dependencies

My recommendation: do not use `module` if you intend to publish analysis scripts, or need your analysis to be reproducible outside of your current HPC environment. Your work is not reproducible if your scripts cannot be run by someone else on their own cluster.

# conda

> "Conda is an open source package management system and environment management system for installing multiple versions of software packages and their dependencies and switching easily between them. It works on Linux, OS X and Windows, and was created for Python programs but can package and distribute any software." Source: [Anaconda website](https://docs.conda.io/en/latest/)

Conda is distributed in a couple of different ways:

* Miniconda = conda + Python
* Anaconda = conda + Python + ~160+ data-science packages

## conda channels

Packages are stored and retrieved from "channels." Examples of channels include [bioconda](https://bioconda.github.io/), and [conda-forge](https://conda-forge.org).

When installing software, conda will look for software in a predetermined set of channels, using the first match. A good set of default channels to use is:

```
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
```

[bioconda](https://bioconda.github.io/) is a channel specializing in bioinformatics software, and [conda-forge](https://conda-forge.org) is a community managed channel of software.

These channels contain thousdands of software packages. You can search for packages on their sites.

## Organizing software environments

A key problem that conda helps solve for computational analyses is that it allows you to isolate software environments and design them to cater to a specific analysis.  If you have ever tried to maintain a single software environment for multiple projects you will quickly run into "dependency hell."

Conda environments are similar to docker/singularity containers which we will explore later on.

## conda environments

Lets create a conda environment.

```bash
conda create -n variant_calling bcftools=1.10.1 bedtools=2.29.2
```

Also note that you can explicitely define the desired channel by prefixing a package with the channel name and two colons (e.g. `bioconda::bcftools=1.10.1`)

Now, to "turn on" a conda environment you can activate it:

```bash
conda activate variant_calling
```

`bcftools` and `bedtools` should now be available.

## Distributing conda environments

You can "export" a conda environment using this command:

```bash
conda env export -n variant_calling > variant_calling.yml
```

The exported environment looks like this:

```yamlname: variant_calling
channels:
  - ggd-genomics
  - conda-forge
  - bioconda
  - defaults
dependencies:
  - bcftools=1.10.1=h27336cd_0
  - bedtools=2.29.2=h37cfd92_0
  - bzip2=1.0.8=h0b31af3_2
  ...
```

Store this `.yml` file with your code so others can easily recreate the software environment you used!

## conda and R environments

Many bioinformaticians and data scientists rely on a combination of R, Python, and Bash. Did you know that conda can be used to manage R environemnts?!

Most R packages are prefixed with `r-`. For example, you can install the [R tidyverse](https://www.tidyverse.org/) package using a command like:

```bash
conda create -n Renv r-tidyverse=1.2.1
```

Now you can build R libraries specific to your project!

## Should I distribute software environments using conda?

Yes! Conda is a great solution, but like most solutions it has some drawbacks. Conda can ta
ke a very long time to resolve environments. And in some cases it will fail due to dependency issues.

Additionally, conda creates software environments specific to a platform - and if no distribution for your platform exists you may be out of luck. This can be the case when a linux-version of a package exists but not one for MacOS or windows. This can make it difficult to debug or run analyses locally before running it an HPC environment.

# Docker

Next up lets talk about Docker. Docker is a software package that uses OS-level virtualization to run "containers." Containers are packaged software-environments.

In general, I have had less issues with Docker than Conda, but Docker tends to be more complicated.

## Docker environments

Docker environments are specified using `.Dockerfiles`.



# singularity


---

[^1]: [Sustainable computational science:
the ReScience initiative](https://www.arxiv-vanity.com/papers/1707.04393)
[^2]: On windows they end in `.dll`, and on macos they end in `.dylib`.