## Prompt

I want to write a Galaxy tool wrapper for functionality based on the Bioconductor package 'SingleCellExperiment' (version 1.32.0).

### Constraints
- You are only allowed to edit this file. Lay out all instructions (files to create and their full contents), but leave it to me to apply the changes.
- Use package version from Bioconductor release 3.22.
- Use the Bioconductor package `LoomExperiment` (version 1.28.0) for loading and saving `SingleCellExperiment` object in files with the extension `.loom` using the Galaxy datatype `loom`.
- Embed all R code directly inside the XML in a `<configfile>` block; do not write external R scripts.
- Do not use `--conda_dependency_resolution` in any planemo command.
- `planemo serve` requires a pre-existing Galaxy checkout via `--galaxy_root`; planemo cannot build the Galaxy client from scratch in most environments.
- Do not edit instructions associated with any functionality marked with the tag FREEZE below.

### Functionality

In a file `singlecellexperiment_matchfeatures.xml`, write a tool that takes a `SingleCellExperiment` object (imported from an RDS file), a plain-text file of feature identifiers (one per line, no header), and the name of a column in `rowData` to match against, and produces a single-column `txt` file (without column name) listing the `rownames(sce)` for the features whose `rowData` column value appears in the feature file.

In a separate file `singlecellexperiment_display.xml`, write a tool that takes a `SingleCellExperiment` object (imported from an RDS file), and offers a dropdown menu to print either `colData(sce)` or `rowData(sce)` to a plain-text file.
For testing, use `https://zenodo.org/records/19665318/files/sce_after_addpercellqcmetrics.rds` as input file.

### Test data
Test files will be hosted on Zenodo; do not commit them. Use `value=` attributes in `<param>` elements for now (local files); note that these will be replaced with `location=` Zenodo URLs in due course. After running `planemo test` once, hardcode the observed output sizes into `<has_size>` assertions.

### Planemo commands
Include commands for linting, programmatic testing, and interactive testing (planemo serve).

---

# Galaxy Tool: SingleCellExperiment 1.32.0 — `matchfeatures`

## Approach

Each tool in this suite:
- accepts a `SingleCellExperiment` object serialised as an RDS file (Galaxy format `rdata.sce`)
- embeds all R code in a `<configfile name="script_file">` block using Cheetah template syntax
- exposes `@CMD@`, `@CMD_imports@`, `@CMD_read_inputs@` macro tokens for boilerplate — mirroring `tools/seurat/macros.xml` and `tools/scuttle-1.20.0/macros.xml`
- one `.xml` file per tool function; shared macros live in `macros.xml`
- test data in `test-data/` is gitignored (`.gitignore` already present); will be uploaded to Zenodo

## Directory layout

```
tools/singlecellexperiment-1.32.0/
├── macros.xml
├── singlecellexperiment_matchfeatures.xml
├── test-data/
│   ├── sce_input.rds              # not committed (gitignored)
│   ├── mt_gene_symbols.txt        # committed; gene symbols to match
│   └── .gitignore
├── .shed.yml
└── dev/plan.md                # this file
```

---

## Step 0 – Understand the functionality

The tool takes a pre-existing plain-text file of feature identifiers (one per
line, no header) and looks them up in a user-chosen `rowData` column of a
`SingleCellExperiment`.  It writes the corresponding `rownames`
(feature identifiers — typically Ensembl IDs) to a plain text file, one per
line.  No header.

The R implementation is two lines:

```r
feature_ids <- readLines('feature_list.txt')
matched <- rownames(sce)[rowData(sce)[[rowdata_col]] %in% feature_ids]
writeLines(matched, 'output.txt')
```

Parameters to expose in Galaxy:

| Parameter | Galaxy param type | Notes |
|---|---|---|
| SCE object | `data` (rdata.sce) | input SCE |
| feature file | `data` (txt) | one feature identifier per line, no header |
| `rowdata_col` | `text` | rowData column to match against; default `"Symbol"` |

Output: a `txt` dataset, one feature identifier per line.

---

## Step 1 – Test data

`test-data/sce_input.rds` is the same minimal SCE used by the scuttle tools
(~50 cells, ~200 genes, generated from the 10x dataset in
`dev-scuttle-galaxy/data/`).  It is already present.

`test-data/mt_gene_symbols.txt` is a committed plain-text file containing the
gene symbols of the MT genes present in the test SCE (one per line, no header).
Obtain the symbols by running:

```r
library(SingleCellExperiment)
sce <- readRDS("test-data/sce_input.rds")
mt_symbols <- rowData(sce)[grep("^MT-", rowData(sce)[["Symbol"]]), "Symbol"]
writeLines(mt_symbols, "test-data/mt_gene_symbols.txt")
length(mt_symbols)   # → 5 (use this as n in has_n_lines)
```

