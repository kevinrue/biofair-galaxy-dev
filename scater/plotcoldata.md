## Task

Write an implementation plan for Galaxy tool wrapper for the function `plotColData()` of the `scater` package.

Do not write the tool itself, but instead write a file `tools/scater-1.38.0/dev/plotcoldata.plan.md` that describes the a step-by-step process that I should follow to create that tool myself (e.g., files to be created or edited, contents, planemo commands to run).

## Functionality

The `object` input should be a `SingleCellExperiment` object that is given to the Galaxy tool as a `loom` file.

The `y` input is column-level metadata field to show on the y-axis that should be given to the Galaxy tool as a simple text string.

The optional `x` input can be given following the same rule as the `y` input.

## Examples

If only `y` is supplied, the command would look like this:

```r
scater::plotColData(
  object = sce,
  y = "detected"
)
```

If both `y` and `x` are supplied, the command would look like this:

```r
scater::plotColData(
  object = sce,
  y = "detected",
  x = "sum"
)
```

## Test data

Use <https://zenodo.org/records/19665848/files/sce_after_addpercellqcmetrics.loom> in planemo tests.

## Constraints

- Do not declare indirect R depedendencies unless required.
- For Bioconductor dependencies, use package versions from Bioconductor release `3.22`, that is scater version `1.38.0`.
- Ignore the existing Galaxy tool under `tools/scater` and work on the assumption that we start a new one under `tools/scater-1.38.0`
- Do not use dummy placeholder values in tests; these cause linting errors. Set a random arbitrary value that I'll fix based on the report of the first failed test.

## Claude

Use the `galaxy/tool-dev` skill if available.
