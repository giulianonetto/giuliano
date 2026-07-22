---
title: "{here} and {argparse} bridge interactive analysis in R and Nextflow pipelines"
subtitle: ""
summary: "One small habit, argparse defaults set with here::here(), lets the same R script run interactively in the console and as a Nextflow process, with no rewrite in between."
authors:
- admin
tags:
- Nextflow
- R
- Reproducibility
- Bioinformatics
categories:
- Bioinformatic software engineering
date: "2026-07-22T00:00:00Z"
lastmod: "2026-07-22T00:00:00Z"
featured: false
draft: false
image:
  caption: ""
  focal_point: ""
  preview_only: false
projects: []
---

Reproducibility and pipeline management take time. When we're actually *doing* data analysis, though, we just want to focus on the analysis at hand, not the engineering overhead wrapped around it. In R, a simple trick that combines two small packages, [`argparse`](https://cran.r-project.org/package=argparse) and [`here`](https://here.r-lib.org/), lets us write code exactly the way we would in any interactive session, while still plugging straight into the reproducibility power of [Nextflow](https://www.nextflow.io/).

{{% alert note %}}
The trick is to set our `argparse` defaults with `here::here()`.
{{% /alert %}}

```r
#!/usr/bin/env Rscript

library(argparse)
library(here)
library(readr)

parser <- ArgumentParser()
parser$add_argument("--input_data", default = here::here("data/data.tsv"))
parser$add_argument("--output",     default = here::here("tmp/counts.tsv"))
args <- parser$parse_args()

data <- read_tsv(args$input_data)

data |>
  dplyr::count(group) |>
  write_tsv(args$output)
```

That's the whole idea. When we source or run this in the console, `parse_args()` sees no command-line arguments, so every option falls back to its default. Because the defaults are `here::here()` paths, they resolve relative to the **project root**, so `read_tsv(args$input_data)` just finds `data/data.tsv`, exactly as if we'd hard-coded it. Our normal develop-in-place loop is untouched. We point the output default at `tmp/`, which we `.gitignore`: throwaway results while we iterate, and nothing accidentally committed.

To turn the script into a pipeline step, we add the shebang line already visible at the top, make it executable, and drop it in the pipeline's `bin/` folder:

```bash
chmod +x bin/analyze.R
```

Nextflow automatically puts `bin/` on the `PATH`, so the script is callable by name from any process. The whole project stays small:

```
.
├── main.nf
├── bin/
│   └── analyze.R
└── data/
    └── data.tsv
```

A complete pipeline is then just a `main.nf` that wires an input file to the script:

```groovy
// main.nf
params.input_data = "data/data.tsv"

process ANALYZE {
    publishDir "results", mode: "copy"

    input:
    path input_data

    output:
    path "counts.tsv"

    script:
    """
    analyze.R --input_data ${input_data} --output counts.tsv
    """
}

workflow {
    ANALYZE(Channel.fromPath(params.input_data))
}
```

We run it with `nextflow run main.nf`. Now the process calls the same script, but passes the staged input **explicitly**. That `--input_data` value overrides the default, so `here::here()` is never consulted at runtime, which is exactly what we want, because inside a process the script runs in Nextflow's task work directory, where `here::here()` would point somewhere meaningless. The default only ever matters while we develop.

## The payoff

So the argparse default is the seam. `here::here()` makes the dev-time default project-relative and reproducible; Nextflow supplies the real path when it actually matters. In exchange we get the things pipelines are genuinely good at: provenance through git, isolation through containers, and `-resume` so we're not recomputing finished steps.

There are heavier options for R-native reproducibility. [`{targets}`](https://docs.ropensci.org/targets/) builds a proper dependency graph, and we can [render Rmarkdown reports straight from a process](https://www.solshenker.com/rmarkdown-nextflow-process/). But the nice part is that this one asks almost nothing of us: write the script the way we always would, and add one sensible default per argument.

If something here doesn't hold up, we'd love to hear about it.