Commit `mt_gene_symbols.txt` (it is plain text; only `*.rds` is gitignored).
Do **not** commit the RDS file — it is gitignored and will be hosted on Zenodo.

---

## Step 2 – Write `macros.xml`

```xml
<?xml version="1.0"?>
<macros>
    <token name="@TOOL_VERSION@">1.32.0</token>
    <token name="@VERSION_SUFFIX@">0</token>
    <token name="@PROFILE@">23.0</token>

    <xml name="requirements">
        <requirements>
            <requirement type="package" version="@TOOL_VERSION@">bioconductor-singlecellexperiment</requirement>
            <yield/>
        </requirements>
    </xml>

    <!-- Mirrors @CMD@ in seurat/macros.xml -->
    <token name="@CMD@"><![CDATA[
cp '$input_rds' sce.rds &&
cat '$script_file' > '$hidden_output' &&
Rscript '$script_file' >> '$hidden_output'
    ]]></token>

    <!-- Mirrors @CMD_imports@ in seurat/macros.xml -->
    <token name="@CMD_imports@"><![CDATA[
library(SingleCellExperiment)
    ]]></token>

    <!-- Mirrors @CMD_read_inputs@ in seurat/macros.xml -->
    <token name="@CMD_read_inputs@"><![CDATA[
sce <- readRDS('sce.rds')
    ]]></token>

    <xml name="input_rds">
        <param name="input_rds" type="data" format="rdata.sce"
               label="Input SingleCellExperiment (RDS)"/>
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
            <citation type="doi">10.1038/s41592-019-0654-x</citation>
        </citations>
    </xml>
</macros>
```

---

## Step 3 – Write `singlecellexperiment_matchfeatures.xml`

```xml
<?xml version="1.0"?>
<tool id="singlecellexperiment_matchfeatures" name="SingleCellExperiment: Match features"
      version="@TOOL_VERSION@+galaxy@VERSION_SUFFIX@" profile="@PROFILE@">
    <description>by feature list in a SingleCellExperiment object</description>
    <macros>
        <import>macros.xml</import>
    </macros>
    <expand macro="requirements"/>
    <command detect_errors="exit_code"><![CDATA[
cp '$input_rds' sce.rds &&
cp '$feature_file' feature_list.txt &&
cat '$script_file' > '$hidden_output' &&
Rscript '$script_file' >> '$hidden_output'
    ]]></command>
    <configfiles>
        <configfile name="script_file"><![CDATA[
@CMD_imports@
@CMD_read_inputs@

feature_ids <- readLines('feature_list.txt')
matched <- rownames(sce)[rowData(sce)[['$rowdata_col']] %in% feature_ids]
writeLines(matched, 'output.txt')
        ]]></configfile>
    </configfiles>
    <inputs>
        <expand macro="input_rds"/>
        <param name="feature_file" type="data" format="txt"
               label="Feature list"
               help="Plain text file with one feature identifier per line, no header"/>
        <param name="rowdata_col" type="text" value="Symbol"
               label="rowData column to match against"
               help="The column whose values are compared to the feature list"/>
        <expand macro="inputs_common_advanced"/>
    </inputs>
    <outputs>
        <data name="output" format="txt" from_work_dir="output.txt"
              label="${tool.name} on ${on_string}"/>
        <expand macro="outputs_common_advanced"/>
    </outputs>
    <tests>
        <test expect_num_outputs="2">
            <param name="input_rds" value="sce_input.rds" ftype="rdata.sce"/>
            <param name="feature_file" value="mt_gene_symbols.txt" ftype="txt"/>
            <param name="rowdata_col" value="Symbol"/>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="writeLines"/>
                </assert_contents>
            </output>
            <output name="output" ftype="txt">
                <assert_contents>
                    <has_n_lines n="5"/>
                </assert_contents>
            </output>
        </test>
    </tests>
    <help><![CDATA[
**What it does**

Takes a plain-text file of feature identifiers (one per line, no header) and
looks them up in a chosen ``rowData`` column of a ``SingleCellExperiment``
object.  Returns the corresponding ``rownames`` (typically Ensembl IDs) as a
plain text file, one per line, with no header.

**Inputs**

- A ``SingleCellExperiment`` object saved as an RDS file.
- A plain text file of feature identifiers to match (e.g. gene symbols).
- The name of a ``rowData`` column to match against (default ``Symbol``).

**Outputs**

- A plain text file listing the matching feature identifiers (rownames), one
  per line.  This can be used as input to other tools that accept gene lists.
    ]]></help>
    <expand macro="citations"/>
</tool>
```

---

## Step 4 – Lint and test with Planemo

