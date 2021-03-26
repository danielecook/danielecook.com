---
title: "A bash alias for Microsoft Excel (Mac only)"
date: 2019-06-28T00:26:29+01:00
draft: false
tags:
  - bash
  - gist
  - excel
---

Years ago I wrote a function for [opening excel from R](/an-r-function-for-opening-a-dataframe-in-excel-mac-only/). While I would never use Excel for data analysis, it turns out it's pretty good for sorting and browsing data. Thats why I wrote a simple bash alias for opening up text documents from the terminal.

```bash
function excel() {
    tmp=`mktemp`
    out=${1}
    cat ${out} > $tmp
    open -a "Microsoft Excel" $tmp
}
```

Usage:

```bash
cat spreadsheet.tsv | excel
```