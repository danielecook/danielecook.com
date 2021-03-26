---
title: "A tool for writing TILs (Today I Learned)"
author: Daniel Cook
date: 2020-04-22T01:15:53Z
tags:
  - nim
---

There is a great repo on GitHub of [TILs](https://github.com/jbranchaud/til/blob/master/README.md). The author (jbranchaud) states:

> A collection of concise write-ups on small things I learn day to day across a variety of languages and technologies. These are things that don't really warrant a full blog post.

I think this is a pretty cool idea, but putting together the repo with the index can take time and interrupt your workflow. I wanted to make it very quick and easy to add TILs, so I wrote [TIL-Tool](https://www.github.com/danielecook/til-tool). __TIL-tool__ is a command line application invoked using `til` that makes it very easy to write TILs and generate an index.

To create a new TIL, run `til open topic/title`:

```bash
til open Python/list_comprehensions
```

This will open up a new text document which you can edit. Once you are done you can save the document and `til`. If you have configured a git remote (e.g. on GitHub), then you can then run `til push` which will build an index and push changes.

All TILs are stored in `~/.til`

* __[TIL-Tool on GitHub](https://github.com/danielecook/til-tool)__
* __[TIL Example Repo](http://www.github.com/danielecook/til)__