```bash
# From the tools/singlecellexperiment-1.32.0/ directory:

# Lint XML
planemo lint singlecellexperiment_matchfeatures.xml

# Run tests (builds Conda env on first run – slow)
planemo test singlecellexperiment_matchfeatures.xml

# Serve locally to click through the Galaxy UI
# Requires a pre-existing Galaxy checkout; planemo cannot build the client from scratch.
planemo serve singlecellexperiment_matchfeatures.xml
```

---

## Step 5 – Write `.shed.yml`

```yaml
categories:
  - Transcriptomics
  - Single-cell
description: Galaxy wrappers for the Bioconductor SingleCellExperiment package (v1.32.0)
long_description: |
  SingleCellExperiment provides a container for single-cell genomics data,
  built around the SummarizedExperiment class with extensions for reduced
  dimensions and alternative experiments.
name: singlecellexperiment
owner: iuc
```

---

## Step 6 – Iterate for subsequent tool functions

Repeat Steps 1–4 for each new tool.  Each tool gets its own `.xml` with a
`<configfile>` block; shared tokens and macros grow in `macros.xml`.

---

# Galaxy Tool: SingleCellExperiment 1.32.0 — `display`

## Approach

Follow the same structure as `matchfeatures`: embed R code in a `<configfile>` block
using shared macros from `macros.xml`.

## Step 7 – Understand the functionality

The tool takes a `SingleCellExperiment` object and displays its column data (cell
metadata). The R implementation is one line:

```r
write.table(colData(sce), 'output.txt', sep='\t', row.names=TRUE)
```

Parameters to expose in Galaxy:

| Parameter | Galaxy param type | Notes |
|---|---|---|
| SCE object | `data` (rdata.sce) | input SCE |

Output: a `txt` dataset with tab-separated cell metadata (columns as colData columns,
rows as cells).

---

## Step 8 – Test data

Use `location="https://zenodo.org/records/19665318/files/sce_after_addpercellqcmetrics.rds"`.

Provenance: downloaded from local Galaxy instance after testing `scuttle_addpercellqcmetrics`.

---

## Step 9 – Write `singlecellexperiment_display.xml`

```xml
<?xml version="1.0"?>
<tool id="singlecellexperiment_display" name="SingleCellExperiment: Display cell data"
      version="@TOOL_VERSION@+galaxy@VERSION_SUFFIX@" profile="@PROFILE@">
    <description>display colData from a SingleCellExperiment object</description>
    <macros>
        <import>macros.xml</import>
    </macros>
    <expand macro="requirements"/>
    <command detect_errors="exit_code"><![CDATA[
cp '$input_rds' sce.rds &&
cat '$script_file' > '$hidden_output' &&
Rscript '$script_file' >> '$hidden_output'
    ]]></command>
    <configfiles>
        <configfile name="script_file"><![CDATA[
@CMD_imports@
@CMD_read_inputs@

write.table(colData(sce), 'output.txt', sep='\t', row.names=FALSE, quote = FALSE)
        ]]></configfile>
    </configfiles>
    <inputs>
        <expand macro="input_rds"/>
        <expand macro="inputs_common_advanced"/>
    </inputs>
    <outputs>
        <data name="output" format="txt" from_work_dir="output.txt"
              label="${tool.name} on ${on_string}"/>
        <expand macro="outputs_common_advanced"/>
    </outputs>
    <tests>
        <test expect_num_outputs="2">
            <param name="input_rds" location="https://zenodo.org/records/19665318/files/sce_after_addpercellqcmetrics.rds" ftype="rdata.sce"/>
            <section name="advanced_common">
                <param name="show_log" value="true"/>
            </section>
            <output name="hidden_output">
                <assert_contents>
                    <has_text_matching expression="write.table"/>
                </assert_contents>
            </output>
            <output name="output" ftype="txt">
                <assert_contents>
                    <has_size value="2500" delta="100"/>
                </assert_contents>
            </output>
        </test>
    </tests>
    <help><![CDATA[
**What it does**

Extracts and displays the column data (cell metadata) from a ``SingleCellExperiment``
object as a tab-separated plain text file.

**Inputs**

- A ``SingleCellExperiment`` object saved as an RDS file.

**Outputs**

- A tab-separated plain text file with cell metadata.  Columns correspond to the
  metadata variables in ``colData(sce)``; rows correspond to cells.  The first column
  contains cell barcodes (``colnames(sce)``).
    ]]></help>
    <expand macro="citations"/>
</tool>
```

---

## Step 10 – Lint and test with Planemo

```bash
# From the tools/singlecellexperiment-1.32.0/ directory:

# Lint XML
planemo lint singlecellexperiment_display.xml

# Run tests
planemo test singlecellexperiment_display.xml

# Serve locally for interactive testing
planemo serve singlecellexperiment_display.xml
```

---

## Step 11 – Update `.shed.yml` if needed

The existing `.shed.yml` already covers the tool suite; no updates required
(both tools will be packaged together under the same repository).
