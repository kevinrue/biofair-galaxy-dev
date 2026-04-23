## BioFAIR Pathfinder - Connecting Bioconductor, Galaxy, and nf-core

This repository documents plans and implementation notes for the development of Galaxy tools for Bioconductor software for single-cell analysis.

## Context

The tools being implemented in this project are designed to combine into a workflow demonstrated in a new GTN tutorial that will mirror existing GTN tutorials using [Seurat](https://training.galaxyproject.org/training-material/topics/single-cell/tutorials/scrna-seurat-pbmc3k/tutorial.html) and [Scanpy](https://training.galaxyproject.org/training-material/topics/single-cell/tutorials/scrna-scanpy-pbmc3k/tutorial.html) GTN tutorials.

## Tools

In order of usage within a workflow:

`singlecellexperiment_matchfeatures`
Produce a text file listing the identifiers of genes that match a set of conditions.
In this case, 'identifiers' refers to `rownames` and conditions might be "gene symbol matches a pattern".

`scuttle_addpercellqcmetrics`
Adds per-cell quality metrics in the `colData` component of a `SingleCellExperiment` object.
This tool accepts optional lists of genes (e.g. mitochondrial) given as separate text files.

`singlecellexperiment_display`
Produce a text file that display aspects of a `SingleCellExperiment`.
In this case, 'aspects' might be the `colData` or `rowData`.
Future plan includes the full object's summary view, as in `print(sce`.

## News

21 April 2026 [BioFAIR Pathfinder Projects Launch with £800k to Transform UK FAIR Practices](https://biofair.uk/updates/2026/biofair-pathfinder-projects-launch-with-800k-to-transform-uk-fair-practices/)

October 2025 [Pathfinder Projects](https://biofair.uk/pathfinder-projects/)
