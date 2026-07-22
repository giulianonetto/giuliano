---
title: "{here} and {argparse} bridge interactive analysis in R and Nextflow pipelines"
subtitle: ""
summary: "One small habit, argparse defaults set with here::here(), lets the same R script run interactively in your console and as a Nextflow process, with no rewrite in between."
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

Reproducibility and pipeline management take time. When I'm actually *doing* data analysis, though, I just want to focus on the analysis at hand, not the engineering overhead wrapped around it. In R, a simple trick that combines two small packages, [`argparse`](https://cran.r-project.org/package=argparse) and [`here`](https://here.r-lib.org/), lets me write code exactly the way I would in any interactive session, while still plugging straight into the reproducibility power of [Nextflow](https://www.nextflow.io/).

{{% alert note %}}
The trick is to set your `argparse` defaults with `here::here()`.
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

That's the whole idea. When I source or run this in the console, `parse_args()` sees no command-line arguments, so every option falls back to its default. Because the defaults are `here::here()` paths, they resolve relative to my **project root**, so `read_tsv(args$input_data)` just finds `data/data.tsv`, exactly as if I'd hard-coded it. My normal develop-in-place loop is untouched. I point the output default at `tmp/`, which I `.gitignore`: throwaway results while I iterate, and nothing accidentally committed.

To turn the script into a pipeline step, I add the shebang line you can already see at the top, make it executable, and drop it in the pipeline's `bin/` folder:

```bash
chmod +x bin/analyze.R
```

Nextflow puts `bin/` on the `PATH`, so the script is callable by name from any process:

```groovy
process ANALYZE {
    input:
    path input_data

    output:
    path "counts.tsv"

    script:
    """
    analyze.R --input_data ${input_data} --output counts.tsv
    """
}
```

Now Nextflow calls the same file, but passes the staged input **explicitly**. That `--input_data` value overrides the default, so `here::here()` is never consulted at runtime, which is exactly what you want, because inside a process the script runs in Nextflow's task work directory, where `here::here()` would point somewhere meaningless. The default is only ever for me, at my desk.

## Reusing the interface across scripts

Once you have more than one script, you don't want to redefine the same arguments in each. Factor the parser into a small module, say `R/cli.R`, and expose it as a function:

```r
# R/cli.R: shared command-line interface

get_parser <- function() {
  parser <- argparse::ArgumentParser()
  parser$add_argument("--input_data", default = here::here("data/data.tsv"))
  parser$add_argument("--output",     default = here::here("tmp/counts.tsv"))
  parser
}
```

Then each analysis script pulls it in with [`import`](https://cran.r-project.org/package=import), which sources just the object you ask for into the current environment. Unlike the argument *defaults*, though, this call actually runs and has to find `cli.R`. Inside a Nextflow task, your project's `R/` isn't on the filesystem. The Nextflow-native fix is to stage the module the same way you stage any input: declare it and pass it from `projectDir`.

```groovy
process ANALYZE {
    input:
    path input_data
    path cli            // R/cli.R, staged into the task work dir

    output:
    path "counts.tsv"

    script:
    """
    analyze.R --input_data ${input_data} --output counts.tsv
    """
}

workflow {
    cli = file("${projectDir}/R/cli.R")
    ANALYZE(input_ch, cli)
}
```

Nextflow copies `cli.R` into the task's working directory, so the script finds it right there; at your desk it isn't in the working directory, so `here()` locates it in the project instead. One line picks the right one:

```r
#!/usr/bin/env Rscript

cli <- if (file.exists("cli.R")) "cli.R" else here::here("R", "cli.R")
import::here("get_parser", .from = cli, .character_only = TRUE)

args <- get_parser()$parse_args()

data <- readr::read_tsv(args$input_data)

data |>
  dplyr::count(group) |>
  readr::write_tsv(args$output)
```

(The `.character_only = TRUE`, and the quoted `"get_parser"`, are what make `import` use the *value* of `cli` rather than the literal symbol. See the [`import` docs](https://rticulate.github.io/import/).)

Staging keeps the shared code versioned with the pipeline and available on every executor (local, container, HPC, or cloud) with no `.here` file or image surgery. Baking it into the container image or installing your project as a package works too, if you prefer.

## The payoff

So the argparse default is the seam. `here::here()` makes the dev-time default project-relative and reproducible; Nextflow supplies the real path when it actually matters. In exchange I get the things pipelines are genuinely good at: provenance through git, isolation through containers, and `-resume` so I'm not recomputing finished steps.

There are heavier options for R-native reproducibility. [`{targets}`](https://docs.ropensci.org/targets/) builds a proper dependency graph, and you can [render Rmarkdown reports straight from a process](https://www.solshenker.com/rmarkdown-nextflow-process/). But I like that this one asks almost nothing of me: write the script the way I always would, and add one sensible default per argument.

If you spot a hole in this, I'd love to hear it.
