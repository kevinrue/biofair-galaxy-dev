## Prompt

I want to write a Galaxy tool wrapper for a function of the Bioconductor package 'scuttle' (version 1.20.0; Bioconductor release 3.22).

### Constraints
- You are only allowed to edit this file. Lay out all instructions (files to create and their full contents), but leave it to me to apply the changes.
- Use `tools/seurat/neighbors_clusters_markers.xml` as a structural template: embed all R code directly inside the XML in a `<configfile>` block; do not write external R scripts.
- Do not use `--conda_dependency_resolution` in any planemo command.
- `planemo serve` requires a pre-existing Galaxy checkout via `--galaxy_root`; planemo cannot build the Galaxy client from scratch in most environments.
- Allow both the Galaxy datatypes `rdata.sce` and `rds` to describe input RDS files that contain a `SingleCellExperiment` object.
- Use the the Galaxy datatypes `rdata.sce` to describe output RDs files that contain a `SingleCellExperiment` object.
- Do not declare any indirect dependency unless explicitly requested (e.g. `bioconductor-singlecellexperiment`)

### First tool: `addPerCellQCMetrics`
Wrap `scuttle::addPerCellQCMetrics()`, which takes a `SingleCellExperiment` object (RDS input) and returns the same object with QC metrics added to `colData` (RDS output, Galaxy format `rds`).

Expose only these arguments:
- `x` — RDS file input
- `subsets` — built from a `<repeat>` of name/filepaths pairs, each matched against a user-chosen `rowData` column. The files should be plain text files with the rownames of features to include in the set.
- `assay.type` — free-form text field, default `'counts'`

Leave all other arguments (`use_altexps`, `flatten`, etc.) at their R defaults.

### Test data
Test files will be hosted on Zenodo; do not commit them. Use `value=` attributes in `<param>` elements for now (local files); note that these will be replaced with `location=` Zenodo URLs in due course. After running `planemo test` once, hardcode the observed output sizes into `<has_size>` assertions.

### Planemo commands
Include commands for linting, programmatic testing, and interactive testing (planemo serve).

Do not suggest the option `--galaxy_root`. Assume this is configured in `~/.planemo.yml`.

---

# Galaxy Tool: scuttle 1.20.0 — `addPerCellQCMetrics`

## Approach

Each tool in this suite:
- accepts/returns a `SingleCellExperiment` object serialised as an RDS file (Galaxy format `rdata.sce`)
- embeds all R code in a `<configfile name="script_file">` block using Cheetah template syntax (`#for`/`#end for`, `#if`/`#end if`)
- exposes `@CMD@`, `@CMD_imports@`, `@CMD_read_inputs@`, `@CMD_rds_write_outputs@` macro tokens for boilerplate — mirroring `tools/seurat/macros.xml`
- one `.xml` file per tool function; shared macros live in `macros.xml`
- test data in `test-data/` is local only, not committed; will be uploaded to Zenodo

## Directory layout

```
tools/scuttle-1.20.0/
├── macros.xml
├── scuttle_addpercellqcmetrics.xml
├── test-data/
│   └── sce_input.rds          # minimal SCE (~50 cells, ~200 genes); not committed
├── .shed.yml
└── dev-scuttle-galaxy/plan.md # this file
```

---

## Step 0 – Understand `addPerCellQCMetrics`

`scuttle::addPerCellQCMetrics()` computes per-cell
summary statistics and appends them to `colData(sce)`:

| Added column | Meaning |
|---|---|
| `sum` | total counts per cell |
| `detected` | number of genes with count > 0 |
| `subsets_<name>_sum` | total counts in gene subset |
| `subsets_<name>_detected` | detected genes in subset |
| `subsets_<name>_percent` | % of total counts in subset |

Key parameters to expose in Galaxy:

| R argument | Galaxy param type | Notes |
|---|---|---|
| `x` | `data` (rdata.sce) | input SCE |
| `subsets` | `repeat` of name + `data` (txt) pairs | file contains rownames, one per line, no header |
| `assay.type` | `text` | assay to use; default `"counts"` |

---

## Step 1 – Create test data

Run the following once in R (inside `dev-scuttle-galaxy/`) to create a minimal
input file.  The `all.qmd` prototype already has the download code; adapt it:

