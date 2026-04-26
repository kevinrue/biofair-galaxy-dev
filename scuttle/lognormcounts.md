## Task

Write an implementation plan for Galaxy tool wrapper for the function `logNormCounts()` of the `scuttle` package.

Do not write the tool itself, but instead write a file `tools/scuttle-1.20.0/dev/lognormcounts.plan.md` that describes the a step-by-step process that I should follow to create that tool myself (e.g., files to be created or edited, contents, planemo commands to run).

## Functionality

The `object` input should be a `SingleCellExperiment` object that is given to the Galaxy tool as a `loom` file.

Start with the simplest wrapper that does not offer any of the optional arguments.

Only offer the typical Galaxy 'advanced common' argument to optionally output a log file.

## Examples

The core command would look like this:

```r
sce <- scuttle::logNormCounts(x = sce)
```

Then the tool should write the output `sce` object to a Loom file.

## Test data

Use <https://zenodo.org/records/19665848/files/sce_after_addpercellqcmetrics.loom> in planemo tests.

## Constraints

- Do not declare indirect R depedendencies unless required.
- For Bioconductor dependencies, use package versions from Bioconductor release `3.22`, that is scuttle version `1.20.0`.
- Do not use dummy placeholder values in tests; these cause linting errors. Set a random arbitrary value that I'll fix based on the report of the first failed test.

## Claude

Use the `galaxy/tool-dev` skill if available.
