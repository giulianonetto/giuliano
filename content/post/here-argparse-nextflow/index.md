---
title: "Integrating interactive data analysis in R with Nextflow pipelines"
subtitle: ""
summary: "A simple workflow for leveraging the power of Nextflow pipelines while still keeping your data analysis in R easily interactive."
authors:
- admin
tags:
- Nextflow
- R
- Reproducibility
- Bioinformatics
categories:
- Bioinformatics software engineering
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

Most data analysis in R is done interactively. At some point, you may want this analysis to run as a reproducible [Nextflow](https://www.nextflow.io/) pipeline. You carefully craft a structured script (e.g., with no hard-coded paths). By then, your code is reproducible, but often not so interactive anymore (e.g., lots of command-line options floating around). Iterating between Nextflow runs and analysis development becomes a bit of a headache. Here is a quick recipe to make your data analysis in R ready for Nextflow, while still supporting your regular interactive workflow.

## The analysis script

We write the script the way we always would, and give it a small command-line interface with [`argparse`](https://cran.r-project.org/package=argparse). The one detail worth adding is to set each argument's default with [`here::here()`](https://here.r-lib.org/), so the default paths always resolve from the project root.

```r
#!/usr/bin/env Rscript
library(tidyverse)
library(argparse)
library(here)

parser <- ArgumentParser()
parser$add_argument("--input_data", default = here::here("data/data.tsv"))
parser$add_argument("--output_dir", default = here::here("tmp"))
args <- parser$parse_args()

dir.create(args$output_dir, showWarnings = FALSE, recursive = TRUE)

data <- read_tsv(args$input_data)

data %>%
  dplyr::count(group) %>%
  write_tsv(file.path(args$output_dir, "counts.tsv"))
```

Run this in the R console and `parse_args()` sees no command-line arguments, so every option falls back to its default.

Because the defaults use `here::here()`, they resolve from the project root regardless of our current working directory, so `read_tsv(args$input_data)` just finds `<project-root>/data/data.tsv`. We also point the outputs to a temporary directory (`<project-root>/tmp/`) by default, as the final outputs will be created reproducibly in Nextflow later.

Without `here::here()`, the default would evaluate to `<working-directory>/data/data.tsv` at runtime, which would only work if our working directory in R happened to match the project root. Anchoring to the project root explicitly keeps each path stable.

Our interactive data analysis workflow is then untouched: we can continue analyzing data using the above script as usual.

## The Nextflow pipeline

A minimal Nextflow pipeline that runs this one step is a single `main.nf`:

```groovy
// main.nf
params.input_data = "data/data.tsv"

process ANALYZE {
    input:
    path input_data

    output:
    path "counts.tsv"

    script:
    """
    analyze.R --input_data ${input_data} --output_dir "."
    """
}

workflow {
    ANALYZE(Channel.fromPath(params.input_data))
}
```

The process calls `analyze.R` and passes the input and output paths explicitly on the command line.

## Putting the script in the pipeline

For Nextflow to find the script, we add the shebang line (`#!/usr/bin/env Rscript`, already at the top of the R script above), make it executable, and drop it in the pipeline's `bin/` folder:

```bash
chmod +x bin/analyze.R # make it executable
```

Nextflow automatically puts `bin/` on the `PATH`, so the process can call `analyze.R` by name. The whole project now looks like the following:

```
.
├── main.nf
├── bin/
│   └── analyze.R
└── data/
    └── data.tsv
```

We run it with `nextflow run main.nf`. Now the same script runs inside the pipeline, except Nextflow passes the input and output paths explicitly (`--input_data` and `--output_dir`), overriding the `here::here()` defaults. Nothing in the script changed: the defaults serve our interactive session, and Nextflow supplies the real paths at runtime.

That is the whole integration. We keep writing data analysis code exactly as before while leveraging the power of Nextflow.