```r
library(DropletUtils)
library(scuttle)

# Read the 10x dataset already in dev-scuttle-galaxy/data/
sce <- read10xCounts("data")

# Keep only first 200 genes and 50 cells to keep file small
set.seed(42)
sce <- sce[1:200, sample(ncol(sce), 50)]

saveRDS(sce, "../test-data/sce_input.rds")

# Run the tool once via planemo to discover the output sizes, then hardcode
# them in the <has_size> assertions in scuttle_addpercellqcmetrics.xml.
```

Verify test file is small enough (`ls -lh test-data/`). Do **not** commit
the RDS file — it will be hosted on Zenodo. Update the `<param ... location="..."/>`
attribute in the test section of the XML once the Zenodo record is created.

`test-data/mitochondrial_gene_ids.txt` is a committed plain-text file of
rownames used as the MT subset in test 2.  Only `*.rds` is gitignored; commit
this file.

---

## Step 2 – Write `macros.xml`

The macros mirror the Seurat pattern exactly, substituting the SCE object
and scuttle library.

```xml
<?xml version="1.0"?>
<macros>
    <token name="@TOOL_VERSION@">1.20.0</token>
    <token name="@VERSION_SUFFIX@">0</token>
    <token name="@PROFILE@">23.0</token>

    <xml name="requirements">
        <requirements>
            <requirement type="package" version="@TOOL_VERSION@">bioconductor-scuttle</requirement>
            <requirement type="package" version="1.28.0">bioconductor-loomexperiment</requirement>
            <yield/>
        </requirements>
    </xml>

    <!-- Mirrors @CMD@ in seurat/macros.xml -->
    <token name="@CMD@"><![CDATA[
cp '$input_loom' sce.loom &&
cat '$script_file' > '$hidden_output' &&
Rscript '$script_file' >> '$hidden_output'
    ]]></token>

    <!-- Mirrors @CMD_imports@ in seurat/macros.xml -->
    <token name="@CMD_imports@"><![CDATA[
library(scuttle)
library(LoomExperiment)
    ]]></token>

    <!-- Mirrors @CMD_read_inputs@ in seurat/macros.xml -->
    <token name="@CMD_read_inputs@"><![CDATA[
sce <- import('sce.loom', format = "loom", type = "SingleCellLoomExperiment")
    ]]></token>

    <!-- Mirrors @CMD_loom_write_outputs@ in seurat/macros.xml -->
    <token name="@CMD_loom_write_outputs@"><![CDATA[
export(sce, "match-feature-names/sce.loom", format = "loom")
    ]]></token>

    <xml name="input_sce">
        <param name="input_loom" type="data" format="loom"
               label="Input SingleCellExperiment (loom)"/>
    </xml>

    <xml name="output_sce">
        <data name="output_loom" format="rdata.sce" from_work_dir="sce.loom"
              label="${tool.name} on ${on_string}: loom"/>
        <expand macro="outputs_common_advanced"/>
    </xml>

    <xml name="inputs_common_advanced">
        <section name="advanced_common" title="Advanced Output" expanded="false">
            <param name="show_log" type="boolean" checked="false"
                   label="Output log?"/>
        </section>
    </xml>

    <xml name="outputs_common_advanced">
        <data name="hidden_output" format="txt"
              label="${tool.name} on ${on_string}: log">
            <filter>advanced_common['show_log']</filter>
        </data>
    </xml>

    <xml name="citations">
        <citations>
            <citation type="doi">10.1093/bioinformatics/btw777</citation>
        </citations>
    </xml>
</macros>
```

---

## Step 3 – Write `scuttle_addpercellqcmetrics.xml`

All R logic lives in `<configfile name="script_file">`.  The `<command>`
block contains only `@CMD@`.  A Cheetah `#for` loop builds the `subsets` list
from the `<repeat>` inputs, following the pattern in `seurat/neighbors_clusters_markers.xml`.

