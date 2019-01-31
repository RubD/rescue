
<!-- README.md is generated from README.Rmd. Please edit that file -->
README
------

This package provides a bootstrap imputation method for dropout events in scRNAseq data.

Installation
------------

``` r
install.packages("devtools", repos="http://cran.rstudio.com/")
library(devtools)
devtools::install_github("seasamgo/rescue")
library(rescue)
```

Method
------

`bootstrapImputation` takes a log-normalized expression matrix and returns a list containing the imputed and original matrices.

``` r
bootstrapImputation(
  expression_matrix,                  # expression matrix
  select_cells = NULL,                # subset cells
  select_genes = NULL,                # informative genes
  proportion_genes = 0.6,             # proportion of genes to sample
  bootstrap_samples = 100,            # number of samples
  number_pcs = 8,                     # number of PC's to consider
  snn_resolution = 0.9,               # clustering resolution
  impute_index = NULL,                # specify counts to impute, defaults to zero values
  use_mclapply = FALSE,               # run in parallel
  cores = 2,                          # number of parallel cores
  return_individual_results = FALSE,  # return sample means
  verbose = FALSE                     # print progress to console
  )
```

Similar cells are determined with shared nearest neighbors clustering upon the principal components of informative gene expression (e.g. highly variable or differentially expressed genes). The names of these informative genes may be indicated with `select_genes`, which defaults to the most highly variable. For more, please view the help files.

Example
-------

To illustrate, we'll need the Splatter package to simulate some scRNAseq data.

``` r
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("splatter", version = "3.7")
library(splatter)
```

We'll consider a hypothetical example of 500 cells and 10,000 genes containing five distinct cell types of near equal size, then introduce some dropout events.

``` r
params <- splatter::newSplatParams(
  nGenes = 1e4,
  batchCells = 500,
  group.prob = rep(.2, 5),
  de.prob = .05,
  dropout.mid = rep(0, 5),
  dropout.shape = rep(-.5, 5),
  dropout.type = 'group',
  seed = 940
  )

splat <- splatter::splatSimulate(params = params, method = 'groups')

cell_types <- SummarizedExperiment::colData(splat)$Group
cell_types <- gsub('Group', 'Cell type ', cell_types)
```

To visualize this data we'll use the Seurat pipeline, which imports with the rescue package.

``` r
library(Seurat)
```

First, we should remove genes that lost all counts to dropout.

``` r
counts_true <- SummarizedExperiment::assays(splat)$TrueCounts
counts_dropout <- SummarizedExperiment::assays(splat)$counts
comparable_genes <- rowSums(counts_dropout) != 0
```

Next, we normalize and scale the data.

``` r
expression_true <- Seurat::CreateSeuratObject(raw.data = counts_true, normalization.method = 'LogNormalize', do.scale = TRUE, do.center = TRUE)
expression_dropout <- Seurat::CreateSeuratObject(raw.data = counts_dropout[comparable_genes, ], normalization.method = 'LogNormalize', do.scale = TRUE, do.center = TRUE)
```

The last step is dimension reduction with PCA and then visualization with t-SNE.

``` r
expression_true <- Seurat::RunPCA(expression_true, pc.genes = rownames(expression_true@data), pcs.print = 0)
expression_true <- Seurat::SetIdent(expression_true, ident.use = cell_types)
expression_true <- Seurat::RunTSNE(expression_true)
Seurat::TSNEPlot(expression_true)
```

![](README-unnamed-chunk-9-1.png)

``` r

expression_dropout <- Seurat::RunPCA(expression_dropout, pc.genes = rownames(expression_dropout@data), pcs.print = 0)
expression_dropout <- Seurat::SetIdent(expression_dropout, ident.use = cell_types)
expression_dropout <- Seurat::RunTSNE(expression_dropout)
Seurat::TSNEPlot(expression_dropout)
```

![](README-unnamed-chunk-9-2.png)

It's clear that dropout has distorted our evaluation of the data by cell type as compared to what we should see with the full set of counts. Now let's impute zero counts to recover missing expression values and reevaluate.

``` r
impute <- rescue::bootstrapImputation(expression_matrix = expression_dropout@data, bootstrap_samples = 100, use_mclapply = TRUE, cores = 3)

expression_imputed <- Seurat::CreateSeuratObject(impute$final_imputation)
expression_imputed <- Seurat::NormalizeData(expression_imputed, normalization.method = 'none')
expression_imputed <- Seurat::ScaleData(expression_imputed)
expression_imputed <- Seurat::RunPCA(expression_imputed, pc.genes = rownames(expression_imputed@data), pcs.print = 0)
expression_imputed <- Seurat::RunTSNE(expression_imputed)
expression_imputed <- Seurat::SetIdent(expression_imputed, ident.use = cell_types)
Seurat::TSNEPlot(expression_imputed)
```

![](README-unnamed-chunk-10-1.png)

The recovery of missing expression values due to dropout events allows us to correctly distinguish cell types with basic data visualization techniques in this simulated example data.