```xml
<?xml version="1.0"?>
<tool id="scuttle_addpercellqcmetrics" name="Scuttle: Add per-cell QC metrics"
      version="@TOOL_VERSION@+galaxy@VERSION_SUFFIX@" profile="@PROFILE@">
    <description>to a SingleCellExperiment object</description>
    <macros>
        <import>macros.xml</import>
    </macros>
    <expand macro="requirements"/>
    <command detect_errors="exit_code"><![CDATA[
@CMD@
    ]]></command>
    <configfiles>
        <configfile name="script_file"><![CDATA[
@CMD_imports@
@CMD_read_inputs@

subsets <- list()
#for $s in $subsets_repeat
subsets[['${s.name}']] <- which(rownames(sce) %in% readLines('$s.feature_file'))
#end for

sce <- scuttle::addPerCellQCMetrics(
    x          = sce,
    subsets    = subsets,
    assay.type = '$assay_type'
)

@CMD_loom_write_outputs@
        ]]></configfile>
    </configfiles>
    <inputs>
        <expand macro="input_sce"/>
        <param name="assay_type" type="text" value="counts"
               label="Assay to use for QC computation"/>
        <repeat name="subsets_repeat" title="Gene subset" min="0">
            <param name="name" type="text" label="Subset name"
                   help="Short label, e.g. MT"/>
            <param name="feature_file" type="data" format="txt"
                   label="Feature list"
                   help="Plain text file with one rowname per line, no header"/>
        </repeat>
        <expand macro="inputs_common_advanced"/>
    </inputs>
    <outputs>
        <expand macro="output_sce"/>
    </outputs>
    <tests>
        <test expect_num_outputs="2">
            <!-- test1: no subsets -->
            <param name="input_loom" location="https://zenodo.org/records/19665848/files/sce_with_mitochondrial_features.loom" ftype="loom"/>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="addPerCellQCMetrics"/>
                </assert_contents>
            </output>
            <output name="output_loom" ftype="loom">
                <assert_contents>
                    <has_size size="1191" delta="10"/>
                </assert_contents>
            </output>
        </test>
        <test expect_num_outputs="2">
            <!-- test2: MT subset -->
            <param name="input_loom" location="https://zenodo.org/records/19665848/files/sce_with_mitochondrial_features.loom" ftype="loom"/>
            <repeat name="subsets_repeat">
                <param name="name" value="MT"/>
                <param name="feature_file" value="mitochondrial_gene_ids.txt" ftype="txt"/>
            </repeat>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="addPerCellQCMetrics"/>
                </assert_contents>
            </output>
            <output name="output_loom" ftype="loom">
                <assert_contents>
                    <has_size size="1322" delta="10"/>
                </assert_contents>
            </output>
        </test>
    </tests>
    <help><![CDATA[
**What it does**

Runs ``scuttle::addPerCellQCMetrics()`` on a ``SingleCellExperiment`` object
and returns the same object with new columns appended to ``colData``:

- ``sum`` – total counts per cell
- ``detected`` – number of detected features per cell
- ``subsets_<name>_sum``, ``subsets_<name>_detected``, ``subsets_<name>_percent``
  for each user-defined gene subset (e.g. mitochondrial genes)

**Inputs**

- A ``SingleCellExperiment`` object saved as a loom file.
- Optionally, one or more named gene subsets, each defined by a plain text file
  listing the rownames to include (one per line, no header).

**Outputs**

- The same ``SingleCellExperiment`` with QC columns added to ``colData``,
  saved as a loom file.
    ]]></help>
    <expand macro="citations"/>
</tool>
```

---

## Step 4 – Lint and test with Planemo

```bash
# From the tools/scuttle-1.20.0/ directory:

# Lint XML
planemo lint scuttle_addpercellqcmetrics.xml

# Run tests (builds Conda env on first run – slow)
planemo test scuttle_addpercellqcmetrics.xml

# Serve locally to click through the Galaxy UI
# Requires a pre-existing Galaxy checkout; planemo cannot build the client from scratch.
planemo serve scuttle_addpercellqcmetrics.xml
```

---

## Step 5 – Write `.shed.yml`

```yaml
categories:
  - Transcriptomics
  - Single-cell
description: Galaxy wrappers for the Bioconductor scuttle package (v1.20.0)
long_description: |
  scuttle provides utility functions for single-cell RNA-seq analysis,
  including quality control, normalisation, and transformation, built
  around the SingleCellExperiment class.
name: scuttle
owner: iuc
```

---

## Step 6 – Iterate for subsequent tool functions

Repeat Steps 1–4 for each new tool.  Each tool gets its own `.xml` with a
`<configfile>` block; shared tokens and macros grow in `macros.xml`.

Suggested order following the scuttle functions used in `dev-scuttle-galaxy/all.qmd`:

1. `scuttle_addpercellqcmetrics` ← **this document**
2. `scuttle_lognormcounts` – library-size normalisation (`scuttle::logNormCounts`)
3. `scuttle_addperfeatureqcmetrics` – per-feature QC metrics (`scuttle::addPerFeatureQCMetrics`)
4. `scuttle_filtercells` – filter cells by QC thresholds (`scuttle::filterCells` / direct SCE subsetting)
